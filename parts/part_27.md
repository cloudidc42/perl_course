# Part 27: Perl Networking
## Steps 261-270: การเขียนโปรแกรมเครือข่าย

---

## Step 261: Socket พื้นฐาน

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Socket qw(:all);

# =====================
# TCP Client (low-level)
# =====================

sub tcp_connect {
    my ($host, $port) = @_;
    
    # Resolve hostname
    my $ip = inet_aton($host) or die "Cannot resolve $host\n";
    my $addr = sockaddr_in($port, $ip);
    
    # Create socket
    socket(my $sock, AF_INET, SOCK_STREAM, getprotobyname('tcp'))
        or die "socket: $!";
    
    # Connect
    connect($sock, $addr) or die "connect to $host:$port: $!";
    
    # Disable buffering
    my $old = select $sock;
    $| = 1;
    select $old;
    
    return $sock;
}

# Simple HTTP/1.0 request using raw socket
sub http_get {
    my ($host, $path) = @_;
    $path //= "/";
    
    eval {
        my $sock = tcp_connect($host, 80);
        
        # Send request
        print $sock "GET $path HTTP/1.0\r\n";
        print $sock "Host: $host\r\n";
        print $sock "Connection: close\r\n";
        print $sock "\r\n";
        
        # Read response
        my $response = "";
        while (my $chunk = <$sock>) {
            $response .= $chunk;
        }
        close $sock;
        
        return $response;
    };
    
    return undef;
}

printf "Socket module: %s\n", Socket->VERSION // "built-in";
printf "AF_INET: %d\n", AF_INET;
printf "SOCK_STREAM: %d\n", SOCK_STREAM;

# Demonstrate inet functions
my @hosts = ("127.0.0.1", "10.0.0.1", "192.168.1.100");
for my $h (@hosts) {
    my $n = inet_aton($h);
    printf "%-18s -> %s\n", $h, inet_ntoa($n);
}

# Port number lookup
for my $svc (qw(http https ftp ssh smtp)) {
    my $port = getservbyname($svc, "tcp");
    printf "%-8s port: %d\n", $svc, $port // 0;
}

# Hostname
my $hostname = `hostname`;
chomp $hostname;
printf "\nHostname: %s\n", $hostname;

if (my $ip = inet_aton($hostname)) {
    printf "IP: %s\n", inet_ntoa($ip);
}
```

---

## Step 262: IO::Socket::INET

```perl
#!/usr/bin/perl
use strict;
use warnings;
use IO::Socket::INET;
use IO::Select;

# =====================
# Simple TCP Server (background)
# =====================

sub start_echo_server {
    my $port = shift // 0;  # 0 = OS assigns port
    
    my $server = IO::Socket::INET->new(
        LocalPort => $port,
        Type      => SOCK_STREAM,
        Reuse     => 1,
        Listen    => 5,
    ) or die "Cannot create server: $!\n";
    
    printf "Echo server on port %d\n", $server->sockport;
    return $server;
}

# =====================
# TCP Client
# =====================

sub connect_to {
    my ($host, $port) = @_;
    return IO::Socket::INET->new(
        PeerAddr => $host,
        PeerPort => $port,
        Proto    => 'tcp',
        Timeout  => 5,
    ) or die "Cannot connect: $!\n";
}

# =====================
# Demo with fork
# =====================

my $server = start_echo_server(0);
my $port   = $server->sockport;

my $pid = fork();
defined $pid or die "fork: $!";

if ($pid == 0) {
    # Child: simple echo server for 3 connections
    for (1..3) {
        my $client = $server->accept or last;
        while (my $line = <$client>) {
            print $client $line;  # echo back
            last if $line =~ /quit/;
        }
        close $client;
    }
    $server->close;
    exit 0;
}

# Parent: client
sleep 1;  # Wait for server to start

my @messages = ("Hello\n", "World\n", "From Perl\n");
for my $msg (@messages) {
    my $sock = connect_to("127.0.0.1", $port);
    print $sock $msg;
    my $reply = <$sock>;
    chomp(my $sent = $msg);
    chomp(my $got  = $reply // "");
    printf "Sent: '%-15s' Got: '%s'\n", $sent, $got;
    close $sock;
}

# Signal server to stop
{
    my $sock = connect_to("127.0.0.1", $port);
    print $sock "quit\n";
    close $sock;
}

waitpid($pid, 0);
$server->close;

# =====================
# UDP Socket
# =====================

{
    my $recv = IO::Socket::INET->new(
        LocalAddr => "127.0.0.1",
        LocalPort => 0,
        Proto     => "udp",
    ) or die "UDP recv: $!\n";
    
    my $recv_port = $recv->sockport;
    
    my $send = IO::Socket::INET->new(
        PeerAddr  => "127.0.0.1",
        PeerPort  => $recv_port,
        Proto     => "udp",
    ) or die "UDP send: $!\n";
    
    $send->send("UDP Hello");
    
    my $buf;
    $recv->recv($buf, 256);
    printf "\nUDP received: '%s'\n", $buf;
    
    $recv->close;
    $send->close;
}
```

---

## Step 263: LWP::UserAgent Advanced

```perl
#!/usr/bin/perl
use strict;
use warnings;
use LWP::UserAgent;
use HTTP::Request;
use HTTP::Response;
use URI;
use URI::QueryParam;

# =====================
# LWP::UserAgent setup
# =====================

my $ua = LWP::UserAgent->new(
    agent      => "PerlBot/1.0 (compatible)",
    timeout    => 30,
    max_size   => 1_000_000,  # 1MB max
    keep_alive => 1,
);

# Custom headers
$ua->default_header('Accept'          => 'application/json, text/html');
$ua->default_header('Accept-Encoding' => 'gzip, deflate');
$ua->default_header('Accept-Language' => 'en-US,en;q=0.9,th;q=0.8');

# =====================
# Request/Response handling
# =====================

sub make_request {
    my ($method, $url, %opts) = @_;
    
    my $req = HTTP::Request->new($method => $url);
    
    # Add custom headers
    if (my $headers = $opts{headers}) {
        $req->header($_, $headers->{$_}) for keys %$headers;
    }
    
    # Add body
    if (my $body = $opts{body}) {
        $req->content_type($opts{content_type} // 'application/json');
        $req->content($body);
    }
    
    return $ua->request($req);
}

# =====================
# HTTP response helpers
# =====================

sub parse_response {
    my $res = shift;
    return {
        ok          => $res->is_success,
        status      => $res->code,
        status_line => $res->status_line,
        content     => $res->decoded_content,
        content_type => $res->content_type,
        headers     => { map { $_ => $res->header($_) } $res->header_field_names },
        size        => length($res->content),
    };
}

# =====================
# Retry logic
# =====================

sub get_with_retry {
    my ($url, %opts) = @_;
    my $max_retries = $opts{retries} // 3;
    my $delay       = $opts{delay}   // 1;
    
    for my $attempt (1..$max_retries) {
        my $res = $ua->get($url);
        return $res if $res->is_success;
        
        if ($attempt < $max_retries) {
            my $wait = $delay * (2 ** ($attempt - 1));
            printf "Attempt %d failed (%s), retrying in %.1fs\n",
                $attempt, $res->status_line, $wait;
            select undef, undef, undef, $wait;
        }
    }
    
    return $ua->get($url);  # final attempt
}

# =====================
# URI building
# =====================

sub build_url {
    my ($base, %params) = @_;
    my $uri = URI->new($base);
    $uri->query_form(%params);
    return $uri->as_string;
}

my $url = build_url("https://api.example.com/search",
    q => "perl moose",
    page => 1,
    per_page => 10,
);
printf "Built URL: %s\n", $url;

# =====================
# Simulated API demo (httpbin-style)
# =====================

# Since we can't hit real internet, demonstrate the API structure
my $mock_response = HTTP::Response->new(200, "OK");
$mock_response->content_type("application/json");
$mock_response->content('{"status":"ok","data":{"user":"alice","role":"admin"}}');

my $parsed = parse_response($mock_response);
printf "\nMock response:\n";
printf "  Status:   %d %s\n", $parsed->{status}, "(OK)";
printf "  OK:       %s\n",    $parsed->{ok} ? "yes" : "no";
printf "  Type:     %s\n",    $parsed->{content_type};
printf "  Size:     %d bytes\n", $parsed->{size};
printf "  Body:     %s\n",    $parsed->{content};

# =====================
# Cookie handling
# =====================

use HTTP::CookieJar::LWP;

my $jar = HTTP::CookieJar::LWP->new;
$ua->cookie_jar($jar);

printf "\nCookie jar configured\n";
printf "UA agent: %s\n", $ua->agent;
```

---

## Step 264: HTTP::Tiny and JSON API

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

# =====================
# HTTP::Tiny — lightweight HTTP client
# =====================

{
package SimpleHTTP;

sub new {
    my ($class, %opts) = @_;
    return bless {
        agent     => $opts{agent}   // "SimpleHTTP/1.0",
        timeout   => $opts{timeout} // 30,
        base_url  => $opts{base_url} // "",
        headers   => $opts{headers} // {},
        _json     => JSON::PP->new->utf8,
    }, $class;
}

sub _build_headers {
    my ($self, $extra) = @_;
    return { %{$self->{headers}}, %{$extra//{}}, 'User-Agent' => $self->{agent} };
}

sub get {
    my ($self, $path, %opts) = @_;
    return $self->_request("GET", $path, %opts);
}

sub post {
    my ($self, $path, $data, %opts) = @_;
    my $body = ref($data) ? $self->{_json}->encode($data) : $data;
    return $self->_request("POST", $path, body => $body, %opts);
}

sub _request {
    my ($self, $method, $path, %opts) = @_;
    
    # Try HTTP::Tiny if available
    eval { require HTTP::Tiny };
    unless ($@) {
        my $http = HTTP::Tiny->new(
            agent   => $self->{agent},
            timeout => $self->{timeout},
        );
        
        my $url = $self->{base_url} . $path;
        my %req_opts;
        $req_opts{content} = $opts{body} if $opts{body};
        $req_opts{headers} = $self->_build_headers($opts{headers});
        $req_opts{headers}{'Content-Type'} = 'application/json' if $opts{body};
        
        my $res = $http->request($method, $url, \%req_opts);
        return $self->_wrap_response($res);
    }
    
    # Fallback: return mock
    return {
        ok      => 1,
        status  => 200,
        content => '{"mock":true}',
        headers => {},
    };
}

sub _wrap_response {
    my ($self, $res) = @_;
    my $content = $res->{content} // "";
    my $data;
    eval { $data = $self->{_json}->decode($content) };
    return {
        ok      => $res->{success},
        status  => $res->{status},
        content => $content,
        data    => $data,
        headers => $res->{headers} // {},
    };
}
}

# =====================
# REST API Client
# =====================

{
package RESTClient;

sub new {
    my ($class, %opts) = @_;
    return bless {
        http     => SimpleHTTP->new(%opts),
        base_url => $opts{base_url} // "",
        token    => $opts{token},
    }, $class;
}

sub _headers {
    my $self = shift;
    my %h = ('Accept' => 'application/json');
    $h{'Authorization'} = "Bearer $self->{token}" if $self->{token};
    return \%h;
}

sub get    { my ($self, $path) = @_; $self->{http}->get($path, headers => $self->_headers) }
sub post   { my ($self, $path, $data) = @_; $self->{http}->post($path, $data, headers => $self->_headers) }
sub put    { my ($self, $path, $data) = @_; $self->{http}->_request("PUT",    $path, body => JSON::PP->new->utf8->encode($data), headers => $self->_headers) }
sub delete { my ($self, $path) = @_;        $self->{http}->_request("DELETE", $path, headers => $self->_headers) }
}

# =====================
# Demo with mock responses
# =====================

package main;

my $client = RESTClient->new(
    base_url => "https://jsonplaceholder.typicode.com",
    agent    => "PerlRESTClient/1.0",
    token    => "my-api-token",
);

# Demonstrate the structure
printf "REST client created\n";
printf "Base URL: %s\n", "https://jsonplaceholder.typicode.com";

# Mock a successful API response
sub mock_api_call {
    my ($method, $endpoint, $data) = @_;
    my %responses = (
        "GET /users/1"  => { id => 1, name => "Alice", email => "alice\@example.com" },
        "GET /posts"    => [{ id => 1, title => "Hello Perl" }, { id => 2, title => "CGI Programming" }],
        "POST /posts"   => { id => 101, %{$data//{}}, created => 1 },
    );
    my $key = "$method $endpoint";
    return $responses{$key} // { error => "not found" };
}

# User
my $user = mock_api_call("GET", "/users/1");
printf "\nUser: %s (%s)\n", $user->{name}, $user->{email};

# Posts
my $posts = mock_api_call("GET", "/posts");
printf "Posts (%d):\n", scalar @$posts;
printf "  [%d] %s\n", $_->{id}, $_->{title} for @$posts;

# Create post
my $new_post = mock_api_call("POST", "/posts", { title => "New Article", body => "Content here", userId => 1 });
printf "Created post ID: %d\n", $new_post->{id};
```

---

## Step 265: Web Scraping

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# HTML parser without modules
# =====================

sub parse_html_simple {
    my $html = shift;
    my %data;
    
    # Extract title
    if ($html =~ /<title[^>]*>(.*?)<\/title>/si) {
        ($data{title} = $1) =~ s/<[^>]+>//g;
        $data{title} =~ s/\s+/ /g;
        $data{title} =~ s/^\s+|\s+$//g;
    }
    
    # Extract all links
    my @links;
    while ($html =~ /<a\s+[^>]*href\s*=\s*["']([^"']+)["'][^>]*>(.*?)<\/a>/gsi) {
        my ($href, $text) = ($1, $2);
        $text =~ s/<[^>]+>//g;
        $text =~ s/\s+/ /g;
        $text =~ s/^\s+|\s+$//g;
        push @links, { href => $href, text => $text };
    }
    $data{links} = \@links;
    
    # Extract meta tags
    my %meta;
    while ($html =~ /<meta\s+([^>]+)>/gsi) {
        my $attrs = $1;
        my ($name)    = $attrs =~ /name\s*=\s*["']([^"']+)["']/i;
        my ($prop)    = $attrs =~ /property\s*=\s*["']([^"']+)["']/i;
        my ($content) = $attrs =~ /content\s*=\s*["']([^"']+)["']/i;
        if (($name // $prop) && $content) {
            $meta{$name // $prop} = $content;
        }
    }
    $data{meta} = \%meta;
    
    # Extract headings
    my @headings;
    while ($html =~ /<h([1-6])[^>]*>(.*?)<\/h\1>/gsi) {
        my $text = $2;
        $text =~ s/<[^>]+>//g;
        $text =~ s/\s+/ /g;
        push @headings, { level => $1, text => $text };
    }
    $data{headings} = \@headings;
    
    # Extract text (strip all tags)
    (my $plain_text = $html) =~ s/<[^>]+>//g;
    $plain_text =~ s/\s+/ /g;
    $plain_text =~ s/^\s+|\s+$//g;
    $data{text} = $plain_text;
    
    return %data;
}

# Sample HTML
my $html = q{
<!DOCTYPE html>
<html>
<head>
    <title>  Perl Web Scraping Demo  </title>
    <meta name="description" content="A demo page for Perl scraping">
    <meta name="keywords" content="perl,scraping,web">
    <meta property="og:title" content="Perl Demo">
</head>
<body>
    <h1>Welcome to Perl</h1>
    <p>Perl is a <strong>powerful</strong> language.</p>
    <h2>Features</h2>
    <ul>
        <li><a href="/docs">Documentation</a></li>
        <li><a href="https://cpan.org">CPAN</a></li>
        <li><a href="/tutorial" class="active">Tutorial</a></li>
    </ul>
    <h2>Links</h2>
    <a href="https://perl.org">Official Perl</a>
    <a href="https://metacpan.org">MetaCPAN</a>
</body>
</html>
};

my %parsed = parse_html_simple($html);

printf "Title: %s\n", $parsed{title};
printf "\nMeta:\n";
printf "  %-15s %s\n", $_, $parsed{meta}{$_} for sort keys %{$parsed{meta}};

printf "\nHeadings:\n";
printf "  H%d: %s\n", $_->{level}, $_->{text} for @{$parsed{headings}};

printf "\nLinks (%d):\n", scalar @{$parsed{links}};
printf "  %-30s %s\n", $_->{href}, $_->{text} for @{$parsed{links}};

# =====================
# Data extraction
# =====================

sub extract_table {
    my $html = shift;
    my @rows;
    
    while ($html =~ /<tr[^>]*>(.*?)<\/tr>/gsi) {
        my $row_html = $1;
        my @cells;
        while ($row_html =~ /<t[hd][^>]*>(.*?)<\/t[hd]>/gsi) {
            my $cell = $1;
            $cell =~ s/<[^>]+>//g;
            $cell =~ s/\s+/ /g;
            $cell =~ s/^\s+|\s+$//g;
            push @cells, $cell;
        }
        push @rows, \@cells if @cells;
    }
    
    return @rows;
}

my $table_html = q{
<table>
  <tr><th>Name</th><th>Price</th><th>Stock</th></tr>
  <tr><td>Widget A</td><td>$9.99</td><td>100</td></tr>
  <tr><td><strong>Widget B</strong></td><td>$14.99</td><td>50</td></tr>
</table>
};

my @table_rows = extract_table($table_html);
printf "\n%-15s %-10s %s\n", @{$table_rows[0]};
printf "%-15s %-10s %s\n",   @{$_} for @table_rows[1..$#table_rows];
```

---

## Step 266: Email with MIME

```perl
#!/usr/bin/perl
use strict;
use warnings;
use MIME::Base64 qw(encode_base64 decode_base64);
use Encode qw(encode decode);

# =====================
# MIME message builder
# =====================

{
package MIMEBuilder;

use MIME::Base64 qw(encode_base64);

sub new {
    return bless {
        from     => '',
        to       => [],
        cc       => [],
        subject  => '',
        parts    => [],
        headers  => {},
    }, shift;
}

sub from    { $_[0]->{from} = $_[1]; $_[0] }
sub to      { push @{$_[0]->{to}}, $_[1];   $_[0] }
sub cc      { push @{$_[0]->{cc}}, $_[1];   $_[0] }
sub subject { $_[0]->{subject} = $_[1];      $_[0] }
sub header  { $_[0]->{headers}{$_[1]} = $_[2]; $_[0] }

sub text_part {
    my ($self, $text, %opts) = @_;
    push @{$self->{parts}}, {
        type     => 'text/plain',
        charset  => $opts{charset} // 'UTF-8',
        encoding => 'base64',
        content  => $text,
    };
    return $self;
}

sub html_part {
    my ($self, $html, %opts) = @_;
    push @{$self->{parts}}, {
        type     => 'text/html',
        charset  => $opts{charset} // 'UTF-8',
        encoding => 'base64',
        content  => $html,
    };
    return $self;
}

sub attachment {
    my ($self, $filename, $content, $mime_type) = @_;
    push @{$self->{parts}}, {
        type        => $mime_type // 'application/octet-stream',
        filename    => $filename,
        encoding    => 'base64',
        disposition => 'attachment',
        content     => $content,
    };
    return $self;
}

sub build {
    my $self = shift;
    my $boundary = "=_perl_" . time() . "_$$";
    my @lines;
    
    # Headers
    push @lines, "From: $self->{from}";
    push @lines, "To: " . join(", ", @{$self->{to}});
    push @lines, "Cc: " . join(", ", @{$self->{cc}}) if @{$self->{cc}};
    push @lines, "Subject: $self->{subject}";
    push @lines, "MIME-Version: 1.0";
    push @lines, "Content-Type: multipart/mixed; boundary=\"$boundary\"";
    push @lines, "$_: $self->{headers}{$_}" for keys %{$self->{headers}};
    push @lines, "";
    
    # Parts
    for my $part (@{$self->{parts}}) {
        push @lines, "--$boundary";
        
        my $ct = $part->{type};
        $ct .= "; charset=$part->{charset}" if $part->{charset};
        $ct .= "; name=\"$part->{filename}\"" if $part->{filename};
        push @lines, "Content-Type: $ct";
        push @lines, "Content-Transfer-Encoding: $part->{encoding}";
        
        if ($part->{disposition}) {
            push @lines, "Content-Disposition: $part->{disposition}; filename=\"$part->{filename}\"";
        }
        
        push @lines, "";
        push @lines, encode_base64($part->{content}, "\n");
    }
    
    push @lines, "--$boundary--";
    return join("\r\n", @lines);
}
}

package main;

# Build a MIME email
my $email = MIMEBuilder->new
    ->from("sender\@example.com")
    ->to("alice\@example.com")
    ->to("bob\@example.com")
    ->cc("manager\@example.com")
    ->subject("Test Email from Perl")
    ->header("X-Mailer", "PerlMailer/1.0")
    ->text_part("Hello!\n\nThis is a test email built with Perl.\n\nRegards")
    ->html_part("<h1>Hello!</h1><p>This is a <b>test email</b> built with <em>Perl</em>.</p>")
    ->attachment("data.csv", "name,age\nalice,28\nbob,35\n", "text/csv")
    ->build;

printf "Email built: %d bytes\n", length($email);
printf "First line: %s\n", (split /\r\n/, $email)[0];

# Show structure
my @mime_parts = ($email =~ /^Content-Type: (.+)$/mg);
printf "\nMIME parts:\n";
printf "  %s\n", $_ for @mime_parts;

# =====================
# Base64 utilities
# =====================

printf "\n--- Base64 ---\n";
my $data = "Hello, World! This is Perl.";
my $encoded = encode_base64($data);
my $decoded = decode_base64($encoded);
printf "Original:  %s\n", $data;
printf "Encoded:   %s", $encoded;
printf "Decoded:   %s\n", $decoded;
printf "Match: %s\n", $data eq $decoded ? "yes" : "no";
```

---

## Step 267: Simple HTTP Server

```perl
#!/usr/bin/perl
use strict;
use warnings;
use IO::Socket::INET;
use POSIX ":sys_wait_h";

# =====================
# Minimal HTTP/1.0 server
# =====================

{
package MiniHTTPServer;

sub new {
    my ($class, %opts) = @_;
    return bless {
        port     => $opts{port}    // 8080,
        host     => $opts{host}    // "127.0.0.1",
        routes   => {},
        not_found => sub { ("404 Not Found", "text/plain", "404 Not Found\n") },
    }, $class;
}

sub route {
    my ($self, $method, $path, $handler) = @_;
    $self->{routes}{"$method $path"} = $handler;
    return $self;
}

sub get  { $_[0]->route("GET",  $_[1], $_[2]) }
sub post { $_[0]->route("POST", $_[1], $_[2]) }

sub _parse_request {
    my ($self, $client) = @_;
    my $request_line = <$client>;
    return unless $request_line;
    
    chomp $request_line;
    $request_line =~ s/\r$//;
    my ($method, $path, $proto) = split /\s+/, $request_line;
    
    # Parse headers
    my %headers;
    while (my $line = <$client>) {
        $line =~ s/[\r\n]//g;
        last unless length $line;
        my ($name, $value) = split /:\s*/, $line, 2;
        $headers{lc $name} = $value;
    }
    
    # Parse query string
    my %params;
    if ($path =~ s/\?(.*)$//) {
        for my $pair (split /&/, $1) {
            my ($k, $v) = split /=/, $pair, 2;
            $params{_decode_uri($k)} = _decode_uri($v//"");
        }
    }
    
    # Read body
    my $body = "";
    if (my $len = $headers{'content-length'}) {
        read $client, $body, $len;
    }
    
    return { method => $method, path => $path, headers => \%headers, params => \%params, body => $body };
}

sub _decode_uri {
    my $s = shift // "";
    $s =~ s/\+/ /g;
    $s =~ s/%([0-9A-Fa-f]{2})/chr(hex($1))/ge;
    return $s;
}

sub _send_response {
    my ($client, $status, $type, $body) = @_;
    $type //= "text/plain";
    my $len = length($body);
    
    print $client "HTTP/1.0 $status\r\n";
    print $client "Content-Type: $type\r\n";
    print $client "Content-Length: $len\r\n";
    print $client "Connection: close\r\n";
    print $client "\r\n";
    print $client $body;
}

sub handle_once {
    my $self   = shift;
    my $server = IO::Socket::INET->new(
        LocalAddr => $self->{host},
        LocalPort => $self->{port},
        Type      => SOCK_STREAM,
        Reuse     => 1,
        Listen    => 5,
    ) or die "Cannot start server: $!\n";
    
    $self->{port} = $server->sockport;
    printf "Server on %s:%d\n", $self->{host}, $self->{port};
    
    my $client = $server->accept;
    if ($client) {
        my $req = $self->_parse_request($client);
        if ($req) {
            my $key     = "$req->{method} $req->{path}";
            my $handler = $self->{routes}{$key} // $self->{not_found};
            my ($status, $type, $body) = $handler->($req);
            _send_response($client, $status, $type, $body);
        }
        close $client;
    }
    
    $server->close;
}
}

package main;

# Setup routes
my $server = MiniHTTPServer->new(port => 0);

$server->get("/", sub {
    return ("200 OK", "text/html",
        "<html><body><h1>Hello from Perl!</h1></body></html>");
});

$server->get("/api/hello", sub {
    my $req = shift;
    my $name = $req->{params}{name} // "World";
    return ("200 OK", "application/json", "{\"message\":\"Hello, $name!\"}");
});

$server->get("/api/time", sub {
    return ("200 OK", "application/json",
        sprintf('{"time":%d,"formatted":"%s"}', time(), scalar localtime));
});

# Run one iteration in a fork
my $pid = fork() // die "fork: $!";

if ($pid == 0) {
    $server->handle_once;
    exit 0;
}

sleep 1;

# Get the port
# Make a request
my $port = 8080;  # demo port - in real usage get from server

printf "\nHTTP server demo complete.\n";
waitpid($pid, 0);
```

---

## Step 268: Net::SMTP

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# SMTP client (simulation since we can't connect)
# =====================

{
package SMTPClient;

sub new {
    my ($class, %opts) = @_;
    return bless {
        host     => $opts{host}     // "localhost",
        port     => $opts{port}     // 25,
        from     => $opts{from}     // "",
        auth     => $opts{auth},
        log      => [],
    }, $class;
}

# SMTP protocol simulation
sub send_mail {
    my ($self, %args) = @_;
    
    my @to = ref $args{to} ? @{$args{to}} : ($args{to});
    
    # Build MIME message
    my @lines;
    push @lines, "From: $args{from}";
    push @lines, "To: " . join(", ", @to);
    push @lines, "Subject: $args{subject}";
    push @lines, "Date: " . scalar(localtime);
    push @lines, "MIME-Version: 1.0";
    push @lines, "Content-Type: text/plain; charset=UTF-8";
    push @lines, "";
    push @lines, $args{body};
    
    my $message = join("\r\n", @lines);
    
    push @{$self->{log}}, {
        from    => $args{from},
        to      => \@to,
        subject => $args{subject},
        bytes   => length($message),
        time    => time(),
        status  => "queued",
    };
    
    printf "[SMTP] Would send: From=%s To=%s Subject='%s'\n",
        $args{from}, join(",", @to), $args{subject};
    
    return 1;
}

sub log { @{$_[0]->{log}} }
}

package main;

my $smtp = SMTPClient->new(
    host => "smtp.example.com",
    port => 587,
    from => "noreply\@example.com",
);

# Send various types
$smtp->send_mail(
    from    => "noreply\@example.com",
    to      => "alice\@example.com",
    subject => "Welcome!",
    body    => "Welcome to our service, Alice!\n\nBest regards,\nThe Team",
);

$smtp->send_mail(
    from    => "noreply\@example.com",
    to      => ["alice\@example.com", "bob\@example.com"],
    subject => "Newsletter #42",
    body    => "Monthly newsletter content here...",
);

# Show log
my @sent = $smtp->log;
printf "\nSMTP Log (%d messages):\n", scalar @sent;
for my $entry (@sent) {
    printf "  Subject: %-30s To: %s\n",
        $entry->{subject}, join(", ", @{$entry->{to}});
}

# =====================
# Email template system
# =====================

{
package EmailTemplate;

my %templates = (
    welcome => {
        subject => "Welcome, {{name}}!",
        body    => "Dear {{name}},\n\nWelcome to {{app}}!\nYour account has been created.\n\nRegards",
    },
    password_reset => {
        subject => "Password Reset Request",
        body    => "Hello {{name}},\n\nClick: {{reset_url}}\n\nExpires in {{expire_hours}} hours.\n\nRegards",
    },
    order_confirm => {
        subject => "Order Confirmed #{{order_id}}",
        body    => "Dear {{name}},\n\nYour order #{{order_id}} for \${{total}} is confirmed.\n\nThank you!",
    },
);

sub render {
    my ($tmpl_name, %vars) = @_;
    my $tmpl = $templates{$tmpl_name} or die "Unknown template: $tmpl_name\n";
    
    my %rendered;
    for my $key (keys %$tmpl) {
        ($rendered{$key} = $tmpl->{$key}) =~ s/\{\{(\w+)\}\}/$vars{$1}\/\/""/ge;
    }
    return %rendered;
}
}

printf "\n--- Email Templates ---\n";
my %welcome = EmailTemplate::render("welcome", name => "Alice", app => "MyApp");
printf "Subject: %s\n", $welcome{subject};
printf "Body:\n%s\n", $welcome{body};

my %order = EmailTemplate::render("order_confirm",
    name => "Bob", order_id => "O-12345", total => "89.99");
printf "Subject: %s\n", $order{subject};
```

---

## Step 269: Protocol Utilities

```perl
#!/usr/bin/perl
use strict;
use warnings;
use MIME::Base64 qw(encode_base64 decode_base64);
use Digest::SHA qw(sha1_hex sha256_hex hmac_sha256_hex);

# =====================
# HTTP Basic Auth
# =====================

sub basic_auth_encode {
    my ($user, $pass) = @_;
    return "Basic " . encode_base64("$user:$pass", "");
}

sub basic_auth_decode {
    my $header = shift;
    $header =~ s/^Basic //;
    my ($user, $pass) = split /:/, decode_base64($header), 2;
    return ($user, $pass);
}

my $auth = basic_auth_encode("alice", "s3cr3t");
printf "Auth header: %s\n", $auth;

my ($user, $pass) = basic_auth_decode($auth);
printf "Decoded: user=%s pass=%s\n", $user, $pass;

# =====================
# JWT-like token (simplified)
# =====================

sub b64url_encode {
    my $s = encode_base64(shift, "");
    $s =~ tr|+/=|-_|d;
    return $s;
}

sub b64url_decode {
    my $s = shift;
    $s =~ tr|-_|+/|;
    $s .= "=" x ((4 - length($s) % 4) % 4);
    return decode_base64($s);
}

sub create_token {
    my ($payload, $secret) = @_;
    require JSON::PP;
    my $json = JSON::PP->new->utf8->canonical;
    
    my $header  = b64url_encode($json->encode({ alg => "HS256", typ => "JWT" }));
    my $body    = b64url_encode($json->encode($payload));
    my $sig_input = "$header.$body";
    my $sig = b64url_encode(pack("H*", hmac_sha256_hex($sig_input, $secret)));
    
    return "$header.$body.$sig";
}

sub verify_token {
    my ($token, $secret) = @_;
    require JSON::PP;
    my $json = JSON::PP->new->utf8;
    
    my ($header, $body, $sig) = split /\./, $token;
    return undef unless $header && $body && $sig;
    
    my $sig_input    = "$header.$body";
    my $expected_sig = b64url_encode(pack("H*", hmac_sha256_hex($sig_input, $secret)));
    
    return undef unless $sig eq $expected_sig;
    
    # Check expiry
    my $payload = $json->decode(b64url_decode($body));
    return undef if $payload->{exp} && $payload->{exp} < time();
    
    return $payload;
}

my $secret = "my_jwt_secret_key";
my $token = create_token({
    sub  => "user123",
    role => "admin",
    iat  => time(),
    exp  => time() + 3600,
}, $secret);

printf "\nJWT token: %s...\n", substr($token, 0, 50);

my $verified = verify_token($token, $secret);
printf "Valid: %s\n", $verified ? "yes" : "no";
printf "Subject: %s\n", $verified->{sub} if $verified;
printf "Role: %s\n", $verified->{role} if $verified;

# Invalid secret
my $bad = verify_token($token, "wrong_secret");
printf "Bad secret: %s\n", $bad ? "valid (ERROR)" : "rejected (correct)";

# =====================
# URL encoding
# =====================

sub uri_encode {
    my $s = shift;
    $s =~ s/([^A-Za-z0-9\-_.~])/sprintf("%%%02X", ord($1))/ge;
    return $s;
}

sub uri_decode {
    my $s = shift;
    $s =~ s/%([0-9A-Fa-f]{2})/chr(hex($1))/ge;
    return $s;
}

printf "\n--- URI Encoding ---\n";
my $original = "Hello World! 你好 & more=stuff";
my $encoded  = uri_encode($original);
my $decoded  = uri_decode($encoded);
printf "Original: %s\n", $original;
printf "Encoded:  %s\n", $encoded;
printf "Decoded:  %s\n", $decoded;
```

---

## Step 270: โปรแกรมสรุป — Mini Web Framework

```perl
#!/usr/bin/perl
# mini_web.pl — Simple web framework
use strict;
use warnings;
use JSON::PP;
use Digest::SHA qw(sha256_hex);
use MIME::Base64 qw(encode_base64 decode_base64);

# =====================
# Request/Response objects
# =====================

{
package Request;
sub new {
    my ($class, %env) = @_;
    my %params;
    if (my $qs = $env{QUERY_STRING}) {
        for my $pair (split /&/, $qs) {
            my ($k, $v) = map { _decode($_) } split /=/, $pair, 2;
            $params{$k} = $v;
        }
    }
    return bless { env => \%env, params => \%params }, $class;
}
sub method  { $_[0]->{env}{REQUEST_METHOD} // "GET" }
sub path    { $_[0]->{env}{PATH_INFO}      // "/" }
sub param   { $_[0]->{params}{$_[1]} }
sub params  { %{$_[0]->{params}} }
sub _decode { my $s=shift//""; $s=~s/\+/ /g; $s=~s/%([0-9A-Fa-f]{2})/chr(hex $1)/ge; $s }
}

{
package Response;
sub new {
    my ($class, $status, $body, $type) = @_;
    return bless { status=>$status//200, body=>$body//"", type=>$type//"text/html", headers=>{} }, $class;
}
sub header    { $_[0]->{headers}{$_[1]} = $_[2]; $_[0] }
sub status    { $_[0]->{status} }
sub body      { $_[0]->{body} }
sub content_type { $_[0]->{type} }
sub render    {
    my $self = shift;
    my $out  = "Status: $self->{status}\r\nContent-Type: $self->{type}\r\n";
    $out .= "$_: $self->{headers}{$_}\r\n" for keys %{$self->{headers}};
    $out .= "\r\n$self->{body}";
    return $out;
}
}

# =====================
# Router
# =====================

{
package Router;
sub new { bless { routes => [] }, shift }
sub add {
    my ($self, $method, $pattern, $handler) = @_;
    push @{$self->{routes}}, { method=>uc $method, pattern=>$pattern, handler=>$handler };
}
sub get    { $_[0]->add("GET",    $_[1], $_[2]) }
sub post   { $_[0]->add("POST",   $_[1], $_[2]) }
sub put    { $_[0]->add("PUT",    $_[1], $_[2]) }
sub delete { $_[0]->add("DELETE", $_[1], $_[2]) }

sub dispatch {
    my ($self, $req) = @_;
    for my $r (@{$self->{routes}}) {
        next unless $r->{method} eq $req->method || $r->{method} eq "ANY";
        my $pat = $r->{pattern};
        my %captures;
        
        # Named params :id
        $pat =~ s/:(\w+)/(?<$1>[^\/]+)/g;
        
        if ($req->path =~ /^${pat}$/) {
            %captures = %+;
            return ($r->{handler}, \%captures);
        }
    }
    return (undef, {});
}
}

# =====================
# Session
# =====================

{
package Session;
my %_store;
sub new      { bless { id => sha256_hex(rand()."$$".time()), data => {} }, shift }
sub id       { $_[0]->{id} }
sub set      { $_[0]->{data}{$_[1]} = $_[2]; $_[0] }
sub get      { $_[0]->{data}{$_[1]} }
sub delete   { delete $_[0]->{data}{$_[1]}; $_[0] }
sub save     { $_store{$_[0]->{id}} = $_[0]->{data} }
sub load     {
    my ($class, $id) = @_;
    return undef unless $id && $_store{$id};
    my $s = bless { id => $id, data => $_store{$id} }, $class;
    return $s;
}
}

# =====================
# App helpers
# =====================

sub json_ok  {
    my $data = shift;
    return Response->new(200, JSON::PP->new->utf8->encode($data), "application/json");
}

sub json_err {
    my ($code, $msg) = @_;
    return Response->new($code, JSON::PP->new->utf8->encode({error=>$msg}), "application/json");
}

sub html_ok {
    my $html = shift;
    return Response->new(200, $html, "text/html; charset=utf-8");
}

# =====================
# Mini Application
# =====================

my $router = Router->new;

# Routes
$router->get("/", sub {
    my ($req, $params) = @_;
    return html_ok("<h1>Welcome</h1><p>Mini Perl Web Framework</p>");
});

$router->get("/api/users", sub {
    my ($req, $params) = @_;
    my @users = (
        { id=>1, name=>"Alice", role=>"admin" },
        { id=>2, name=>"Bob",   role=>"user"  },
    );
    return json_ok(\@users);
});

$router->get("/api/users/:id", sub {
    my ($req, $params) = @_;
    my %users = (1=>{id=>1,name=>"Alice"}, 2=>{id=>2,name=>"Bob"});
    return $users{$params->{id}}
        ? json_ok($users{$params->{id}})
        : json_err(404, "User not found");
});

$router->get("/greet", sub {
    my ($req, $params) = @_;
    my $name = $req->param("name") // "World";
    return html_ok("<h1>Hello, $name!</h1>");
});

# =====================
# Dispatch requests
# =====================

printf "=== Mini Web Framework Demo ===\n\n";

my @requests = (
    { method => "GET", path => "/", query => "" },
    { method => "GET", path => "/api/users", query => "" },
    { method => "GET", path => "/api/users/1", query => "" },
    { method => "GET", path => "/api/users/99", query => "" },
    { method => "GET", path => "/greet", query => "name=Alice" },
    { method => "GET", path => "/not-found", query => "" },
);

for my $r (@requests) {
    my $req = Request->new(
        REQUEST_METHOD => $r->{method},
        PATH_INFO      => $r->{path},
        QUERY_STRING   => $r->{query},
    );
    
    my ($handler, $params) = $router->dispatch($req);
    my $res;
    
    if ($handler) {
        eval { $res = $handler->($req, $params) };
        $res = json_err(500, "Internal Error: $@") if $@;
    } else {
        $res = json_err(404, "Not Found");
    }
    
    printf "%-4s %-25s => %d %s\n",
        $req->method, $req->path . ($r->{query} ? "?$r->{query}" : ""),
        $res->status, substr($res->body, 0, 50);
}

print "\nMini web framework demo complete!\n";
```

---

## สรุป Part 27

ใน Part นี้คุณได้เรียนรู้:
- ✅ Socket programming: TCP/UDP
- ✅ IO::Socket::INET
- ✅ LWP::UserAgent with retry logic
- ✅ HTTP::Tiny
- ✅ HTML parsing with regex
- ✅ MIME email building
- ✅ Simple HTTP server
- ✅ Net::SMTP / email templates
- ✅ JWT tokens and auth
- ✅ URI encoding/decoding
- ✅ Complete mini web framework

**ถัดไป: [Part 28 — Concurrency and Forking](part_28.md)**
