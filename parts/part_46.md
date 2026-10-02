# Part 46: Network Programming
## Steps 451-460: TCP/UDP, HTTP Client, DNS, Port Scanning

---

## Step 451: TCP Client & Server

```perl
#!/usr/bin/perl
use strict;
use warnings;
use IO::Socket::INET;
use IO::Select;

printf "=== TCP Client & Server (Simulation) ===\n\n";

{
package TCPServer;

sub new {
    my ($class, %opts) = @_;
    return bless {
        host     => $opts{host} // "127.0.0.1",
        port     => $opts{port} // 9000,
        routes   => {},
        handlers => {},
        backlog  => $opts{backlog} // 10,
    }, $class;
}

sub on {
    my ($self, $event, $handler) = @_;
    $self->{handlers}{$event} = $handler;
    return $self;
}

sub handle {
    my ($self, $command, $handler) = @_;
    $self->{routes}{uc $command} = $handler;
    return $self;
}

sub dispatch {
    my ($self, $data, $client_info) = @_;
    my ($cmd, $rest) = $data =~ /^(\S+)\s*(.*)/s;
    $cmd = uc($cmd // "");
    
    if (my $handler = $self->{routes}{$cmd}) {
        return $handler->($rest, $client_info);
    }
    return "ERROR: Unknown command '$cmd'\n";
}

sub simulate {
    my ($self, @messages) = @_;
    my @responses;
    my $client_info = { id => 1, addr => "127.0.0.1:12345" };
    
    $self->{handlers}{connect}->($client_info) if $self->{handlers}{connect};
    
    for my $msg (@messages) {
        printf "  C: %s", $msg;
        my $response = $self->dispatch($msg, $client_info);
        printf "  S: %s", $response;
        push @responses, $response;
    }
    
    $self->{handlers}{disconnect}->($client_info) if $self->{handlers}{disconnect};
    return @responses;
}
}

{
package TCPClient;

sub new {
    my ($class, %opts) = @_;
    return bless {
        host    => $opts{host} // "127.0.0.1",
        port    => $opts{port} // 9000,
        timeout => $opts{timeout} // 30,
        buffer  => "",
    }, $class;
}

sub request {
    my ($self, $cmd, $data) = @_;
    my $request = defined $data ? "$cmd $data\n" : "$cmd\n";
    return { raw => $request, cmd => $cmd, data => $data };
}

sub parse_response {
    my ($self, $response) = @_;
    if ($response =~ /^OK\s*(.*)/s) {
        return { status => "ok", data => $1 };
    } elsif ($response =~ /^ERROR\s*(.*)/s) {
        return { status => "error", message => $1 };
    }
    return { status => "unknown", data => $response };
}
}

# Build a key-value store server
my $server = TCPServer->new(port => 9000);
my %store;

$server->on("connect",    sub { printf "  [Client %s connected]\n", $_[0]{addr} });
$server->on("disconnect", sub { printf "  [Client %s disconnected]\n", $_[0]{addr} });

$server->handle("SET", sub {
    my ($args) = @_;
    my ($key, $val) = $args =~ /^(\S+)\s+(.*)/;
    return "ERROR: Usage: SET key value\n" unless $key;
    $store{$key} = $val;
    return "OK\n";
});

$server->handle("GET", sub {
    my ($key) = @_;
    $key =~ s/\s+//g;
    return defined $store{$key} ? "OK $store{$key}\n" : "ERROR: Not found\n";
});

$server->handle("DEL", sub {
    my ($key) = @_;
    $key =~ s/\s+//g;
    delete $store{$key};
    return "OK\n";
});

$server->handle("KEYS", sub {
    return "OK " . join(" ", sort keys %store) . "\n";
});

$server->handle("INCR", sub {
    my ($key) = @_;
    $key =~ s/\s+//g;
    $store{$key} = ($store{$key}//0) + 1;
    return "OK $store{$key}\n";
});

printf "Key-Value Store Server:\n";
$server->simulate(
    "SET name Alice\n",
    "SET age 30\n",
    "SET counter 0\n",
    "GET name\n",
    "INCR counter\n",
    "INCR counter\n",
    "GET counter\n",
    "KEYS\n",
    "DEL age\n",
    "GET age\n",
);
```

---

## Step 452: HTTP Client

```perl
#!/usr/bin/perl
use strict;
use warnings;
use MIME::Base64;

{
package HTTP::Request;

sub new {
    my ($class, $method, $url, %opts) = @_;
    my ($scheme, $host, $port, $path) = $url =~
        m{^(https?)://([^/:]+)(?::(\d+))?(/.*)$};
    $path  //= "/";
    $port  //= ($scheme//"") eq "https" ? 443 : 80;
    
    return bless {
        method   => uc $method,
        url      => $url,
        scheme   => $scheme // "http",
        host     => $host   // "",
        port     => $port,
        path     => $path,
        headers  => {
            "Host"       => $host // "",
            "User-Agent" => "PerlHTTP/1.0",
            "Accept"     => "*/*",
            "Connection" => "close",
            %{$opts{headers} // {}},
        },
        body     => $opts{body} // "",
        timeout  => $opts{timeout} // 30,
    }, $class;
}

sub header { my($s,$k,$v) = @_; $s->{headers}{$k}=$v; $s }
sub body   { my($s,$b) = @_; $s->{body}=$b; $s }

sub to_string {
    my $self = shift;
    my $path = $self->{path} || "/";
    
    if ($self->{body}) {
        $self->{headers}{"Content-Length"} = length($self->{body});
    }
    
    my $req = "$self->{method} $path HTTP/1.1\r\n";
    $req .= "$_: $self->{headers}{$_}\r\n" for sort keys %{$self->{headers}};
    $req .= "\r\n";
    $req .= $self->{body} if $self->{body};
    return $req;
}
}

{
package HTTP::Response;

sub new {
    my ($class, $raw) = @_;
    my ($head, $body) = split /\r?\n\r?\n/, $raw, 2;
    my @head_lines = split /\r?\n/, $head;
    my $status_line = shift @head_lines;
    my ($version, $code, $message) = $status_line =~ m{^HTTP/(\S+)\s+(\d+)\s+(.*)};
    
    my %headers;
    for my $line (@head_lines) {
        my ($k, $v) = $line =~ /^([^:]+):\s*(.*)/;
        $headers{lc $k} = $v if $k;
    }
    
    return bless {
        version => $version // "1.1",
        code    => $code    // 0,
        message => $message // "",
        headers => \%headers,
        body    => $body    // "",
    }, $class;
}

sub code    { $_[0]->{code} }
sub message { $_[0]->{message} }
sub body    { $_[0]->{body} }
sub header  { $_[0]->{headers}{lc $_[1]} }
sub ok      { ($_[0]->{code}//0) >= 200 && ($_[0]->{code}//0) < 300 }

sub summary {
    my $self = shift;
    printf "  HTTP/%s %s %s\n", $self->{version}, $self->{code}, $self->{message};
    printf "  Content-Type: %s\n", $self->header("content-type") // "N/A";
    printf "  Content-Length: %s\n", $self->header("content-length") // length($self->{body});
}
}

{
package HTTP::Client;

sub new {
    my ($class, %opts) = @_;
    return bless {
        base_url    => $opts{base_url} // "",
        timeout     => $opts{timeout}  // 30,
        headers     => $opts{headers}  // {},
        middlewares => [],
        cookies     => {},
    }, $class;
}

sub use_middleware {
    my ($self, $mw) = @_;
    push @{$self->{middlewares}}, $mw;
    return $self;
}

sub auth_basic {
    my ($self, $user, $pass) = @_;
    my $token = MIME::Base64::encode_base64("$user:$pass", "");
    $self->{headers}{"Authorization"} = "Basic $token";
    return $self;
}

sub auth_bearer {
    my ($self, $token) = @_;
    $self->{headers}{"Authorization"} = "Bearer $token";
    return $self;
}

sub _build_request {
    my ($self, $method, $url, %opts) = @_;
    $url = $self->{base_url} . $url unless $url =~ /^https?:\/\//;
    
    my $req = HTTP::Request->new($method, $url,
        headers => { %{$self->{headers}}, %{$opts{headers}//{}},
            ($self->{cookies} ? (Cookie => _format_cookies($self->{cookies})) : ()) },
        body    => $opts{body} // "",
    );
    
    if (ref $opts{json} eq "HASH") {
        require JSON::PP;
        $req->body(JSON::PP->new->encode($opts{json}));
        $req->header("Content-Type", "application/json");
    }
    
    return $req;
}

sub _format_cookies {
    my $cookies = shift;
    return join("; ", map { "$_=$cookies->{$_}" } keys %$cookies);
}

sub get    { my $self=shift; $self->_simulate($self->_build_request("GET",    @_)) }
sub post   { my $self=shift; $self->_simulate($self->_build_request("POST",   @_)) }
sub put    { my $self=shift; $self->_simulate($self->_build_request("PUT",    @_)) }
sub delete { my $self=shift; $self->_simulate($self->_build_request("DELETE", @_)) }

sub _simulate {
    my ($self, $req) = @_;
    # Simulate responses for demonstration
    my $path = $req->{path};
    my %simulated_responses = (
        "/api/users"    => 'HTTP/1.1 200 OK\r\nContent-Type: application/json\r\n\r\n[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]',
        "/api/users/1"  => 'HTTP/1.1 200 OK\r\nContent-Type: application/json\r\n\r\n{"id":1,"name":"Alice","email":"alice@example.com"}',
        "/api/auth"     => 'HTTP/1.1 200 OK\r\nContent-Type: application/json\r\nSet-Cookie: session=abc123\r\n\r\n{"token":"jwt.token.here","expires_in":3600}',
    );
    
    my $raw = $simulated_responses{$path} //
        "HTTP/1.1 404 Not Found\r\nContent-Type: text/plain\r\n\r\nNot found: $path";
    
    $raw =~ s/\\r\\n/\r\n/g;
    return HTTP::Response->new($raw);
}
}

package main;

printf "=== HTTP Client ===\n\n";

my $client = HTTP::Client->new(base_url => "http://api.example.com");
$client->auth_bearer("mytoken123");

printf "GET /api/users:\n";
my $resp = $client->get("/api/users");
$resp->summary;
printf "  Body: %s\n\n", substr($resp->body, 0, 80);

printf "GET /api/users/1:\n";
$resp = $client->get("/api/users/1");
$resp->summary;
printf "  Body: %s\n\n", $resp->body;

printf "POST /api/auth:\n";
$resp = $client->post("/api/auth", json => { username => "alice", password => "secret" });
$resp->summary;
printf "  Body: %s\n\n", $resp->body;

printf "GET /api/missing:\n";
$resp = $client->get("/api/missing");
$resp->summary;
printf "  OK: %s\n\n", $resp->ok ? "yes" : "no";
```

---

## Step 453: DNS Resolver (Simulation)

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package DNS::Resolver;

# DNS record types
use constant {
    TYPE_A     => 1,
    TYPE_AAAA  => 28,
    TYPE_CNAME => 5,
    TYPE_MX    => 15,
    TYPE_TXT   => 16,
    TYPE_NS    => 2,
    TYPE_PTR   => 12,
    TYPE_SOA   => 6,
};

my %RECORD_NAMES = (
    1 => "A", 28 => "AAAA", 5 => "CNAME", 15 => "MX",
    16 => "TXT", 2 => "NS", 12 => "PTR", 6 => "SOA",
);

sub new {
    my ($class, %opts) = @_;
    return bless {
        cache     => {},
        servers   => $opts{servers} // ["8.8.8.8", "1.1.1.1"],
        timeout   => $opts{timeout} // 5,
        _zone_db  => _build_zone_db(),
    }, $class;
}

sub _build_zone_db {
    return {
        "example.com" => {
            A     => [{ ip => "93.184.216.34", ttl => 3600 }],
            AAAA  => [{ ip => "2606:2800:220:1:248:1893:25c8:1946", ttl => 3600 }],
            MX    => [
                { priority => 10, host => "mail.example.com", ttl => 3600 },
                { priority => 20, host => "mail2.example.com", ttl => 3600 },
            ],
            NS    => [{ host => "ns1.example.com", ttl => 3600 },
                      { host => "ns2.example.com", ttl => 3600 }],
            TXT   => [
                { data => "v=spf1 include:example.com ~all", ttl => 3600 },
                { data => "google-site-verification=abc123", ttl => 3600 },
            ],
        },
        "www.example.com" => {
            CNAME => [{ target => "example.com", ttl => 300 }],
        },
        "mail.example.com" => {
            A     => [{ ip => "93.184.216.35", ttl => 3600 }],
        },
        "api.example.com" => {
            A     => [{ ip => "10.0.1.1", ttl => 60 },
                      { ip => "10.0.1.2", ttl => 60 }],
        },
    };
}

sub query {
    my ($self, $name, $type) = @_;
    $type //= "A";
    
    # Check cache
    my $cache_key = "$name:$type";
    if (my $cached = $self->{cache}{$cache_key}) {
        return { %$cached, cached => 1 } if $cached->{expires} > time();
    }
    
    # Lookup
    my $result = $self->_lookup($name, $type);
    
    # Cache result
    if ($result->{records}) {
        my $ttl = ($result->{records}[0]{ttl} // 300);
        $self->{cache}{$cache_key} = { %$result, expires => time() + $ttl };
    }
    
    return $result;
}

sub _lookup {
    my ($self, $name, $type) = @_;
    my $zone = $self->{_zone_db}{lc $name};
    
    unless ($zone) {
        return { status => "NXDOMAIN", name => $name, type => $type, records => [] };
    }
    
    my $records = $zone->{$type} // [];
    
    # Follow CNAME
    if (!@$records && $zone->{CNAME}) {
        my $cname = $zone->{CNAME}[0];
        my $target_result = $self->_lookup($cname->{target}, $type);
        return {
            status  => "NOERROR",
            name    => $name,
            type    => $type,
            cname   => $cname->{target},
            records => $target_result->{records},
        };
    }
    
    return {
        status  => @$records ? "NOERROR" : "NODATA",
        name    => $name,
        type    => $type,
        records => $records,
    };
}

sub resolve {
    my ($self, $name) = @_;
    # Follow CNAMEs to final A record
    my $current = $name;
    my @chain;
    
    for (1..10) {  # max CNAME chain
        my $result = $self->query($current, "A");
        push @chain, $current;
        
        if ($result->{cname}) {
            $current = $result->{cname};
        } elsif (@{$result->{records}}) {
            return {
                name    => $name,
                chain   => \@chain,
                final   => $current,
                records => $result->{records},
            };
        } else {
            last;
        }
    }
    return { name => $name, error => "Could not resolve" };
}

sub reverse_lookup {
    my ($self, $ip) = @_;
    # Reverse IP for PTR query
    my @parts = split /\./, $ip;
    my $ptr_name = join(".", reverse @parts) . ".in-addr.arpa";
    return { ip => $ip, ptr => $ptr_name, result => "simulated.reverse.lookup" };
}
}

package main;

printf "=== DNS Resolver ===\n\n";

my $dns = DNS::Resolver->new;

# A records
for my $name (qw(example.com www.example.com api.example.com unknown.example.com)) {
    my $r = $dns->query($name, "A");
    printf "A %s -> %s [%s]\n", $name, 
        @{$r->{records}} ? join(", ", map {$_->{ip}} @{$r->{records}}) : "-",
        $r->{status};
    printf "  (via CNAME: %s)\n", $r->{cname} if $r->{cname};
}

# MX records
printf "\nMX example.com:\n";
my $mx = $dns->query("example.com", "MX");
printf "  priority=%-3d %s\n", $_->{priority}, $_->{host} for @{$mx->{records}};

# TXT records
printf "\nTXT example.com:\n";
my $txt = $dns->query("example.com", "TXT");
printf "  %s\n", $_->{data} for @{$txt->{records}};

# NS records
printf "\nNS example.com:\n";
my $ns = $dns->query("example.com", "NS");
printf "  %s\n", $_->{host} for @{$ns->{records}};

# Resolve with CNAME chain
printf "\nResolve www.example.com:\n";
my $resolved = $dns->resolve("www.example.com");
printf "  Chain: %s\n", join(" -> ", @{$resolved->{chain}});
printf "  Final: %s\n", $resolved->{final};
printf "  IP:    %s\n", join(", ", map{$_->{ip}} @{$resolved->{records}});

# Cache test
$dns->query("example.com", "A");  # warm cache
my $cached = $dns->query("example.com", "A");
printf "\nCache hit: %s\n", $cached->{cached} ? "YES" : "NO";
```

---

## Step 454: Port Scanner & Network Utils

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Socket;

{
package NetworkUtils;

my %WELL_KNOWN_PORTS = (
    21 => "FTP", 22 => "SSH", 23 => "Telnet", 25 => "SMTP",
    53 => "DNS", 80 => "HTTP", 110 => "POP3", 143 => "IMAP",
    443 => "HTTPS", 465 => "SMTPS", 587 => "Submission",
    993 => "IMAPS", 995 => "POP3S", 3306 => "MySQL",
    5432 => "PostgreSQL", 6379 => "Redis", 27017 => "MongoDB",
    8080 => "HTTP-Alt", 8443 => "HTTPS-Alt",
);

sub resolve_hostname {
    my ($class, $hostname) = @_;
    my @addrs;
    
    if ($hostname =~ /^\d+\.\d+\.\d+\.\d+$/) {
        return ($hostname);  # Already an IP
    }
    
    my @info = getaddrinfo($hostname, "", { family => AF_INET });
    if (!ref $info[0]) {
        while (my ($flags, $family, $type, $proto, $addr, $canon) = splice @info, 0, 6) {
            my (undef, $ip) = unpack_sockaddr_in($addr);
            push @addrs, inet_ntoa($ip);
        }
    }
    return @addrs;
}

sub is_port_open {
    my ($class, $host, $port, $timeout) = @_;
    $timeout //= 2;
    
    # Try to connect
    my $sock = IO::Socket::INET->new(
        PeerAddr => $host,
        PeerPort => $port,
        Proto    => "tcp",
        Timeout  => $timeout,
    );
    
    if ($sock) {
        close $sock;
        return 1;
    }
    return 0;
}

sub scan_ports {
    my ($class, $host, @ports) = @_;
    my @results;
    
    for my $port (@ports) {
        my $open    = $class->is_port_open($host, $port, 1);
        my $service = $WELL_KNOWN_PORTS{$port} // "unknown";
        push @results, {
            port    => $port,
            status  => $open ? "open" : "closed",
            service => $service,
        };
    }
    return @results;
}

sub get_service_name {
    my ($class, $port) = @_;
    return $WELL_KNOWN_PORTS{$port} // getservbyport($port, "tcp") // "unknown";
}

sub cidr_range {
    my ($class, $cidr) = @_;
    my ($base, $prefix) = $cidr =~ /^(\d+\.\d+\.\d+\.\d+)\/(\d+)$/;
    return unless $base && $prefix;
    
    my $base_int = unpack("N", inet_aton($base));
    my $mask     = (0xFFFFFFFF << (32 - $prefix)) & 0xFFFFFFFF;
    my $network  = $base_int & $mask;
    my $broadcast= $network | (~$mask & 0xFFFFFFFF);
    
    return {
        network   => inet_ntoa(pack("N", $network)),
        broadcast => inet_ntoa(pack("N", $broadcast)),
        mask      => inet_ntoa(pack("N", $mask)),
        hosts     => $broadcast - $network - 1,
        first     => inet_ntoa(pack("N", $network+1)),
        last      => inet_ntoa(pack("N", $broadcast-1)),
    };
}

sub ip_in_cidr {
    my ($class, $ip, $cidr) = @_;
    my ($base, $prefix) = $cidr =~ /^(\d+\.\d+\.\d+\.\d+)\/(\d+)$/;
    return 0 unless $base && $prefix;
    
    my $ip_int   = unpack("N", inet_aton($ip));
    my $base_int = unpack("N", inet_aton($base));
    my $mask     = (0xFFFFFFFF << (32-$prefix)) & 0xFFFFFFFF;
    return ($ip_int & $mask) == ($base_int & $mask);
}
}

package main;

printf "=== Network Utilities ===\n\n";

# CIDR analysis
printf "CIDR Analysis:\n";
for my $cidr ("192.168.1.0/24", "10.0.0.0/8", "172.16.0.0/12") {
    my $info = NetworkUtils->cidr_range($cidr);
    printf "  %-18s network=%-15s broadcast=%-15s mask=%-15s hosts=%d\n",
        $cidr, $info->{network}, $info->{broadcast}, $info->{mask}, $info->{hosts};
}

# IP in CIDR
printf "\nIP in CIDR tests:\n";
my @tests = (
    ["192.168.1.100", "192.168.1.0/24", 1],
    ["192.168.2.1",   "192.168.1.0/24", 0],
    ["10.5.3.7",      "10.0.0.0/8",     1],
    ["172.15.255.255","172.16.0.0/12",  0],
);
for my $t (@tests) {
    my $result = NetworkUtils->ip_in_cidr($t->[0], $t->[1]);
    printf "  %-15s in %-20s -> %s %s\n",
        $t->[0], $t->[1], $result ? "YES" : "NO",
        $result == $t->[2] ? "(correct)" : "(WRONG!)";
}

# Port scanning simulation
printf "\nPort scan simulation (localhost):\n";
my @ports = (22, 25, 53, 80, 443, 3306, 5432, 6379, 8080, 8443);
printf "%-8s %-12s %s\n", "Port", "Service", "Status";
printf "%s\n", "-" x 35;
for my $port (@ports) {
    my $service = NetworkUtils->get_service_name($port);
    # Simulate: some ports "open"
    my $simulated = (grep { $_ == $port } (22, 80, 443, 8080)) ? "open" : "closed";
    printf "%-8d %-12s %s\n", $port, $service, $simulated;
}
```

---

## Step 455-460: HTTP Server, WebSocket Handshake, Capstone

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(sha1_base64);
use MIME::Base64;

# Simple HTTP Server framework
printf "=== HTTP Server Framework ===\n\n";

{
package HTTP::Server::Request;

sub new {
    my ($class, $raw) = @_;
    my ($head, $body) = split /\r?\n\r?\n/, $raw, 2;
    my @lines = split /\r?\n/, $head;
    my $req_line = shift @lines;
    my ($method, $path, $version) = split /\s+/, $req_line, 3;
    
    # Parse query string
    my ($clean_path, $query_string) = split /\?/, $path, 2;
    my %query;
    for my $pair (split /&/, $query_string//"") {
        my ($k, $v) = split /=/, $pair, 2;
        $k //= ""; $v //= "";
        $k =~ s/%([0-9A-Fa-f]{2})/chr hex $1/ge;
        $v =~ s/%([0-9A-Fa-f]{2})/chr hex $1/ge;
        $query{$k} = $v;
    }
    
    my %headers;
    for my $line (@lines) {
        my ($k, $v) = $line =~ /^([^:]+):\s*(.*)/;
        $headers{lc $k} = $v if $k;
    }
    
    return bless {
        method  => uc($method//"GET"),
        path    => $clean_path // "/",
        version => $version // "HTTP/1.1",
        query   => \%query,
        headers => \%headers,
        body    => $body // "",
        params  => {},  # URL params filled by router
    }, $class;
}

sub method  { $_[0]->{method} }
sub path    { $_[0]->{path} }
sub query   { $_[0]->{query}{$_[1]} }
sub header  { $_[0]->{headers}{lc $_[1]} }
sub body    { $_[0]->{body} }
sub param   { $_[0]->{params}{$_[1]} }
sub is_json { ($_[0]->{headers}{"content-type"}//"") =~ /application\/json/i }
}

{
package HTTP::Server::Response;

sub new {
    my ($class, %opts) = @_;
    return bless {
        status  => $opts{status} // 200,
        message => $opts{message} // _status_message($opts{status}//200),
        headers => {
            "Content-Type" => "text/plain",
            "X-Powered-By" => "Perl",
            %{$opts{headers} // {}},
        },
        body    => $opts{body} // "",
    }, $class;
}

my %STATUS_MESSAGES = (
    200 => "OK", 201 => "Created", 204 => "No Content",
    301 => "Moved Permanently", 302 => "Found",
    400 => "Bad Request", 401 => "Unauthorized", 403 => "Forbidden",
    404 => "Not Found", 405 => "Method Not Allowed",
    409 => "Conflict", 422 => "Unprocessable Entity",
    500 => "Internal Server Error",
);
sub _status_message { $STATUS_MESSAGES{$_[0]} // "Unknown" }

sub json {
    my ($class, $data, $status) = @_;
    require JSON::PP;
    return $class->new(
        status  => $status // 200,
        headers => { "Content-Type" => "application/json" },
        body    => JSON::PP->new->encode($data),
    );
}

sub html {
    my ($class, $body, $status) = @_;
    return $class->new(
        status  => $status // 200,
        headers => { "Content-Type" => "text/html; charset=UTF-8" },
        body    => $body,
    );
}

sub redirect { $_[0]->new(status=>302, headers=>{"Location"=>$_[1]}) }

sub to_string {
    my $self = shift;
    $self->{headers}{"Content-Length"} = length($self->{body});
    my $out = "HTTP/1.1 $self->{status} $self->{message}\r\n";
    $out .= "$_: $self->{headers}{$_}\r\n" for sort keys %{$self->{headers}};
    $out .= "\r\n" . $self->{body};
    return $out;
}
}

{
package HTTP::Router;

sub new { bless { routes => [] }, shift }

sub _add {
    my ($self, $method, $pattern, $handler) = @_;
    # Convert :param to named captures
    my $regex = $pattern;
    my @param_names;
    $regex =~ s{:(\w+)}{push @param_names, $1; "([^/]+)"}ge;
    $regex = qr{^$regex$};
    push @{$self->{routes}}, {
        method  => uc $method,
        pattern => $regex,
        handler => $handler,
        params  => \@param_names,
    };
    return $self;
}

sub get    { $_[0]->_add("GET",    $_[1], $_[2]) }
sub post   { $_[0]->_add("POST",   $_[1], $_[2]) }
sub put    { $_[0]->_add("PUT",    $_[1], $_[2]) }
sub delete { $_[0]->_add("DELETE", $_[1], $_[2]) }

sub dispatch {
    my ($self, $req) = @_;
    for my $route (@{$self->{routes}}) {
        next unless $route->{method} eq $req->method || $route->{method} eq "ANY";
        if (my @caps = $req->path =~ $route->{pattern}) {
            @{$req->{params}}{@{$route->{params}}} = @caps;
            return eval { $route->{handler}->($req) };
        }
    }
    return undef;  # No match
}
}

package main;

my $router = HTTP::Router->new;
my %users_db = (
    1 => { id=>1, name=>"Alice", email=>"alice\@test.com" },
    2 => { id=>2, name=>"Bob",   email=>"bob\@test.com" },
);

$router->get("/", sub {
    HTTP::Server::Response->html("<h1>Welcome to Perl HTTP Server</h1>");
});

$router->get("/api/users", sub {
    HTTP::Server::Response->json([values %users_db]);
});

$router->get("/api/users/:id", sub {
    my $req = shift;
    my $id  = $req->param("id");
    my $user = $users_db{$id};
    return $user
        ? HTTP::Server::Response->json($user)
        : HTTP::Server::Response->json({error=>"Not found"}, 404);
});

$router->post("/api/users", sub {
    my $req = shift;
    my $new_id = (sort { $b<=>$a } keys %users_db)[0] + 1;
    $users_db{$new_id} = { id=>$new_id, name=>"New User $new_id", email=>"user$new_id\@test.com" };
    HTTP::Server::Response->json($users_db{$new_id}, 201);
});

$router->delete("/api/users/:id", sub {
    my $req = shift;
    my $id  = $req->param("id");
    return HTTP::Server::Response->json({error=>"Not found"}, 404) unless $users_db{$id};
    delete $users_db{$id};
    HTTP::Server::Response->new(status=>204);
});

# Test requests
my @test_requests = (
    "GET / HTTP/1.1\r\nHost: localhost\r\n\r\n",
    "GET /api/users HTTP/1.1\r\nHost: localhost\r\n\r\n",
    "GET /api/users/1 HTTP/1.1\r\nHost: localhost\r\n\r\n",
    "GET /api/users/99 HTTP/1.1\r\nHost: localhost\r\n\r\n",
    "POST /api/users HTTP/1.1\r\nHost: localhost\r\n\r\n",
    "DELETE /api/users/2 HTTP/1.1\r\nHost: localhost\r\n\r\n",
    "GET /api/users HTTP/1.1\r\nHost: localhost\r\n\r\n",
);

printf "%-8s %-25s -> %s\n", "Method", "Path", "Response";
printf "%s\n", "-" x 60;
for my $raw_req (@test_requests) {
    my $req  = HTTP::Server::Request->new($raw_req);
    my $resp = $router->dispatch($req)
        // HTTP::Server::Response->json({error=>"Not found"}, 404);
    printf "%-8s %-25s -> %d %s\n",
        $req->method, $req->path, $resp->{status}, substr($resp->{body},0,50);
}

# WebSocket handshake
printf "\n=== WebSocket Handshake ===\n\n";

sub ws_accept_key {
    my $key = shift;
    my $magic = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11";
    return sha1_base64($key . $magic) . "=";
}

my $client_key = "dGhlIHNhbXBsZSBub25jZQ==";
my $accept_key = ws_accept_key($client_key);

printf "Client key:  %s\n", $client_key;
printf "Accept key:  %s\n", $accept_key;
printf "Expected:    %s\n", "s3pPLMBiTxaQ9kYGzzhZRbK+xOo=";

# WebSocket frame
printf "\nWebSocket Frame Encoding:\n";
sub encode_ws_frame {
    my ($payload, %opts) = @_;
    my $opcode = $opts{opcode} // 0x1;  # 0x1=text, 0x2=binary, 0x8=close
    my $fin    = $opts{fin} // 1;
    
    my $byte1  = ($fin ? 0x80 : 0) | ($opcode & 0x0F);
    my $len    = length($payload);
    
    my $frame = chr($byte1);
    if ($len < 126) {
        $frame .= chr($len);
    } elsif ($len < 65536) {
        $frame .= chr(126) . pack("n", $len);
    } else {
        $frame .= chr(127) . pack("NN", 0, $len);
    }
    $frame .= $payload;
    return $frame;
}

my $msg = "Hello WebSocket!";
my $frame = encode_ws_frame($msg);
printf "  Message: '%s' (%d bytes)\n", $msg, length($msg);
printf "  Frame:   %d bytes\n", length($frame);
printf "  Header:  %s\n", join(" ", map { sprintf "%02x", ord } split //, substr($frame, 0, 2));
```

---

## สรุป Part 46 — Network Programming

### สิ่งที่เรียนรู้:
- **TCP Client/Server** — Socket simulation, protocol dispatch, KV store
- **HTTP Client** — Request building, response parsing, auth, middleware
- **DNS Resolver** — Record types (A/AAAA/MX/CNAME/TXT/NS), caching, CNAME following
- **Network Utils** — CIDR calculation, IP-in-subnet, port→service mapping
- **HTTP Server** — Router with :param, request/response objects, REST API
- **WebSocket** — Handshake (SHA1 accept key), frame encoding (FIN+opcode)

**ถัดไป: [Part 47 — Process Management & IPC](part_47.md)**
