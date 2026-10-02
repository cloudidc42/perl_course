# Part 35: Mojolicious Web Framework
## Steps 341-350: Mojolicious — The Fun Web Framework

---

## Step 341: Mojolicious Architecture

```perl
#!/usr/bin/perl
# mojo_demo.pl — Mojolicious-style framework simulation
# Real Mojolicious: cpan Mojolicious
use strict;
use warnings;

# Mojo::Base simulation
{
package Mojo::Base;

sub import {
    my ($class, $base) = @_;
    my $caller = caller;
    no strict 'refs';
    if ($base && $base eq '-strict') {
        # Just enable strict/warnings
        return;
    }
    if ($base) {
        push @{"${caller}::ISA"}, $base;
    }
    *{"${caller}::has"} = sub {
        my ($name, %opts) = @_;
        my $default = $opts{default};
        no strict 'refs';
        *{"${caller}::$name"} = sub {
            my $self = shift;
            if (@_) { $self->{$name} = shift; return $self }
            unless (exists $self->{$name}) {
                $self->{$name} = ref($default) eq 'CODE' ? $default->($self) : $default;
            }
            return $self->{$name};
        };
    };
}

sub new {
    my ($class, %args) = @_;
    my $self = bless {}, $class;
    $self->{$_} = $args{$_} for keys %args;
    return $self;
}
}

# Mojo::Controller (context)
{
package Mojo::Controller;
use parent -norequire, 'Mojo::Base';

sub new {
    my ($class, %args) = @_;
    return bless { %args, stash => {}, rendered => 0 }, $class;
}

sub req  { $_[0]->{request}  }
sub res  { $_[0]->{response} }
sub app  { $_[0]->{app}      }

sub param {
    my ($self, $name) = @_;
    return $self->{request}{params}{$name};
}

sub stash {
    my $self = shift;
    return @_ ? ($self->{stash}{$_[0]} = $_[1], $self) : $self->{stash};
}

sub render {
    my ($self, %args) = @_;
    $self->{rendered} = 1;
    $self->{response}{status} = $args{status} // 200;
    
    if ($args{json}) {
        require JSON::PP;
        $self->{response}{body}    = JSON::PP->new->utf8->encode($args{json});
        $self->{response}{content_type} = "application/json";
    } elsif ($args{text}) {
        $self->{response}{body}    = $args{text};
        $self->{response}{content_type} = "text/plain";
    } elsif ($args{inline}) {
        $self->{response}{body}    = $args{inline};
        $self->{response}{content_type} = "text/html";
    } else {
        $self->{response}{body}    = $args{data} // "";
    }
}

sub redirect_to {
    my ($self, $url) = @_;
    $self->{rendered} = 1;
    $self->{response}{status} = 302;
    $self->{response}{headers}{Location} = $url;
}

sub respond_to {
    my ($self, %formats) = @_;
    my $accept = $self->{request}{accept} // "html";
    my $handler = $formats{$accept} // $formats{any};
    $handler->() if $handler;
}
}

# Mojolicious Application
{
package Mojolicious;

sub new {
    my ($class, %args) = @_;
    return bless {
        routes  => [],
        hooks   => { before_dispatch => [], after_dispatch => [] },
        helpers => {},
        config  => { mode => "development", %{$args{config}//{}}, },
    }, $class;
}

sub routes { $_[0] }  # returns self for chaining

sub get    { $_[0]->_add_route("GET",    $_[1], $_[2]) }
sub post   { $_[0]->_add_route("POST",   $_[1], $_[2]) }
sub put    { $_[0]->_add_route("PUT",    $_[1], $_[2]) }
sub patch  { $_[0]->_add_route("PATCH",  $_[1], $_[2]) }
sub delete { $_[0]->_add_route("DELETE", $_[1], $_[2]) }
sub any    {
    my ($self, $methods, $path, $handler) = @_;
    for my $m (ref $methods ? @$methods : ($methods)) {
        $self->_add_route(uc($m), $path, $handler);
    }
    return $self;
}

sub _add_route {
    my ($self, $method, $path, $handler) = @_;
    my $regex = $path;
    $regex =~ s{:(\w+)}{(?<$1>[^/]+)}g;
    $regex =~ s{\*(\w+)}{(?<$1>.+)}g;
    push @{$self->{routes}}, {
        method  => $method,
        path    => $path,
        regex   => qr{^$regex$},
        handler => $handler,
    };
    return $self;
}

sub hook {
    my ($self, $name, $code) = @_;
    push @{$self->{hooks}{$name}}, $code;
    return $self;
}

sub helper {
    my ($self, $name, $code) = @_;
    $self->{helpers}{$name} = $code;
    return $self;
}

sub config {
    my ($self, $key) = @_;
    return $key ? $self->{config}{$key} : $self->{config};
}

sub dispatch {
    my ($self, $method, $path, %opts) = @_;
    $method = uc($method);
    
    my $request  = { method => $method, path => $path, %opts, params => $opts{params}//{}};
    my $response = { status => 200, body => "", headers => {}, content_type => "text/html" };
    
    my $ctx = Mojo::Controller->new(
        request  => $request,
        response => $response,
        app      => $self,
    );
    
    # Before dispatch hooks
    for my $hook (@{$self->{hooks}{before_dispatch}}) {
        $hook->($ctx);
        return $response if $ctx->{rendered};
    }
    
    # Route matching
    for my $route (@{$self->{routes}}) {
        next unless $route->{method} eq $method;
        next unless $path =~ $route->{regex};
        
        $ctx->{request}{captures} = {%+};
        
        eval { $route->{handler}->($ctx) };
        if ($@) {
            $ctx->render(text => "Error: $@", status => 500);
        }
        
        last if $ctx->{rendered};
    }
    
    unless ($ctx->{rendered}) {
        $response->{status} = 404;
        $response->{body}   = "Not Found: $method $path";
    }
    
    # After dispatch hooks
    for my $hook (@{$self->{hooks}{after_dispatch}}) {
        $hook->($ctx);
    }
    
    return $response;
}

sub start {
    my $self = shift;
    printf "Mojolicious app '%s' ready on port %d\n",
        $self->config("name") // "app",
        $self->config("port") // 3000;
}
}

package main;

# Create Mojolicious app
my $app = Mojolicious->new(config => { name => "MyMojo", port => 3000 });

# Hooks
$app->hook(before_dispatch => sub {
    my $ctx = shift;
    # Add request ID
    $ctx->{request}{id} = sprintf "%08x", rand(0xFFFFFFFF);
});

# Helpers
$app->helper(is_authenticated => sub {
    my $ctx   = shift;
    return defined $ctx->{request}{token};
});

$app->helper(current_user => sub {
    my $ctx = shift;
    return $ctx->{request}{user} // { name => "guest" };
});

# Routes
$app->get("/" => sub {
    my $c = shift;
    $c->render(text => "Welcome to Mojolicious! Request ID: " . $c->req->{id});
});

$app->get("/hello/:name" => sub {
    my $c    = shift;
    my $name = $c->req->{captures}{name};
    $c->render(text => "Hello, $name!");
});

$app->get("/api/status" => sub {
    my $c = shift;
    $c->render(json => {
        status  => "ok",
        app     => $c->app->config("name"),
        request => $c->req->{id},
    });
});

$app->post("/api/data" => sub {
    my $c = shift;
    my $name = $c->param("name") // "unknown";
    $c->render(json => { received => { name => $name }, status => "saved" });
});

$app->get("/redirect" => sub {
    my $c = shift;
    $c->redirect_to("/");
});

$app->start;

printf "\n=== Mojolicious Dispatch Tests ===\n\n";

my @tests = (
    ["GET",  "/",            {}],
    ["GET",  "/hello/Perl",  {}],
    ["GET",  "/hello/World", {}],
    ["GET",  "/api/status",  {}],
    ["POST", "/api/data",    { params => { name => "Alice" } }],
    ["GET",  "/redirect",    {}],
    ["GET",  "/not-found",   {}],
);

for my $t (@tests) {
    my ($method, $path, $opts) = @$t;
    my $res = $app->dispatch($method, $path, %$opts);
    printf "%s %-20s => %d: %.60s\n",
        $method, $path, $res->{status},
        $res->{body} =~ s/\n/ /gr;
}
```

---

## Step 342: Mojo::Template

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Mojo::Template;

sub new { bless { tag_start => '<%', tag_end => '%>' }, $_[0] }

sub render {
    my ($self, $template, %vars) = @_;
    
    # Make vars available as $self->{vars} for the template
    my $code = 'my $out = ""; my %_v = %{$_[0]}; ';
    $code   .= "my \$$_ = \$_v{$_}; " for keys %vars;
    
    my $t = $template;
    
    # <%= expr %> — output escaped
    $t =~ s/<%=\s*(.*?)\s*%>/\$out .= _esc($1); /gs;
    
    # <%== expr %> — raw output
    $t =~ s/<%==\s*(.*?)\s*%>/\$out .= ($1)/gs;
    
    # <% code %> — execute
    $t =~ s/<%\s*(.*?)\s*%>/$1; /gs;
    
    # Remaining text
    $t =~ s/([^;]+)/\$out .= q{$1}; /g unless $t =~ /\$out/;
    
    # Actually let's use a proper approach with split
    return $self->_render_proper($template, %vars);
}

sub _render_proper {
    my ($self, $template, %vars) = @_;
    
    my $output = "";
    my @parts  = split /(<%.+?%>)/s, $template;
    
    for my $part (@parts) {
        if ($part =~ /^<%=(.*?)%>$/s) {
            my $expr = $1;
            $expr =~ s/^\s+|\s+$//g;
            my $val = _eval_expr($expr, \%vars);
            $output .= _html_escape($val // "");
        } elsif ($part =~ /^<%==(.*?)%>$/s) {
            my $expr = $1;
            $expr =~ s/^\s+|\s+$//g;
            $output .= _eval_expr($expr, \%vars) // "";
        } elsif ($part =~ /^<%(.*?)%>$/s) {
            # Code block — affects flow via stash
            # For simplicity, skip execution blocks here
        } else {
            $output .= $part;
        }
    }
    
    return $output;
}

sub _html_escape {
    my $s = shift // "";
    $s =~ s/&/&amp;/g; $s =~ s/</&lt;/g; $s =~ s/>/&gt;/g; $s =~ s/"/&quot;/g;
    return $s;
}

sub _eval_expr {
    my ($expr, $vars) = @_;
    local $_ = undef;
    my $result = eval {
        local %_ = %$vars;
        my $code = join("", map { "my \$$_ = \$_{'$_'}; " } keys %$vars);
        eval "$code; $expr";
    };
    return $result;
}

sub render_file {
    my ($self, $file, %vars) = @_;
    open my $fh, "<", $file or die "Cannot read $file: $!";
    my $tmpl = do { local $/; <$fh> };
    close $fh;
    return $self->render($tmpl, %vars);
}
}

package main;

my $mt = Mojo::Template->new;

# Basic template
my $html = $mt->render(
    '<h1>Hello, <%= $name %>!</h1><p>Version: <%= $version %></p>',
    name    => '<script>alert(1)</script>',  # Will be escaped
    version => "1.0",
);
printf "Rendered: %s\n\n", $html;

# Template with data
my $list_tmpl = '<ul>' . join("", map {
    "<li>$_</li>"
} qw(Perl Python Ruby)) . '</ul>';
printf "List: %s\n", $list_tmpl;

# EP template simulation (Embedded Perl)
sub ep_render {
    my ($template, %vars) = @_;
    my $out = $template;
    
    # Replace %= expr with value
    $out =~ s/<%=\s*(.+?)\s*%>/do {
        my $v = eval { local %_ = %vars; my $c = join "", map { "my \$$_ = \$_{'$_'}; " } keys %vars; eval "$c $1" };
        $v // ""
    }/ge;
    
    return $out;
}

my $result = ep_render('<title><%= $title %></title>', title => "My Page");
printf "EP: %s\n", $result;
```

---

## Step 343: WebSocket Simulation

```perl
#!/usr/bin/perl
use strict;
use warnings;

# WebSocket-style event-based communication
{
package WebSocket::Server;

sub new {
    my ($class) = @_;
    return bless {
        connections => {},
        handlers    => {},
        rooms       => {},
        next_id     => 1,
    }, $class;
}

sub on {
    my ($self, $event, $handler) = @_;
    $self->{handlers}{$event} = $handler;
    return $self;
}

sub connect {
    my ($self, %opts) = @_;
    my $id = $self->{next_id}++;
    $self->{connections}{$id} = {
        id       => $id,
        user     => $opts{user},
        rooms    => [],
        messages => [],
    };
    
    # Fire connect event
    if (my $h = $self->{handlers}{connect}) {
        $h->($self->_make_socket($id));
    }
    
    return $id;
}

sub disconnect {
    my ($self, $id) = @_;
    my $conn = $self->{connections}{$id} or return;
    
    # Leave all rooms
    for my $room (@{$conn->{rooms}}) {
        $self->leave_room($id, $room);
    }
    
    # Fire disconnect event
    if (my $h = $self->{handlers}{disconnect}) {
        $h->($self->_make_socket($id));
    }
    
    delete $self->{connections}{$id};
}

sub send_to {
    my ($self, $id, $event, $data) = @_;
    my $conn = $self->{connections}{$id} or return;
    push @{$conn->{messages}}, { event => $event, data => $data, ts => time() };
}

sub broadcast {
    my ($self, $event, $data, %opts) = @_;
    for my $id (keys %{$self->{connections}}) {
        next if $opts{except} && $id == $opts{except};
        $self->send_to($id, $event, $data);
    }
}

sub join_room {
    my ($self, $id, $room) = @_;
    push @{$self->{connections}{$id}{rooms}}, $room
        unless grep { $_ eq $room } @{$self->{connections}{$id}{rooms}};
    push @{$self->{rooms}{$room}}, $id
        unless grep { $_ == $id } @{$self->{rooms}{$room}//[]};
}

sub leave_room {
    my ($self, $id, $room) = @_;
    @{$self->{connections}{$id}{rooms}} = grep { $_ ne $room } @{$self->{connections}{$id}{rooms}};
    @{$self->{rooms}{$room}} = grep { $_ != $id } @{$self->{rooms}{$room}//[]};
}

sub to_room {
    my ($self, $room, $event, $data, %opts) = @_;
    for my $id (@{$self->{rooms}{$room}//[]}) {
        next if $opts{except} && $id == $opts{except};
        $self->send_to($id, $event, $data);
    }
}

sub receive_message {
    my ($self, $from_id, $event, $data) = @_;
    my $socket = $self->_make_socket($from_id);
    if (my $h = $self->{handlers}{"message:$event"}) {
        $h->($socket, $data);
    } elsif (my $h2 = $self->{handlers}{message}) {
        $h2->($socket, $event, $data);
    }
}

sub _make_socket {
    my ($self, $id) = @_;
    return bless { server => $self, id => $id,
                   conn => $self->{connections}{$id} }, 'WebSocket::Socket';
}

sub messages_for {
    my ($self, $id) = @_;
    return @{$self->{connections}{$id}{messages}//[]};
}

sub connection_count { scalar keys %{$_[0]->{connections}} }
}

{
package WebSocket::Socket;

sub id     { $_[0]->{id} }
sub user   { $_[0]->{conn}{user} }
sub server { $_[0]->{server} }

sub emit {
    my ($self, $event, $data) = @_;
    $self->{server}->send_to($self->{id}, $event, $data);
}

sub broadcast {
    my ($self, $event, $data) = @_;
    $self->{server}->broadcast($event, $data, except => $self->{id});
}

sub join  { $_[0]->{server}->join_room($_[0]->{id}, $_[1]) }
sub leave { $_[0]->{server}->leave_room($_[0]->{id}, $_[1]) }

sub to_room {
    my ($self, $room, $event, $data) = @_;
    $self->{server}->to_room($room, $event, $data, except => $self->{id});
}
}

package main;

printf "=== WebSocket Chat Server ===\n\n";

my $ws = WebSocket::Server->new;

# Event handlers
$ws->on(connect => sub {
    my $socket = shift;
    printf "Connected: user=%s id=%d\n", $socket->user//"anon", $socket->id;
    $socket->emit("welcome", { message => "Welcome to the chat!" });
});

$ws->on(disconnect => sub {
    my $socket = shift;
    $socket->broadcast("user_left", { user => $socket->user });
    printf "Disconnected: user=%s\n", $socket->user//"anon";
});

$ws->on("message:join_room" => sub {
    my ($socket, $data) = @_;
    $socket->join($data->{room});
    $socket->to_room($data->{room}, "user_joined", { user => $socket->user, room => $data->{room} });
    printf "  %s joined room: %s\n", $socket->user, $data->{room};
});

$ws->on("message:chat" => sub {
    my ($socket, $data) = @_;
    printf "  [%s] %s: %s\n", $data->{room}//"global", $socket->user, $data->{text};
    $socket->to_room($data->{room}//"global", "chat", {
        user => $socket->user,
        text => $data->{text},
        ts   => time(),
    });
});

# Simulate connections
my $alice_id = $ws->connect(user => "alice");
my $bob_id   = $ws->connect(user => "bob");
my $carol_id = $ws->connect(user => "carol");

printf "\nConnections: %d\n\n", $ws->connection_count;

# Join rooms
$ws->receive_message($alice_id, "join_room", { room => "general" });
$ws->receive_message($bob_id,   "join_room", { room => "general" });
$ws->receive_message($carol_id, "join_room", { room => "perl" });
$ws->receive_message($alice_id, "join_room", { room => "perl" });

# Chat
printf "\nChat messages:\n";
$ws->receive_message($alice_id, "chat", { room => "general", text => "Hello everyone!" });
$ws->receive_message($bob_id,   "chat", { room => "general", text => "Hi Alice!" });
$ws->receive_message($carol_id, "chat", { room => "perl",    text => "Love Perl!" });
$ws->receive_message($alice_id, "chat", { room => "perl",    text => "Me too!" });

# Check messages received
printf "\nMessages for Bob:\n";
for my $msg ($ws->messages_for($bob_id)) {
    printf "  [%s] %s\n", $msg->{event}, ref($msg->{data}) ? "data={...}" : $msg->{data};
}

# Disconnect
printf "\n";
$ws->disconnect($alice_id);
printf "Connections after disconnect: %d\n", $ws->connection_count;
```

---

## Step 344: Mojo::UserAgent HTTP Client

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

# Mojo::UserAgent simulation
{
package Mojo::Transaction;

sub new {
    my ($class, %args) = @_;
    return bless { %args }, $class;
}

sub res { $_[0]->{response} }
sub req { $_[0]->{request} }
}

{
package Mojo::Response;

sub new {
    my ($class, %args) = @_;
    return bless {
        code    => $args{code}    // 200,
        message => $args{message} // "OK",
        headers => $args{headers} // {},
        body    => $args{body}    // "",
    }, $class;
}

sub code    { $_[0]->{code} }
sub message { $_[0]->{message} }
sub body    { $_[0]->{body} }
sub is_success { $_[0]->{code} >= 200 && $_[0]->{code} < 300 }
sub is_error   { $_[0]->{code} >= 400 }
sub header  { $_[0]->{headers}{$_[1]} }

sub json {
    my $self = shift;
    return eval { JSON::PP->new->utf8->decode($self->{body}) };
}

sub dom {
    # Simplified HTML parser
    my $self   = shift;
    my $body   = $self->{body};
    return bless { html => $body }, 'Mojo::DOM';
}
}

{
package Mojo::DOM;

sub find {
    my ($self, $selector) = @_;
    # Very simplified CSS selector (tag only)
    my @results;
    if ($selector =~ /^(\w+)$/) {
        my $tag = $1;
        while ($self->{html} =~ /<$tag[^>]*>(.*?)<\/$tag>/gs) {
            push @results, bless { text => $1 }, 'Mojo::DOM::Element';
        }
    }
    return \@results;
}

sub at {
    my ($self, $selector) = @_;
    my $results = $self->find($selector);
    return $results->[0];
}
}

{
package Mojo::DOM::Element;
sub text { my $t = $_[0]->{text}; $t =~ s/<[^>]+>//g; $t }
}

{
package Mojo::UserAgent;

my %mock_responses;

sub new { bless { timeout => 30, max_redirects => 5 }, $_[0] }
sub timeout       { $_[0]->{timeout} = $_[1]; $_[0] }
sub max_redirects { $_[0]->{max_redirects} = $_[1]; $_[0] }

# For testing: register mock responses
sub mock {
    my ($class, $url, $response) = @_;
    $mock_responses{$url} = $response;
}

sub get {
    my ($self, $url, %opts) = @_;
    return $self->_request("GET", $url, %opts);
}

sub post {
    my ($self, $url, %opts) = @_;
    return $self->_request("POST", $url, %opts);
}

sub put {
    my ($self, $url, %opts) = @_;
    return $self->_request("PUT", $url, %opts);
}

sub delete {
    my ($self, $url, %opts) = @_;
    return $self->_request("DELETE", $url, %opts);
}

sub _request {
    my ($self, $method, $url, %opts) = @_;
    
    my $mock = $mock_responses{$url} // $mock_responses{"$method $url"};
    
    unless ($mock) {
        return Mojo::Transaction->new(
            request  => { method => $method, url => $url },
            response => Mojo::Response->new(code => 404, body => "Not found (mock)"),
        );
    }
    
    my $response = ref($mock) eq 'CODE' ? $mock->($method, $url, %opts) : $mock;
    
    return Mojo::Transaction->new(
        request  => { method => $method, url => $url },
        response => Mojo::Response->new(%$response),
    );
}
}

package main;

printf "=== Mojo::UserAgent ===\n\n";

# Register mocks
Mojo::UserAgent->mock("https://api.example.com/users", {
    code => 200,
    body => '[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]',
    headers => { "Content-Type" => "application/json" },
});

Mojo::UserAgent->mock("https://api.example.com/users/1", {
    code => 200,
    body => '{"id":1,"name":"Alice","email":"alice@example.com"}',
});

Mojo::UserAgent->mock("https://example.com", {
    code => 200,
    body => '<html><body><h1>Hello</h1><p>World</p><p>Perl</p></body></html>',
});

Mojo::UserAgent->mock("https://api.example.com/broken", {
    code => 500,
    body => '{"error":"Server error"}',
});

my $ua = Mojo::UserAgent->new->timeout(10)->max_redirects(3);

# GET JSON
my $tx = $ua->get("https://api.example.com/users");
printf "GET users: %d %s\n", $tx->res->code, $tx->res->is_success ? "OK" : "FAIL";
my $users = $tx->res->json;
printf "  Got %d users\n", scalar @$users;
printf "  %s\n", $_->{name} for @$users;

# GET single
my $tx2 = $ua->get("https://api.example.com/users/1");
my $user = $tx2->res->json;
printf "\nGET user/1: %s (%s)\n", $user->{name}, $user->{email};

# HTML parsing
my $tx3 = $ua->get("https://example.com");
my $dom  = $tx3->res->dom;
my $h1   = $dom->at("h1");
printf "\nH1: %s\n", $h1->text if $h1;
my $paras = $dom->find("p");
printf "Paragraphs: %d\n", scalar @$paras;

# Error handling
my $tx4 = $ua->get("https://api.example.com/broken");
if ($tx4->res->is_error) {
    my $err = $tx4->res->json;
    printf "\nError response: %s (code=%d)\n", $err->{error}, $tx4->res->code;
}
```

---

## Step 345: Mojo::Validator

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Mojo::Validator;

sub new { bless { rules => [], errors => {} }, $_[0] }

sub input {
    my ($self, %data) = @_;
    $self->{data}   = \%data;
    $self->{errors} = {};
    return $self;
}

sub required {
    my ($self, $field, $msg) = @_;
    my $val = $self->{data}{$field};
    unless (defined $val && $val =~ /\S/) {
        $self->{errors}{$field} = $msg // "$field is required";
    }
    return $self;
}

sub optional {
    my ($self, $field) = @_;
    # Just marks field as optional — always passes
    return $self;
}

sub size {
    my ($self, $field, $min, $max) = @_;
    my $val = $self->{data}{$field} // "";
    my $len = length($val);
    if (defined $min && $len < $min) {
        $self->{errors}{$field} = "$field must be at least $min characters";
    } elsif (defined $max && $len > $max) {
        $self->{errors}{$field} = "$field must be at most $max characters";
    }
    return $self;
}

sub like {
    my ($self, $field, $regex, $msg) = @_;
    my $val = $self->{data}{$field} // "";
    unless ($val =~ $regex) {
        $self->{errors}{$field} = $msg // "$field is invalid";
    }
    return $self;
}

sub in {
    my ($self, $field, @values) = @_;
    my $val = $self->{data}{$field} // "";
    unless (grep { $_ eq $val } @values) {
        $self->{errors}{$field} = "$field must be one of: " . join(", ", @values);
    }
    return $self;
}

sub equal_to {
    my ($self, $field, $other) = @_;
    my $v1 = $self->{data}{$field}  // "";
    my $v2 = $self->{data}{$other}  // "";
    unless ($v1 eq $v2) {
        $self->{errors}{$field} = "$field must match $other";
    }
    return $self;
}

sub num {
    my ($self, $field) = @_;
    my $val = $self->{data}{$field} // "";
    unless ($val =~ /^-?\d+(\.\d+)?$/) {
        $self->{errors}{$field} = "$field must be a number";
    }
    return $self;
}

sub between {
    my ($self, $field, $min, $max) = @_;
    my $val = $self->{data}{$field} // 0;
    if ($val < $min || $val > $max) {
        $self->{errors}{$field} = "$field must be between $min and $max";
    }
    return $self;
}

sub custom {
    my ($self, $field, $code, $msg) = @_;
    unless ($code->($self->{data}{$field})) {
        $self->{errors}{$field} = $msg;
    }
    return $self;
}

sub has_error { exists $_[0]->{errors}{$_[1]} }
sub error     { $_[0]->{errors}{$_[1]} }
sub passed    { !%{$_[0]->{errors}} }
sub errors    { %{$_[0]->{errors}} }

sub output {
    my $self = shift;
    # Return validated/sanitized data
    return %{$self->{data}};
}
}

package main;

printf "=== Mojo::Validator ===\n\n";

my $v = Mojo::Validator->new;

# Registration form
sub validate_registration {
    my (%data) = @_;
    
    my $v = Mojo::Validator->new->input(%data)
        ->required("username")
        ->size("username", 3, 20)
        ->like("username", qr/^[a-zA-Z0-9_]+$/, "username can only contain letters, numbers and underscores")
        ->required("email")
        ->like("email", qr/^[^\s@]+@[^\s@]+\.[^\s@]+$/, "invalid email")
        ->required("password")
        ->size("password", 8, 100)
        ->required("password_confirm")
        ->equal_to("password_confirm", "password")
        ->optional("age")
        ->num("age")
        ->between("age", 13, 120)
        ->in("role", "user", "moderator", "admin");
    
    return $v;
}

# Bad data
my $bad = validate_registration(
    username         => "al",
    email            => "not-email",
    password         => "short",
    password_confirm => "different",
    age              => "abc",
    role             => "superuser",
);

printf "Registration (bad data):\n";
printf "  Passed: %s\n", $bad->passed ? "yes" : "no";
my %errs = $bad->errors;
printf "  %s: %s\n", $_, $errs{$_} for sort keys %errs;

# Good data
my $good = validate_registration(
    username         => "alice_99",
    email            => 'alice@example.com',
    password         => "SecurePass1!",
    password_confirm => "SecurePass1!",
    age              => "25",
    role             => "user",
);

printf "\nRegistration (good data):\n";
printf "  Passed: %s\n", $good->passed ? "yes" : "no";
```

---

## Step 346: Mojo::Promise (Async Pattern)

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Synchronous Promise simulation (Mojo::Promise pattern)
{
package Promise;

sub new {
    my ($class, $executor) = @_;
    my $self = bless {
        state    => "pending",
        value    => undef,
        reason   => undef,
        thens    => [],
        catches  => [],
    }, $class;
    
    if ($executor) {
        eval {
            $executor->(
                sub { $self->_resolve($_[0]) },  # resolve
                sub { $self->_reject($_[0])  },  # reject
            );
        };
        $self->_reject($@) if $@;
    }
    
    return $self;
}

sub resolve {
    my ($class, $val) = @_;
    return $class->new(sub { $_[0]->($val) });
}

sub reject {
    my ($class, $reason) = @_;
    return $class->new(sub { $_[1]->($reason) });
}

sub _resolve {
    my ($self, $value) = @_;
    return if $self->{state} ne "pending";
    $self->{state} = "fulfilled";
    $self->{value} = $value;
    $_->($value) for @{$self->{thens}};
}

sub _reject {
    my ($self, $reason) = @_;
    return if $self->{state} ne "pending";
    $self->{state} = "rejected";
    $self->{reason} = $reason;
    $_->($reason) for @{$self->{catches}};
}

sub then {
    my ($self, $on_fulfilled, $on_rejected) = @_;
    
    if ($self->{state} eq "fulfilled") {
        return Promise->new(sub {
            my ($resolve, $reject) = @_;
            eval { $resolve->($on_fulfilled->($self->{value})) };
            $reject->($@) if $@;
        });
    }
    
    if ($self->{state} eq "rejected") {
        return $on_rejected
            ? Promise->new(sub { $_[0]->($on_rejected->($self->{reason})) })
            : $self;
    }
    
    # Pending
    my $next = Promise->new;
    push @{$self->{thens}}, sub {
        my $val = eval { $on_fulfilled->($_[0]) };
        $@ ? $next->_reject($@) : $next->_resolve($val);
    };
    push @{$self->{catches}}, sub {
        $on_rejected ? $next->_resolve($on_rejected->($_[0])) : $next->_reject($_[0]);
    } if $on_rejected;
    
    return $next;
}

sub catch {
    my ($self, $handler) = @_;
    return $self->then(undef, $handler);
}

sub finally {
    my ($self, $handler) = @_;
    return $self->then(
        sub { $handler->(); $_[0] },
        sub { $handler->(); Promise->reject($_[0]) },
    );
}

sub all {
    my ($class, @promises) = @_;
    return Promise->new(sub {
        my ($resolve, $reject) = @_;
        my @results;
        my $pending = scalar @promises;
        
        for my $i (0..$#promises) {
            $promises[$i]->then(sub {
                $results[$i] = $_[0];
                $resolve->(\@results) if --$pending == 0;
            }, $reject);
        }
    });
}
}

package main;

printf "=== Promise Pattern (Async) ===\n\n";

# Simulate async operations
sub fetch_user {
    my $id = shift;
    return Promise->new(sub {
        my ($resolve, $reject) = @_;
        if ($id > 0) {
            $resolve->({ id => $id, name => "User $id", email => "user$id\@example.com" });
        } else {
            $reject->("Invalid user id");
        }
    });
}

sub fetch_posts {
    my $user_id = shift;
    return Promise->new(sub {
        my ($resolve, $reject) = @_;
        $resolve->([
            { title => "Post 1 by user $user_id" },
            { title => "Post 2 by user $user_id" },
        ]);
    });
}

# Chain promises
fetch_user(42)
    ->then(sub {
        my $user = shift;
        printf "Got user: %s (%s)\n", $user->{name}, $user->{email};
        return fetch_posts($user->{id});
    })
    ->then(sub {
        my $posts = shift;
        printf "Got %d posts\n", scalar @$posts;
        printf "  %s\n", $_->{title} for @$posts;
    })
    ->catch(sub {
        printf "Error: %s\n", shift;
    });

# Rejection
printf "\n";
fetch_user(-1)
    ->then(sub { printf "This should not run\n" })
    ->catch(sub { printf "Caught error: %s\n", shift });

# Promise.all
printf "\n";
Promise->all(
    fetch_user(1),
    fetch_user(2),
    fetch_user(3),
)->then(sub {
    my $users = shift;
    printf "All resolved: %d users\n", scalar @$users;
    printf "  %s\n", $_->{name} for @$users;
});
```

---

## Step 347: Mojo::IOLoop Event Loop

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package EventLoop;

sub new {
    return bless {
        timers    => [],
        intervals => [],
        running   => 0,
        tick      => 0,
    }, $_[0];
}

sub timer {
    my ($self, $delay, $code) = @_;
    my $id = push @{$self->{timers}}, {
        fire_at => $self->{tick} + $delay,
        code    => $code,
        fired   => 0,
    };
    return "timer_$id";
}

sub recurring {
    my ($self, $interval, $code) = @_;
    my $id = push @{$self->{intervals}}, {
        interval => $interval,
        next_at  => $self->{tick} + $interval,
        code     => $code,
    };
    return "interval_$id";
}

sub remove {
    my ($self, $id) = @_;
    if ($id =~ /^timer_(\d+)$/)    { delete $self->{timers}[$1-1] }
    if ($id =~ /^interval_(\d+)$/) { delete $self->{intervals}[$1-1] }
}

sub tick {
    my ($self, $steps) = @_;
    $steps //= 1;
    
    for (1..$steps) {
        $self->{tick}++;
        
        # Fire one-shot timers
        for my $t (@{$self->{timers}}) {
            next unless $t && !$t->{fired};
            if ($self->{tick} >= $t->{fire_at}) {
                $t->{fired} = 1;
                $t->{code}->();
            }
        }
        
        # Fire recurring intervals
        for my $i (@{$self->{intervals}}) {
            next unless $i;
            if ($self->{tick} >= $i->{next_at}) {
                $i->{next_at} = $self->{tick} + $i->{interval};
                $i->{code}->();
            }
        }
    }
}

sub run_until {
    my ($self, $max_ticks) = @_;
    $max_ticks //= 100;
    while ($self->{tick} < $max_ticks) {
        $self->tick(1);
        last unless @{$self->{timers}} || @{$self->{intervals}};
    }
}
}

package main;

printf "=== Event Loop ===\n\n";

my $loop = EventLoop->new;
my @log;

# One-shot timers
$loop->timer(2,  sub { push @log, "Timer 2s fired" });
$loop->timer(5,  sub { push @log, "Timer 5s fired" });
$loop->timer(10, sub { push @log, "Timer 10s fired" });

# Recurring intervals
my $count = 0;
my $interval_id = $loop->recurring(3, sub {
    $count++;
    push @log, "Interval 3s: count=$count";
});

# Stop interval after 3 fires
$loop->timer(10, sub {
    push @log, "Stopping interval";
});

# Tick through 15 steps
for my $tick (1..15) {
    $loop->tick(1);
    if (@log) {
        printf "t=%2d: %s\n", $tick, $_ for @log;
        @log = ();
    }
}

printf "\nTotal interval fires: %d\n", $count;
```

---

## Step 348: Database Plugin Pattern

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

{
package Mojolicious::Plugin::Database;

sub register {
    my ($class, $app, %config) = @_;
    
    my $dsn  = $config{dsn}      // "dbi:SQLite::memory:";
    my $user = $config{user}     // "";
    my $pass = $config{password} // "";
    my %opts = (RaiseError => 1, AutoCommit => 1, sqlite_unicode => 1,
                %{$config{options}//{}});
    
    my $dbh  = DBI->connect($dsn, $user, $pass, \%opts);
    
    # Add helper to app
    $app->helper(db => sub { $dbh });
    
    # Run migrations if provided
    if (my $sql = $config{migrations}) {
        for my $stmt (split /;\s*\n/, $sql) {
            $stmt =~ s/^\s+|\s+$//g;
            $dbh->do($stmt) if $stmt;
        }
    }
    
    printf "  DB plugin registered (DSN: %s)\n", $dsn;
    return $dbh;
}
}

package main;

printf "=== Database Plugin ===\n\n";

# Simulate plugin registration
my $app = Mojolicious->new;

my $dbh = Mojolicious::Plugin::Database->register($app,
    dsn        => "dbi:SQLite::memory:",
    migrations => q{
        CREATE TABLE IF NOT EXISTS notes (
            id         INTEGER PRIMARY KEY AUTOINCREMENT,
            title      TEXT NOT NULL,
            body       TEXT,
            created_at INTEGER DEFAULT (strftime('%s','now'))
        )
    }
);

# Use the db helper
$app->helper(create_note => sub {
    my ($ctx, $title, $body) = @_;
    $dbh->do("INSERT INTO notes (title, body) VALUES (?,?)", undef, $title, $body);
    return $dbh->last_insert_id;
});

$app->helper(list_notes => sub {
    return $dbh->selectall_arrayref("SELECT * FROM notes ORDER BY created_at", {Slice=>{}});
});

# Routes
$app->get("/notes" => sub {
    my $c = shift;
    my $notes = $dbh->selectall_arrayref("SELECT * FROM notes", {Slice=>{}});
    $c->render(json => $notes);
});

$app->post("/notes" => sub {
    my $c     = shift;
    my $title = $c->param("title") // "Untitled";
    my $body  = $c->param("body")  // "";
    $dbh->do("INSERT INTO notes (title, body) VALUES (?,?)", undef, $title, $body);
    $c->render(json => { id => $dbh->last_insert_id, created => 1 }, status => 201);
});

# Test
for my $note (["Learn Perl", "Start with basics"], ["Build Web App", "Use Mojolicious"],
              ["Deploy", "Use Docker or CGI"]) {
    $dbh->do("INSERT INTO notes (title, body) VALUES (?,?)", undef, @$note);
}

my $notes = $dbh->selectall_arrayref("SELECT * FROM notes", {Slice=>{}});
printf "Notes in DB: %d\n", scalar @$notes;
printf "  [%d] %s\n", $_->{id}, $_->{title} for @$notes;

# Test routes
printf "\nRoute tests:\n";
my $res1 = $app->dispatch("GET",  "/notes");
printf "GET /notes: %d\n", $res1->{status};

my $res2 = $app->dispatch("POST", "/notes", params => { title => "New Note", body => "Content" });
printf "POST /notes: %d\n", $res2->{status};
```

---

## Step 349: Content Negotiation

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

{
package ContentNeg;

my %formatters = (
    "application/json" => sub { JSON::PP->new->utf8->encode($_[0]) },
    "text/plain"       => sub { ref($_[0]) ? _to_text($_[0]) : $_[0] },
    "text/html"        => sub { ref($_[0]) ? _to_html($_[0]) : "<p>$_[0]</p>" },
    "text/csv"         => sub { ref($_[0]) eq 'ARRAY' ? _to_csv($_[0]) : "$_[0]\n" },
);

sub negotiate {
    my ($class, $accept, $data) = @_;
    
    # Parse Accept header: "text/html,application/xhtml+xml,application/json;q=0.9,*/*;q=0.8"
    my @types = sort { ($b->{q}//1) <=> ($a->{q}//1) }
        map {
            my ($type, @params) = split /;/, $_;
            $type =~ s/^\s+|\s+$//g;
            my $q = 1;
            for my $p (@params) {
                $q = $1 if $p =~ /q=([\d.]+)/;
            }
            { type => $type, q => $q }
        } split /,/, ($accept//"*/*");
    
    # Find best match
    for my $t (@types) {
        my $type = $t->{type};
        if ($type eq "*/*") {
            return ("application/json", $formatters{"application/json"}->($data));
        }
        if ($formatters{$type}) {
            return ($type, $formatters{$type}->($data));
        }
    }
    
    return ("application/json", $formatters{"application/json"}->($data));
}

sub _to_text {
    my $data = shift;
    return ref($data) eq 'ARRAY'
        ? join("\n", map { ref($_) eq 'HASH' ? join(" | ", map {"$_: $_->{$_}"} keys %$_) : $_ } @$data)
        : join("\n", map { "$_: $data->{$_}" } sort keys %$data);
}

sub _to_html {
    my $data = shift;
    if (ref($data) eq 'ARRAY') {
        my $html = "<table border='1'>";
        if (@$data && ref($data->[0]) eq 'HASH') {
            my @cols = sort keys %{$data->[0]};
            $html .= "<tr>" . join("", map { "<th>$_</th>" } @cols) . "</tr>";
            for my $row (@$data) {
                $html .= "<tr>" . join("", map { "<td>" . ($row->{$_}//"") . "</td>" } @cols) . "</tr>";
            }
        }
        $html .= "</table>";
        return $html;
    }
    return "<dl>" . join("", map { "<dt>$_</dt><dd>$data->{$_}</dd>" } sort keys %$data) . "</dl>";
}

sub _to_csv {
    my $rows = shift;
    return "" unless @$rows;
    my @cols = sort keys %{$rows->[0]};
    my $csv  = join(",", @cols) . "\n";
    for my $row (@$rows) {
        $csv .= join(",", map { my $v = $row->{$_}//""; $v =~ s/"/""/g; "\"$v\"" } @cols) . "\n";
    }
    return $csv;
}
}

package main;

printf "=== Content Negotiation ===\n\n";

my $data = [
    { id => 1, name => "Alice", role => "admin" },
    { id => 2, name => "Bob",   role => "user"  },
];

my @accept_tests = (
    "application/json",
    "text/html",
    "text/plain",
    "text/csv",
    "text/html,application/json;q=0.9,*/*;q=0.8",
    "application/xml,text/plain;q=0.7",
);

for my $accept (@accept_tests) {
    my ($type, $body) = ContentNeg->negotiate($accept, $data);
    printf "Accept: %-50s => %s\n", $accept, $type;
    printf "  %.80s\n\n", $body =~ s/\n/ /gr;
}
```

---

## Step 350: Capstone — Mojolicious Mini Application

```perl
#!/usr/bin/perl
# mojo_app.pl — Complete Mojolicious-style application
use strict;
use warnings;
use DBI;
use JSON::PP;

# Initialize
my $dbh = DBI->connect("dbi:SQLite::memory:", "", "", {
    RaiseError => 1, AutoCommit => 1, sqlite_unicode => 1 });

for my $sql (split /;/, q{
CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT UNIQUE, email TEXT, role TEXT DEFAULT 'user');
CREATE TABLE bookmarks (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, url TEXT NOT NULL,
    title TEXT, tags TEXT, created_at INTEGER DEFAULT (strftime('%s','now')));
}) {
    my $s = $sql; $s =~ s/^\s+|\s+$//g; $dbh->do($s) if $s;
}

# Seed data
$dbh->do("INSERT INTO users (username, email, role) VALUES (?,?,?)", undef, "alice", "alice\@test.com", "admin");
$dbh->do("INSERT INTO users (username, email, role) VALUES (?,?,?)", undef, "bob",   "bob\@test.com",   "user");
for my $bm (
    [1, "https://perl.org",          "Perl Language",   "perl,language"],
    [1, "https://metacpan.org",      "MetaCPAN",        "perl,cpan"],
    [2, "https://perlmonks.org",     "PerlMonks",       "perl,community"],
    [1, "https://mojolicious.org",   "Mojolicious",     "perl,web,framework"],
    [2, "https://dancer.pm",         "Dancer",          "perl,web,framework"],
) {
    $dbh->do("INSERT INTO bookmarks (user_id,url,title,tags) VALUES (?,?,?,?)", undef, @$bm);
}

my $app  = Mojolicious->new(config => { name => "Bookmarks App" });
my $json = JSON::PP->new->utf8->canonical;

# Helpers
$app->helper(db => sub { $dbh });

# Hooks
$app->hook(before_dispatch => sub {
    my $c = shift;
    $c->{request}{user} = $dbh->selectrow_hashref(
        "SELECT * FROM users WHERE id=?", undef, $c->{request}{user_id}//1);
});

# Routes
$app->get("/api/bookmarks" => sub {
    my $c    = shift;
    my $user = $c->{request}{user};
    my $tag  = $c->param("tag");
    
    my ($sql, @params) = ("SELECT b.*, u.username FROM bookmarks b JOIN users u ON b.user_id=u.id", );
    if ($tag) {
        $sql .= " WHERE b.tags LIKE ?";
        push @params, "%$tag%";
    }
    $sql .= " ORDER BY b.created_at DESC";
    
    my $items = $dbh->selectall_arrayref($sql, {Slice=>{}}, @params);
    $c->render(json => { items => $items, count => scalar @$items });
});

$app->get("/api/bookmarks/:id" => sub {
    my $c  = shift;
    my $id = $c->{request}{captures}{id};
    my $bm = $dbh->selectrow_hashref("SELECT * FROM bookmarks WHERE id=?", undef, $id);
    $bm ? $c->render(json => $bm) : $c->render(json => { error => "Not found" }, status => 404);
});

$app->post("/api/bookmarks" => sub {
    my $c    = shift;
    my $user = $c->{request}{user};
    my $url  = $c->param("url")   or do { $c->render(json => {error=>"url required"}, status=>400); return };
    my $title= $c->param("title") // $url;
    my $tags = $c->param("tags")  // "";
    
    $dbh->do("INSERT INTO bookmarks (user_id,url,title,tags) VALUES (?,?,?,?)",
        undef, $user->{id}, $url, $title, $tags);
    my $bm = $dbh->selectrow_hashref("SELECT * FROM bookmarks WHERE id=?", undef, $dbh->last_insert_id);
    $c->render(json => $bm, status => 201);
});

$app->delete("/api/bookmarks/:id" => sub {
    my $c  = shift;
    my $id = $c->{request}{captures}{id};
    $dbh->do("DELETE FROM bookmarks WHERE id=?", undef, $id);
    $c->render(json => { deleted => \1, id => $id+0 });
});

$app->get("/api/stats" => sub {
    my $c = shift;
    $c->render(json => {
        total_bookmarks => $dbh->selectrow_array("SELECT COUNT(*) FROM bookmarks")+0,
        total_users     => $dbh->selectrow_array("SELECT COUNT(*) FROM users")+0,
        top_tags        => do {
            my @bms = @{$dbh->selectall_arrayref("SELECT tags FROM bookmarks WHERE tags != ''", {Slice=>{}})};
            my %tc;
            for my $b (@bms) { $tc{$_}++ for split /,/, $b->{tags} }
            [sort { $tc{$b} <=> $tc{$a} } keys %tc][0..2];
        },
    });
});

# Tests
printf "=== Bookmarks App (Mojolicious) ===\n\n";

my @tests = (
    ["GET",    "/api/bookmarks",          { params => {} }],
    ["GET",    "/api/bookmarks",          { params => { tag => "web" } }],
    ["GET",    "/api/bookmarks/1",        {}],
    ["GET",    "/api/bookmarks/999",      {}],
    ["POST",   "/api/bookmarks",          { params => { url => "https://new.example.com", title => "New", tags => "new,test" } }],
    ["DELETE", "/api/bookmarks/1",        {}],
    ["GET",    "/api/stats",              {}],
);

for my $t (@tests) {
    my ($method, $path, $opts) = @$t;
    my $res  = $app->dispatch($method, $path, %$opts, user_id => 1);
    my $data = eval { $json->decode($res->{body}) };
    
    printf "%s %-30s %d: ", $method, $path, $res->{status};
    if (ref $data eq 'HASH' && $data->{items}) {
        printf "count=%d\n", $data->{count};
    } elsif (ref $data eq 'HASH' && $data->{error}) {
        printf "error=%s\n", $data->{error};
    } elsif (ref $data eq 'HASH' && $data->{deleted}) {
        printf "deleted=yes\n";
    } elsif (ref $data eq 'HASH' && $data->{total_bookmarks}) {
        printf "total=%d users=%d\n", $data->{total_bookmarks}, $data->{total_users};
    } elsif (ref $data eq 'HASH' && $data->{url}) {
        printf "title=%s\n", $data->{title};
    } else {
        printf "%s\n", substr($res->{body},0,50);
    }
}
```

---

## สรุป Part 35 — Mojolicious Web Framework

ใน Part นี้คุณได้เรียนรู้:

### Mojolicious Concepts
- **App Architecture** — Routes, hooks, helpers, dispatch
- **Mojo::Template** — EP template syntax with `<%= %>` tags
- **WebSocket** — Event-based real-time communication simulation
- **Mojo::UserAgent** — HTTP client with mock support
- **Mojo::Validator** — Fluent input validation chain
- **Mojo::Promise** — Async programming pattern (then/catch/all)
- **EventLoop** — Timer and recurring interval management
- **Database Plugin** — Plugin registration pattern
- **Content Negotiation** — JSON/HTML/CSV/text based on Accept header
- **Capstone** — Bookmark manager REST API

**ถัดไป: [Part 36 — Template Toolkit](part_36.md)**
