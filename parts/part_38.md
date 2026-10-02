# Part 38: REST API Design Patterns
## Steps 371-380: Building Production-Grade REST APIs in Perl

---

## Step 371: REST Fundamentals

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

# REST API Design Principles
{
package REST::Response;

my %status_texts = (
    200 => "OK",           201 => "Created",      204 => "No Content",
    301 => "Moved Permanently", 302 => "Found",    304 => "Not Modified",
    400 => "Bad Request",  401 => "Unauthorized",  403 => "Forbidden",
    404 => "Not Found",    405 => "Method Not Allowed", 409 => "Conflict",
    410 => "Gone",         422 => "Unprocessable Entity",
    429 => "Too Many Requests", 500 => "Internal Server Error",
    501 => "Not Implemented",   503 => "Service Unavailable",
);

sub new {
    my ($class, %args) = @_;
    return bless {
        status  => $args{status}  // 200,
        data    => $args{data},
        meta    => $args{meta}    // {},
        errors  => $args{errors}  // [],
        headers => $args{headers} // {},
    }, $class;
}

# Factory methods
sub ok       { shift->new(status=>200, data=>$_[0]) }
sub created  { shift->new(status=>201, data=>$_[0]) }
sub no_content { bless {status=>204}, $_[0] }
sub bad_request  { shift->new(status=>400, errors=>[{ code=>"BAD_REQUEST",  message=>$_[0] }]) }
sub unauthorized { shift->new(status=>401, errors=>[{ code=>"UNAUTHORIZED", message=>$_[0]//"Unauthorized" }]) }
sub forbidden    { shift->new(status=>403, errors=>[{ code=>"FORBIDDEN",    message=>$_[0]//"Forbidden" }]) }
sub not_found    { shift->new(status=>404, errors=>[{ code=>"NOT_FOUND",    message=>$_[0]//"Not found" }]) }
sub conflict     { shift->new(status=>409, errors=>[{ code=>"CONFLICT",     message=>$_[0] }]) }
sub unprocessable{ shift->new(status=>422, errors=>$_[0]) }
sub server_error { shift->new(status=>500, errors=>[{ code=>"SERVER_ERROR", message=>$_[0]//"Internal error" }]) }

sub with_meta {
    my ($self, %meta) = @_;
    $self->{meta} = { %{$self->{meta}}, %meta };
    return $self;
}

sub with_header {
    my ($self, $name, $value) = @_;
    $self->{headers}{$name} = $value;
    return $self;
}

sub paginate {
    my ($self, %p) = @_;
    return $self->with_meta(
        pagination => {
            page        => $p{page}  // 1,
            per_page    => $p{per_page} // 20,
            total       => $p{total} // 0,
            total_pages => int(($p{total}+($p{per_page}//20)-1) / ($p{per_page}//20)),
            has_next    => ($p{page}//1) * ($p{per_page}//20) < ($p{total}//0),
            has_prev    => ($p{page}//1) > 1,
        }
    );
}

sub to_json {
    my $self = shift;
    my $body = {};
    
    if (@{$self->{errors}//[]}) {
        $body->{success} = JSON::PP::false();
        $body->{errors}  = $self->{errors};
    } else {
        $body->{success} = JSON::PP::true();
        $body->{data}    = $self->{data} if defined $self->{data};
        $body->{meta}    = $self->{meta} if %{$self->{meta}//{}};
    }
    
    return JSON::PP->new->utf8->canonical->encode($body);
}

sub status_text { $status_texts{$_[0]->{status}} // "Unknown" }
sub is_success  { $_[0]->{status} >= 200 && $_[0]->{status} < 300 }
sub is_error    { $_[0]->{status} >= 400 }
sub status      { $_[0]->{status} }
}

package main;

printf "=== REST Response Objects ===\n\n";

my $json = JSON::PP->new->utf8->canonical->pretty;

# OK response
my $res1 = REST::Response->ok({ id => 1, name => "Alice", email => 'alice@example.com' });
printf "200 OK:\n%s\n", $res1->to_json;

# Created
my $res2 = REST::Response->created({ id => 42, title => "New Post" })
    ->with_header("Location", "/api/posts/42");
printf "201 Created:\n%s\n", $res2->to_json;

# Not Found
my $res3 = REST::Response->not_found("User with id 999 not found");
printf "404 Not Found:\n%s\n", $res3->to_json;

# Unprocessable
my $res4 = REST::Response->unprocessable([
    { field => "email",    code => "INVALID_FORMAT", message => "Invalid email format" },
    { field => "username", code => "TOO_SHORT",       message => "Username must be at least 3 characters" },
]);
printf "422 Unprocessable:\n%s\n", $res4->to_json;

# Paginated list
my $res5 = REST::Response->ok([
    { id => 1, title => "Post 1" },
    { id => 2, title => "Post 2" },
])->paginate(page => 1, per_page => 10, total => 45);
printf "200 Paginated:\n%s\n", $res5->to_json;
```

---

## Step 372: Resource Router

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

{
package REST::Router;

sub new {
    return bless { routes => [], middleware => [], prefix => "" }, $_[0];
}

sub prefix {
    my ($self, $p) = @_;
    my $new = bless { %$self, prefix => $p, routes => [@{$self->{routes}}] }, ref $self;
    return $new;
}

sub use_middleware {
    my ($self, $mw) = @_;
    push @{$self->{middleware}}, $mw;
    return $self;
}

sub _add {
    my ($self, $method, $path, $handler, %opts) = @_;
    my $full = $self->{prefix} . $path;
    # Convert :param and *glob to named captures
    my $regex = $full;
    $regex =~ s{:(\w+)}{(?<$1>[^/]+)}g;
    $regex =~ s{\*(\w+)}{(?<$1>.+)}g;
    push @{$self->{routes}}, {
        method  => uc $method,
        path    => $full,
        regex   => qr{^$regex$},
        handler => $handler,
        name    => $opts{name},
        auth    => $opts{auth} // 0,
    };
    return $self;
}

sub get    { $_[0]->_add("GET",    @_[1..$#_]) }
sub post   { $_[0]->_add("POST",   @_[1..$#_]) }
sub put    { $_[0]->_add("PUT",    @_[1..$#_]) }
sub patch  { $_[0]->_add("PATCH",  @_[1..$#_]) }
sub delete { $_[0]->_add("DELETE", @_[1..$#_]) }

sub resource {
    my ($self, $name, $controller, %opts) = @_;
    my $base = "/$name";
    $self->get("$base",             sub { $controller->index(@_)   }, name => "${name}.index");
    $self->post("$base",            sub { $controller->create(@_)  }, name => "${name}.create", auth => $opts{auth});
    $self->get("$base/:id",         sub { $controller->show(@_)    }, name => "${name}.show");
    $self->put("$base/:id",         sub { $controller->update(@_)  }, name => "${name}.update", auth => $opts{auth});
    $self->patch("$base/:id",       sub { $controller->patch(@_)   }, name => "${name}.patch",  auth => $opts{auth});
    $self->delete("$base/:id",      sub { $controller->destroy(@_) }, name => "${name}.destroy", auth => $opts{auth});
    return $self;
}

sub dispatch {
    my ($self, $method, $path, %req) = @_;
    $method = uc $method;
    $req{method} = $method;
    $req{path}   = $path;
    $req{params} //= {};
    $req{body}   //= {};
    
    my ($matched, $ctx) = (0, {%req});
    
    for my $route (@{$self->{routes}}) {
        next unless $route->{method} eq $method;
        next unless $path =~ $route->{regex};
        
        $ctx->{captures} = {%+};
        $matched = 1;
        
        # Run middleware
        my $proceed = 1;
        for my $mw (@{$self->{middleware}}) {
            unless ($mw->($ctx)) { $proceed = 0; last }
        }
        
        next unless $proceed;
        
        my $result = eval { $route->{handler}->($ctx) };
        return $@ ? { status => 500, body => "{\"error\":\"$@\"}" } : $result;
    }
    
    # Method not allowed?
    my @allowed = map { $_->{method} } grep { $path =~ $_->{regex} } @{$self->{routes}};
    if (@allowed) {
        return { status => 405, headers => { Allow => join(",", @allowed) },
                 body => "{\"error\":\"Method not allowed\"}" };
    }
    
    return { status => 404, body => "{\"error\":\"Not found: $method $path\"}" };
}

sub path_for {
    my ($self, $name, %params) = @_;
    my ($route) = grep { ($_->{name}//"") eq $name } @{$self->{routes}};
    return undef unless $route;
    my $path = $route->{path};
    $path =~ s{:(\w+)}{$params{$1} // ":$1"}ge;
    return $path;
}

sub list_routes {
    my $self = shift;
    for my $r (@{$self->{routes}}) {
        printf "  %-8s %-35s %s\n", $r->{method}, $r->{path}, $r->{name}//"";
    }
}
}

package main;

printf "=== REST Router ===\n\n";

my $json = JSON::PP->new->utf8;

# Mock controllers
{
package UserController;
my @users = ({id=>1,name=>"Alice"},{id=>2,name=>"Bob"});

sub index   { my $c=shift; { status=>200, body=>$json->encode(\@users) } }
sub show    { my $c=shift; my $id=$c->{captures}{id}; my ($u)=grep{$_->{id}==$id}@users; $u ? {status=>200,body=>$json->encode($u)} : {status=>404,body=>"{\"error\":\"not found\"}"} }
sub create  { my $c=shift; {status=>201,body=>"{\"id\":3,\"name\":\"New\"}"} }
sub update  { my $c=shift; {status=>200,body=>"{\"updated\":true}"} }
sub patch   { my $c=shift; {status=>200,body=>"{\"patched\":true}"} }
sub destroy { my $c=shift; {status=>204,body=>""} }
}

my $router = REST::Router->new;

# Auth middleware
$router->use_middleware(sub {
    my $ctx = shift;
    # Skip auth for GET
    return 1 if $ctx->{method} eq "GET";
    return 1 if $ctx->{auth_token};  # Has token
    return 1;  # Allow all for demo
});

# Register resource
$router->resource("users", "UserController", auth => 1);

# Additional routes
$router->get("/api/health" => sub { { status=>200, body=>"{\"ok\":true}" } });
$router->get("/api/users/:id/posts" => sub {
    my $c = shift;
    { status=>200, body=>"{\"user_id\":$c->{captures}{id},\"posts\":[]}" }
});

printf "Registered routes:\n";
$router->list_routes;

printf "\nDispatching:\n";
for my $test (
    ["GET",    "/users",     {}],
    ["GET",    "/users/1",   {}],
    ["GET",    "/users/999", {}],
    ["POST",   "/users",     {}],
    ["PUT",    "/users/1",   {}],
    ["DELETE", "/users/1",   {}],
    ["GET",    "/api/health",{}],
    ["PATCH",  "/missing",   {}],
) {
    my ($m,$p,%o) = @$test;
    my $res = $router->dispatch($m, $p, %o);
    printf "  %s %-25s => %d\n", $m, $p, $res->{status};
}

# Path generation
printf "\nPath helpers:\n";
printf "  users.show(id=42): %s\n", $router->path_for("users.show", id=>42) // "N/A";
printf "  users.index: %s\n",       $router->path_for("users.index") // "N/A";
```

---

## Step 373: Request/Response Handling

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

{
package REST::Request;

sub new {
    my ($class, %args) = @_;
    return bless {
        method  => uc($args{method} // "GET"),
        path    => $args{path}    // "/",
        headers => $args{headers} // {},
        query   => $args{query}   // {},
        body    => $args{body}    // {},
        params  => {},
        user    => undef,
    }, $class;
}

sub method     { $_[0]->{method} }
sub path       { $_[0]->{path} }
sub header     { $_[0]->{headers}{$_[1]} }
sub query      { my $s=shift; @_ ? $s->{query}{$_[0]} : $s->{query} }
sub param      { my $s=shift; @_ ? ($s->{params}{$_[0]}//=$s->{query}{$_[0]}//$s->{body}{$_[0]}) : {%{$s->{params}},%{$s->{query}},%{$s->{body}}} }
sub body       { my $s=shift; @_ ? $s->{body}{$_[0]} : $s->{body} }
sub json_body  { my $s=shift; my $ct=$s->content_type; return $s->{body} if ref $s->{body}; eval{JSON::PP->new->decode($s->{body}//"{}")} }
sub user       { $_[0]->{user} }
sub set_user   { $_[0]->{user} = $_[1]; $_[0] }
sub set_param  { $_[0]->{params}{$_[1]} = $_[2]; $_[0] }
sub is_json    { ($_[0]->{headers}{"Content-Type"}//"") =~ /application\/json/i }
sub is_ajax    { ($_[0]->{headers}{"X-Requested-With"}//"") eq "XMLHttpRequest" }
sub bearer_token {
    my $auth = $_[0]->{headers}{Authorization} // "";
    $auth =~ /^Bearer\s+(.+)/ ? $1 : undef
}
sub content_type { $_[0]->{headers}{"Content-Type"} // "application/json" }
sub accept       { $_[0]->{headers}{Accept} // "*/*" }
sub ip           { $_[0]->{headers}{"X-Real-IP"} // $_[0]->{headers}{"X-Forwarded-For"} // "127.0.0.1" }

sub validate {
    my ($self, %rules) = @_;
    my @errors;
    my $data = { %{$self->body}, %{$self->query} };
    
    for my $field (keys %rules) {
        my @field_rules = ref $rules{$field} ? @{$rules{$field}} : ($rules{$field});
        my $val = $data->{$field};
        
        for my $rule (@field_rules) {
            if ($rule eq "required" && !defined $val) {
                push @errors, { field => $field, code => "REQUIRED", message => "$field is required" };
            } elsif (ref $rule eq 'CODE') {
                my $err = $rule->($val);
                push @errors, { field => $field, code => "INVALID", message => $err } if $err;
            } elsif ($rule =~ /^min:(\d+)$/ && defined $val && length($val) < $1) {
                push @errors, { field => $field, code => "TOO_SHORT", message => "min length $1" };
            } elsif ($rule =~ /^max:(\d+)$/ && defined $val && length($val) > $1) {
                push @errors, { field => $field, code => "TOO_LONG", message => "max length $1" };
            } elsif ($rule eq "email" && defined $val && $val !~ /^[^\s@]+@[^\s@]+\.[^\s@]+$/) {
                push @errors, { field => $field, code => "INVALID_EMAIL", message => "Invalid email" };
            } elsif ($rule eq "numeric" && defined $val && $val !~ /^-?\d+(\.\d+)?$/) {
                push @errors, { field => $field, code => "NOT_NUMERIC", message => "Must be numeric" };
            }
        }
    }
    
    return @errors;
}
}

package main;

printf "=== REST Request Handling ===\n\n";

# Simulate requests
my $req1 = REST::Request->new(
    method  => "POST",
    path    => "/api/users",
    headers => { "Content-Type" => "application/json", "Authorization" => "Bearer mytoken123" },
    body    => { username => "alice", email => 'alice@example.com', age => 25 },
);

printf "Request 1:\n";
printf "  method: %s  path: %s\n", $req1->method, $req1->path;
printf "  token: %s\n", $req1->bearer_token // "none";
printf "  is_json: %s\n", $req1->is_json ? "yes" : "no";
printf "  body.username: %s\n", $req1->body("username");

# Validate
my @errors = $req1->validate(
    username => ["required", "min:3", "max:20"],
    email    => ["required", "email"],
    age      => ["required", "numeric", sub { $_[0]<13 ? "Must be 13+" : undef }],
);
printf "  validation: %s\n", @errors ? "failed: ".join(", ", map{"$_->{field}: $_->{message}"}@errors) : "passed";

# Bad request
my $req2 = REST::Request->new(
    method => "POST",
    path   => "/api/users",
    body   => { username => "x", email => "bad-email", age => 5 },
);

my @err2 = $req2->validate(
    username => ["required","min:3"],
    email    => ["required","email"],
    age      => ["required","numeric", sub { ($_[0]//0)<13 ? "Must be 13+" : undef }],
);
printf "\nRequest 2 validation errors:\n";
printf "  %s: %s\n", $_->{field}, $_->{message} for @err2;

# Query params
my $req3 = REST::Request->new(
    method => "GET",
    path   => "/api/products",
    query  => { page => 2, per_page => 20, q => "laptop", sort => "price", order => "asc" },
);
printf "\nQuery params: %s\n", join(", ", map {"$_=$req3->query($_)"} qw(page per_page q sort order));
my $qp = $req3->query;
printf "  page=%s per_page=%s q=%s\n", $qp->{page}, $qp->{per_page}, $qp->{q};
```

---

## Step 374: API Versioning

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

{
package API::Versioner;

sub new {
    my ($class) = @_;
    return bless { versions => {}, default => "v1" }, $class;
}

sub version {
    my ($self, $ver, $router) = @_;
    $self->{versions}{$ver} = $router;
    return $self;
}

sub default_version {
    my ($self, $ver) = @_;
    $self->{default} = $ver;
    return $self;
}

sub detect_version {
    my ($self, %req) = @_;
    
    # 1. URL prefix: /v1/users
    if ($req{path} =~ m{^/(v\d+)/}) {
        return $1 if $self->{versions}{$1};
    }
    
    # 2. Accept header: application/vnd.api+json; version=2
    if (($req{headers}{Accept}//"") =~ /version=(\d+)/) {
        return "v$1" if $self->{versions}{"v$1"};
    }
    
    # 3. Custom header: X-API-Version: 2
    if (my $v = $req{headers}{"X-API-Version"}) {
        my $ver = $v =~ /^v/ ? $v : "v$v";
        return $ver if $self->{versions}{$ver};
    }
    
    return $self->{default};
}

sub dispatch {
    my ($self, $method, $path, %req) = @_;
    $req{path}   = $path;
    $req{method} = $method;
    
    my $ver = $self->detect_version(%req);
    
    # Strip version from path for routing
    (my $clean_path = $path) =~ s{^/v\d+}{};
    $clean_path = "/" if $clean_path eq "";
    
    my $router = $self->{versions}{$ver};
    unless ($router) {
        return { status => 400, body => "{\"error\":\"Unknown API version: $ver\"}" };
    }
    
    my $res = $router->dispatch($method, $clean_path, %req, api_version => $ver);
    $res->{headers}{"X-API-Version"} = $ver;
    return $res;
}
}

package main;

printf "=== API Versioning ===\n\n";

# Fake router for each version
{
package FakeRouter;

sub new {
    my ($class, $ver) = @_;
    return bless { ver => $ver }, $class;
}

sub dispatch {
    my ($self, $method, $path, %req) = @_;
    my $body = JSON::PP->new->utf8->encode({
        version  => $self->{ver},
        method   => $method,
        path     => $path,
        response => "Data from API $self->{ver}",
    });
    return { status => 200, body => $body, headers => {} };
}
}

my $api = API::Versioner->new;
$api->version("v1", FakeRouter->new("v1"));
$api->version("v2", FakeRouter->new("v2"));
$api->version("v3", FakeRouter->new("v3"));
$api->default_version("v2");

my @tests = (
    # [method, path, headers]
    ["GET", "/v1/users", {}],
    ["GET", "/v2/users", {}],
    ["GET", "/v3/products", {}],
    ["GET", "/users",    {}],  # Uses default
    ["GET", "/users",    {"X-API-Version" => "1"}],
    ["GET", "/users",    {Accept => "application/json; version=3"}],
    ["GET", "/v99/foo",  {}],  # Unknown version
);

my $json = JSON::PP->new->utf8;

for my $t (@tests) {
    my ($m, $p, $h) = @$t;
    my $res  = $api->dispatch($m, $p, headers => $h);
    my $data = eval { $json->decode($res->{body}) } // {};
    printf "%s %-25s [%s] => %d (version=%s)\n",
        $m, $p, (join ",",map{"$_=$h->{$_}"}keys%$h)||"-", $res->{status}, $data->{version}//"err";
}
```

---

## Step 375: HATEOAS Links

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

{
package REST::HATEOAS;

my $base_url = "https://api.example.com";

sub set_base { $base_url = $_[1] }

sub link {
    my ($class, $rel, $href, %opts) = @_;
    return {
        rel    => $rel,
        href   => $base_url . $href,
        method => $opts{method} // "GET",
        type   => $opts{type}   // "application/json",
        title  => $opts{title},
    };
}

sub user_links {
    my ($class, $user) = @_;
    my $id = $user->{id};
    return [
        $class->link("self",     "/users/$id",              title => "This user"),
        $class->link("update",   "/users/$id", method=>"PUT",  title => "Update user"),
        $class->link("delete",   "/users/$id", method=>"DELETE", title => "Delete user"),
        $class->link("posts",    "/users/$id/posts",        title => "User's posts"),
        $class->link("collection", "/users",                title => "All users"),
    ];
}

sub post_links {
    my ($class, $post) = @_;
    my $id = $post->{id};
    return [
        $class->link("self",     "/posts/$id"),
        $class->link("update",   "/posts/$id", method=>"PUT"),
        $class->link("delete",   "/posts/$id", method=>"DELETE"),
        $class->link("author",   "/users/$post->{author_id}"),
        $class->link("comments", "/posts/$id/comments"),
        $class->link("collection", "/posts"),
    ];
}

sub paginated_links {
    my ($class, $path, %p) = @_;
    my @links;
    push @links, $class->link("self", "$path?page=$p{page}&per_page=$p{per_page}");
    push @links, $class->link("first", "$path?page=1&per_page=$p{per_page}");
    push @links, $class->link("last",  "$path?page=$p{total_pages}&per_page=$p{per_page}");
    push @links, $class->link("prev",  "$path?page=".($p{page}-1)."&per_page=$p{per_page}")
        if $p{page} > 1;
    push @links, $class->link("next",  "$path?page=".($p{page}+1)."&per_page=$p{per_page}")
        if $p{page} < ($p{total_pages}//1);
    return \@links;
}

sub embed {
    my ($class, $resource, $links) = @_;
    return { %$resource, _links => { map { $_->{rel} => $_ } @$links } };
}
}

package main;

printf "=== HATEOAS Links ===\n\n";

my $json = JSON::PP->new->utf8->pretty->canonical;

# Single user with links
my $user = { id => 1, username => "alice", email => 'alice@example.com' };
my $user_with_links = REST::HATEOAS->embed($user, REST::HATEOAS->user_links($user));
printf "User with links:\n%s\n", $json->encode($user_with_links);

# Paginated collection
my $users = [
    { id => 1, username => "alice" },
    { id => 2, username => "bob" },
];

my $response = {
    data  => [map { REST::HATEOAS->embed($_, REST::HATEOAS->user_links($_)) } @$users],
    _links => REST::HATEOAS->paginated_links("/users", page=>2, per_page=>10, total_pages=>5),
    meta  => { total => 45, page => 2 },
};

printf "Paginated users:\n%s\n", $json->encode($response);
```

---

## Step 376: API Authentication

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(hmac_sha256_base64);
use MIME::Base64;
use JSON::PP;

{
package API::Auth;

# JWT-style token (simplified)
my $SECRET = "super-secret-key-change-in-production";

sub generate_token {
    my ($class, %claims) = @_;
    $claims{iat} //= time();
    $claims{exp} //= time() + 3600;
    $claims{jti} //= sprintf "%08x", rand(0xFFFFFFFF);
    
    my $header  = encode_base64('{"alg":"HS256","typ":"JWT"}', "");
    my $payload = encode_base64(JSON::PP->new->utf8->canonical->encode(\%claims), "");
    $header  =~ s/=+$//; $payload =~ s/=+$//;
    $header  =~ tr|+/|−_|; $payload =~ tr|+/|−_|;
    
    my $sig = hmac_sha256_base64("$header.$payload", $SECRET);
    $sig =~ s/=+$//; $sig =~ tr|+/|−_|;
    
    return "$header.$payload.$sig";
}

sub verify_token {
    my ($class, $token) = @_;
    
    my ($header, $payload, $sig) = split /\./, $token;
    return undef unless $header && $payload && $sig;
    
    # Verify signature
    my $expected = hmac_sha256_base64("$header.$payload", $SECRET);
    $expected =~ s/=+$//; $expected =~ tr|+/|−_|;
    return undef unless $sig eq $expected;
    
    # Decode payload
    (my $padded = $payload) =~ tr|−_|+/|;
    my $pad = (4 - length($padded) % 4) % 4;
    $padded .= "=" x $pad;
    my $claims = eval { JSON::PP->new->utf8->decode(decode_base64($padded)) };
    return undef unless $claims;
    
    # Check expiry
    return undef if ($claims->{exp}//0) < time();
    
    return $claims;
}

# API Key management
my %api_keys;

sub create_api_key {
    my ($class, $user_id, %opts) = @_;
    my $key = "sk_" . join("", map { sprintf "%04x", rand(0xFFFF) } 1..8);
    $api_keys{$key} = {
        user_id    => $user_id,
        key        => $key,
        created_at => time(),
        expires_at => $opts{expires_at},
        scopes     => $opts{scopes} // ["read"],
        name       => $opts{name}   // "API Key",
        last_used  => undef,
    };
    return $key;
}

sub verify_api_key {
    my ($class, $key) = @_;
    my $info = $api_keys{$key} or return undef;
    return undef if $info->{expires_at} && $info->{expires_at} < time();
    $info->{last_used} = time();
    return $info;
}

sub has_scope {
    my ($class, $info, $scope) = @_;
    return grep { $_ eq $scope || $_ eq "*" } @{$info->{scopes}//[]};
}

# Basic Auth
sub verify_basic {
    my ($class, $header, $users) = @_;
    return undef unless $header =~ /^Basic\s+(.+)/;
    my $decoded = decode_base64($1);
    my ($user, $pass) = split /:/, $decoded, 2;
    return undef unless defined $users->{$user} && $users->{$user} eq $pass;
    return { username => $user };
}
}

package main;

printf "=== API Authentication ===\n\n";

# JWT tokens
printf "JWT Tokens:\n";
my $token = API::Auth->generate_token(
    sub    => "user_42",
    role   => "admin",
    scopes => ["read", "write"],
);
printf "  Generated: %s...\n", substr($token, 0, 50);

my $claims = API::Auth->verify_token($token);
printf "  Verified: sub=%s role=%s\n", $claims->{sub}, $claims->{role};

# Expired token
my $old_token = API::Auth->generate_token(sub=>"old", exp=>time()-100);
my $old_claims = API::Auth->verify_token($old_token);
printf "  Expired: %s\n", $old_claims ? "verified (bug!)" : "rejected (correct)";

# Tampered
my $tampered = $token . "x";
my $tc = API::Auth->verify_token($tampered);
printf "  Tampered: %s\n", $tc ? "verified (bug!)" : "rejected (correct)";

# API Keys
printf "\nAPI Keys:\n";
my $key1 = API::Auth->create_api_key(1, scopes => ["read","write"], name => "Admin key");
my $key2 = API::Auth->create_api_key(2, scopes => ["read"],         name => "Readonly key");

my $info1 = API::Auth->verify_api_key($key1);
printf "  Key1 (%s): user=%d scopes=%s\n", $key1, $info1->{user_id}, join(",",@{$info1->{scopes}});
printf "  Has write: %s\n", API::Auth->has_scope($info1,"write") ? "yes":"no";

my $info2 = API::Auth->verify_api_key($key2);
printf "  Key2: has write: %s\n", API::Auth->has_scope($info2,"write") ? "yes":"no";

printf "  Invalid key: %s\n", API::Auth->verify_api_key("sk_invalid") ? "found":"not found";
```

---

## Step 377: Rate Limiting & Throttling

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package API::RateLimit;

sub new {
    my ($class, %opts) = @_;
    return bless {
        window   => $opts{window}    // 60,   # seconds
        limit    => $opts{limit}     // 100,  # requests per window
        burst    => $opts{burst}     // 20,   # burst allowance
        buckets  => {},                        # token buckets by key
        counters => {},                        # sliding window counters
    }, $class;
}

# Sliding window counter
sub check_sliding {
    my ($self, $key) = @_;
    my $now    = time();
    my $window = $self->{window};
    
    $self->{counters}{$key} //= [];
    
    # Remove old entries outside window
    @{$self->{counters}{$key}} = grep { $_ > $now - $window } @{$self->{counters}{$key}};
    
    my $count = scalar @{$self->{counters}{$key}};
    
    if ($count >= $self->{limit}) {
        my $oldest = $self->{counters}{$key}[0];
        my $retry  = $oldest + $window - $now + 1;
        return { allowed => 0, count => $count, limit => $self->{limit},
                 remaining => 0, retry_after => $retry };
    }
    
    push @{$self->{counters}{$key}}, $now;
    return { allowed => 1, count => $count+1, limit => $self->{limit},
             remaining => $self->{limit} - $count - 1, retry_after => 0 };
}

# Token bucket
sub check_bucket {
    my ($self, $key) = @_;
    my $now    = time();
    my $bucket = $self->{buckets}{$key};
    
    unless ($bucket) {
        $self->{buckets}{$key} = $bucket = {
            tokens    => $self->{limit},
            last_fill => $now,
        };
    }
    
    # Refill tokens
    my $elapsed = $now - $bucket->{last_fill};
    my $new_tokens = $elapsed * ($self->{limit} / $self->{window});
    $bucket->{tokens} = min($self->{limit}, $bucket->{tokens} + $new_tokens);
    $bucket->{last_fill} = $now;
    
    if ($bucket->{tokens} < 1) {
        my $wait = (1 - $bucket->{tokens}) / ($self->{limit} / $self->{window});
        return { allowed => 0, tokens => $bucket->{tokens}, retry_after => int($wait)+1 };
    }
    
    $bucket->{tokens}--;
    return { allowed => 1, tokens => $bucket->{tokens}, remaining => int($bucket->{tokens}) };
}

sub min { $_[0] < $_[1] ? $_[0] : $_[1] }

sub headers {
    my ($self, $result) = @_;
    return {
        "X-RateLimit-Limit"     => $self->{limit},
        "X-RateLimit-Remaining" => $result->{remaining} // 0,
        "X-RateLimit-Reset"     => time() + ($self->{window}),
        "Retry-After"           => $result->{retry_after} // 0,
    };
}
}

package main;

printf "=== Rate Limiting ===\n\n";

my $limiter = API::RateLimit->new(window => 60, limit => 10);

printf "Sliding window (limit=10 per 60s):\n";
for my $req (1..12) {
    my $result = $limiter->check_sliding("user:1");
    printf "  Request %2d: %s (remaining=%d)\n",
        $req, $result->{allowed} ? "OK" : "BLOCKED", $result->{remaining};
}

printf "\nToken bucket (limit=10 per 60s):\n";
for my $req (1..12) {
    my $result = $limiter->check_bucket("user:2");
    printf "  Request %2d: %s (tokens=%.2f)\n",
        $req, $result->{allowed} ? "OK" : "BLOCKED", $result->{tokens}//0;
}

# Per-endpoint limits
printf "\nPer-endpoint limits:\n";
my %endpoint_limits = (
    "POST:/api/auth/login"  => API::RateLimit->new(window=>300, limit=>5),
    "POST:/api/users"       => API::RateLimit->new(window=>3600, limit=>10),
    "GET:/api/data"         => API::RateLimit->new(window=>60, limit=>100),
);

for my $endpoint (sort keys %endpoint_limits) {
    my $lim = $endpoint_limits{$endpoint};
    for (1..7) {
        my $r = $lim->check_sliding("ip:1.2.3.4");
        printf "  %-35s req=%d: %s\n", $endpoint, $_, $r->{allowed} ? "OK" : "BLOCKED"
            if $_ == 1 || !$r->{allowed};
        last unless $r->{allowed};
    }
}
```

---

## Step 378: API Documentation (OpenAPI)

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

{
package OpenAPI::Builder;

sub new {
    my ($class, %info) = @_;
    return bless {
        openapi => "3.0.3",
        info    => {
            title       => $info{title}   // "My API",
            version     => $info{version} // "1.0.0",
            description => $info{description},
        },
        servers    => [],
        paths      => {},
        components => {
            schemas         => {},
            securitySchemes => {},
            responses       => {},
        },
        tags => [],
    }, $class;
}

sub server {
    my ($self, $url, $desc) = @_;
    push @{$self->{servers}}, { url => $url, description => $desc };
    return $self;
}

sub tag {
    my ($self, $name, $desc) = @_;
    push @{$self->{tags}}, { name => $name, description => $desc };
    return $self;
}

sub schema {
    my ($self, $name, %def) = @_;
    $self->{components}{schemas}{$name} = \%def;
    return $self;
}

sub security_scheme {
    my ($self, $name, %def) = @_;
    $self->{components}{securitySchemes}{$name} = \%def;
    return $self;
}

sub path {
    my ($self, $path, %methods) = @_;
    $self->{paths}{$path} = \%methods;
    return $self;
}

sub operation {
    my ($class, %opts) = @_;
    return {
        summary     => $opts{summary},
        description => $opts{description},
        tags        => $opts{tags}       // [],
        operationId => $opts{operation_id},
        security    => $opts{security}   // [],
        parameters  => $opts{parameters} // [],
        requestBody => $opts{request_body},
        responses   => $opts{responses}  // { "200" => { description => "Success" } },
    };
}

sub param {
    my ($class, %opts) = @_;
    return {
        name        => $opts{name},
        in          => $opts{in} // "query",
        required    => $opts{required} // JSON::PP::false(),
        schema      => { type => $opts{type} // "string" },
        description => $opts{description},
    };
}

sub body {
    my ($class, $schema_ref) = @_;
    return {
        required => JSON::PP::true(),
        content  => {
            "application/json" => { schema => { '$ref' => "#/components/schemas/$schema_ref" } }
        }
    };
}

sub ref_response {
    my ($class, $schema_ref, $desc) = @_;
    return {
        description => $desc // "OK",
        content => {
            "application/json" => { schema => { '$ref' => "#/components/schemas/$schema_ref" } }
        }
    };
}

sub to_yaml {
    my $self = shift;
    # Simplified YAML output
    my $json = JSON::PP->new->utf8->pretty->canonical->encode($self);
    return $json;
}

sub to_json { JSON::PP->new->utf8->pretty->canonical->encode($_[0]) }
}

package main;

printf "=== OpenAPI Documentation ===\n\n";

my $api = OpenAPI::Builder->new(
    title       => "Bookstore API",
    version     => "2.0.0",
    description => "REST API for managing books and orders",
);

$api->server("https://api.bookstore.com/v2", "Production");
$api->server("http://localhost:3000/v2",     "Development");

$api->tag("books",  "Book management operations");
$api->tag("orders", "Order management");
$api->tag("auth",   "Authentication");

$api->security_scheme("bearerAuth", type => "http", scheme => "bearer", bearerFormat => "JWT");
$api->security_scheme("apiKey",     type => "apiKey", in => "header", name => "X-API-Key");

# Schemas
$api->schema("Book",
    type       => "object",
    required   => ["title","price"],
    properties => {
        id     => { type => "integer", readOnly => JSON::PP::true() },
        title  => { type => "string", example => "Learning Perl" },
        author => { type => "string" },
        isbn   => { type => "string", pattern => "^[0-9X-]+\$" },
        price  => { type => "number", format => "float", minimum => 0 },
        stock  => { type => "integer", minimum => 0, default => 0 },
    }
);

$api->schema("Error",
    type       => "object",
    properties => {
        code    => { type => "string" },
        message => { type => "string" },
        field   => { type => "string" },
    }
);

# Paths
$api->path("/books",
    get => OpenAPI::Builder->operation(
        summary      => "List books",
        operation_id => "listBooks",
        tags         => ["books"],
        parameters   => [
            OpenAPI::Builder->param(name=>"q",       in=>"query", description=>"Search query"),
            OpenAPI::Builder->param(name=>"page",    in=>"query", type=>"integer"),
            OpenAPI::Builder->param(name=>"per_page",in=>"query", type=>"integer"),
        ],
        responses => {
            "200" => { description => "List of books", content => {
                "application/json" => { schema => { type=>"array", items=>{'$ref'=>"#/components/schemas/Book"} } }
            }},
        }
    ),
    post => OpenAPI::Builder->operation(
        summary      => "Create book",
        operation_id => "createBook",
        tags         => ["books"],
        security     => [{ bearerAuth => [] }],
        request_body => OpenAPI::Builder->body("Book"),
        responses    => { "201" => OpenAPI::Builder->ref_response("Book","Created") },
    ),
);

$api->path("/books/{id}",
    get => OpenAPI::Builder->operation(
        summary      => "Get book by ID",
        operation_id => "getBook",
        tags         => ["books"],
        parameters   => [OpenAPI::Builder->param(name=>"id",in=>"path",required=>JSON::PP::true(),type=>"integer")],
        responses    => {
            "200" => OpenAPI::Builder->ref_response("Book","Book found"),
            "404" => { description => "Not found" },
        }
    ),
);

# Output
my $doc = $api->to_json;
printf "OpenAPI document (%d bytes):\n", length($doc);
printf "%s\n", substr($doc, 0, 500);
printf "...\n\n";

# Summary
printf "Paths defined: %d\n",   scalar keys %{$api->{paths}};
printf "Schemas defined: %d\n", scalar keys %{$api->{components}{schemas}};
printf "Tags: %s\n", join(", ", map { $_->{name} } @{$api->{tags}});
```

---

## Step 379: API Testing

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

{
package API::TestClient;

sub new {
    my ($class, $app, %opts) = @_;
    return bless {
        app     => $app,
        base    => $opts{base} // "",
        headers => $opts{headers} // {},
        token   => $opts{token},
        log     => [],
    }, $class;
}

sub set_token { $_[0]->{token} = $_[1]; $_[0] }
sub set_header { $_[0]->{headers}{$_[1]} = $_[2]; $_[0] }

sub _request {
    my ($self, $method, $path, %opts) = @_;
    
    my %headers = (
        "Content-Type" => "application/json",
        %{$self->{headers}},
        %{$opts{headers}//{}},
    );
    $headers{Authorization} = "Bearer $self->{token}" if $self->{token} && !$headers{Authorization};
    
    my $res = $self->{app}->dispatch($method, $self->{base}.$path,
        headers => \%headers,
        params  => $opts{params}  // {},
        body    => $opts{json}    // $opts{body} // {},
    );
    
    push @{$self->{log}}, { method=>$method, path=>$path, status=>$res->{status} };
    return $res;
}

sub get    { shift->_request("GET",    @_) }
sub post   { shift->_request("POST",   @_) }
sub put    { shift->_request("PUT",    @_) }
sub patch  { shift->_request("PATCH",  @_) }
sub delete { shift->_request("DELETE", @_) }

sub json_body {
    my ($self, $res) = @_;
    return eval { JSON::PP->new->utf8->decode($res->{body}) } // {};
}
}

{
package API::TestCase;

my (@tests, @failed, $client);
my $json = JSON::PP->new->utf8;

sub new {
    my ($class, $test_client) = @_;
    $client = $test_client;
    return bless { tests=>0, passed=>0, failed=>0 }, $class;
}

sub describe { printf "\n=== %s ===\n", $_[1] }
sub it       { printf "  %s\n", $_[1] }

sub assert_status {
    my ($self, $res, $expected, $label) = @_;
    $self->{tests}++;
    if ($res->{status} == $expected) {
        printf "    [PASS] %s (status=%d)\n", $label, $expected;
        $self->{passed}++;
    } else {
        printf "    [FAIL] %s: expected %d got %d\n", $label, $expected, $res->{status};
        $self->{failed}++;
    }
}

sub assert_json_key {
    my ($self, $res, $key, $label) = @_;
    my $body = eval { $json->decode($res->{body}) };
    $self->{tests}++;
    if ($body && exists $body->{$key}) {
        printf "    [PASS] %s (has key '%s')\n", $label, $key;
        $self->{passed}++;
    } else {
        printf "    [FAIL] %s: missing key '%s'\n", $label, $key;
        $self->{failed}++;
    }
}

sub assert_count {
    my ($self, $res, $count, $label) = @_;
    my $body = eval { $json->decode($res->{body}) };
    my $actual = ref $body eq 'ARRAY' ? scalar @$body :
                 ref $body eq 'HASH' && ref $body->{data} eq 'ARRAY' ? scalar @{$body->{data}} : 0;
    $self->{tests}++;
    if ($actual == $count) {
        printf "    [PASS] %s (count=%d)\n", $label, $count;
        $self->{passed}++;
    } else {
        printf "    [FAIL] %s: expected count %d got %d\n", $label, $count, $actual;
        $self->{failed}++;
    }
}

sub summary {
    my $self = shift;
    printf "\n--- Results: %d/%d passed ---\n", $self->{passed}, $self->{tests};
}
}

package main;

printf "=== API Testing Suite ===\n";

# Mock API app
{
package MockAPI;
use DBI;
my $db = DBI->connect("dbi:SQLite::memory:","","",{RaiseError=>1,AutoCommit=>1});
$db->do("CREATE TABLE items (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, qty INTEGER DEFAULT 0)");
for my $i (["Widget",10],["Gadget",5],["Doohickey",15]) {
    $db->do("INSERT INTO items (name,qty) VALUES (?,?)", undef, @$i) }

my $json = JSON::PP->new->utf8;

sub dispatch {
    my ($class, $method, $path, %req) = @_;
    if ($path eq "/items" && $method eq "GET") {
        my $items = $db->selectall_arrayref("SELECT * FROM items", {Slice=>{}});
        return { status=>200, body=>$json->encode({data=>$items, count=>scalar @$items}) };
    }
    if ($path =~ m{/items/(\d+)} && $method eq "GET") {
        my $item = $db->selectrow_hashref("SELECT * FROM items WHERE id=?", undef, $1);
        return $item ? {status=>200, body=>$json->encode($item)} : {status=>404, body=>'{"error":"not found"}'};
    }
    if ($path eq "/items" && $method eq "POST") {
        my $body = ref $req{body} eq 'HASH' ? $req{body} : eval{$json->decode($req{body}//"{}"){}} // {};
        my $name = (ref $req{body} eq 'HASH' ? $req{body}{name} : undef) // "New";
        $db->do("INSERT INTO items (name,qty) VALUES (?,?)", undef, $name, 0);
        return {status=>201, body=>$json->encode({id=>$db->last_insert_id, name=>$name})};
    }
    if ($path =~ m{/items/(\d+)} && $method eq "DELETE") {
        $db->do("DELETE FROM items WHERE id=?", undef, $1);
        return {status=>204, body=>""};
    }
    return {status=>404, body=>'{"error":"not found"}'};
}
}

my $client = API::TestClient->new("MockAPI");
my $tc = API::TestCase->new($client);

$tc->describe("GET /items");
{
    my $res = $client->get("/items");
    $tc->assert_status($res, 200, "returns 200");
    $tc->assert_json_key($res, "data", "has data key");
    $tc->assert_count($res, 3, "has 3 items");
}

$tc->describe("GET /items/:id");
{
    my $res = $client->get("/items/1");
    $tc->assert_status($res, 200, "existing item returns 200");
    $tc->assert_json_key($res, "name", "has name field");
    
    my $res2 = $client->get("/items/999");
    $tc->assert_status($res2, 404, "missing item returns 404");
}

$tc->describe("POST /items");
{
    my $res = $client->post("/items", json => { name => "NewItem" });
    $tc->assert_status($res, 201, "create returns 201");
    $tc->assert_json_key($res, "id", "has id in response");
}

$tc->describe("DELETE /items/:id");
{
    my $res = $client->delete("/items/1");
    $tc->assert_status($res, 204, "delete returns 204");
}

$tc->summary;
```

---

## Step 380: Capstone — Full REST API

```perl
#!/usr/bin/perl
# rest_api.pl — Production-grade REST API
use strict;
use warnings;
use DBI;
use JSON::PP;
use Digest::SHA qw(sha256_hex);

my $DB   = DBI->connect("dbi:SQLite::memory:","","",{RaiseError=>1,AutoCommit=>1});
my $JSON = JSON::PP->new->utf8->canonical;

# Schema
for my $sql (split /;/, q{
CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT UNIQUE NOT NULL, email TEXT UNIQUE, password_hash TEXT, role TEXT DEFAULT 'user', active INTEGER DEFAULT 1, created_at INTEGER DEFAULT (strftime('%s','now')));
CREATE TABLE tokens (id INTEGER PRIMARY KEY, user_id INTEGER, token TEXT UNIQUE, expires_at INTEGER, created_at INTEGER DEFAULT (strftime('%s','now')));
CREATE TABLE products (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, description TEXT, price REAL NOT NULL, stock INTEGER DEFAULT 0, active INTEGER DEFAULT 1);
CREATE TABLE wishlist (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, product_id INTEGER, created_at INTEGER DEFAULT (strftime('%s','now')));
}) {
    my $s = $sql; $s =~ s/^\s+|\s+$//g; $DB->do($s) if $s;
}

# Seed
$DB->do("INSERT INTO users (username,email,password_hash,role) VALUES (?,?,?,?)", undef,
    "admin", "admin\@test.com", sha256_hex("password"), "admin");
$DB->do("INSERT INTO users (username,email,password_hash) VALUES (?,?,?)", undef,
    "alice", "alice\@test.com", sha256_hex("alice123"));

for my $p (
    ["Widget A",    "A great widget",  9.99, 100],
    ["Widget B",    "Better widget",  19.99,  50],
    ["Gadget C",    "Cool gadget",     5.99, 200],
    ["Premium D",   "Premium item",   49.99,  10],
) { $DB->do("INSERT INTO products (name,description,price,stock) VALUES (?,?,?,?)", undef, @$p) }

# Helpers
sub ok       { {status=>200, body=>$JSON->encode({success=>1,data=>$_[0]})} }
sub created  { {status=>201, body=>$JSON->encode({success=>1,data=>$_[0]})} }
sub no_content { {status=>204, body=>""} }
sub err      { {status=>$_[0], body=>$JSON->encode({success=>0,error=>$_[1]})} }

sub auth_user {
    my $token = shift;
    return undef unless $token;
    my $tok = $DB->selectrow_hashref("SELECT * FROM tokens WHERE token=? AND expires_at>?", undef, $token, time());
    return undef unless $tok;
    return $DB->selectrow_hashref("SELECT * FROM users WHERE id=? AND active=1", undef, $tok->{user_id});
}

sub generate_token {
    my $user_id = shift;
    my $token = sha256_hex(time() . rand() . $user_id);
    $DB->do("INSERT INTO tokens (user_id,token,expires_at) VALUES (?,?,?)", undef, $user_id, $token, time()+86400);
    return $token;
}

sub paginate {
    my ($rows, $total, $page, $per_page) = @_;
    return {
        data  => $rows,
        meta  => {
            total      => $total+0,
            page       => $page+0,
            per_page   => $per_page+0,
            total_pages=> int(($total+$per_page-1)/$per_page)+0,
        }
    };
}

# API Dispatch
sub dispatch {
    my ($method, $path, %req) = @_;
    
    my $token = do {
        my $auth = $req{headers}{Authorization} // "";
        $auth =~ /^Bearer\s+(.+)/ ? $1 : undef;
    };
    my $user = auth_user($token);
    
    # Auth endpoints
    if ($path eq "/auth/login" && $method eq "POST") {
        my $body = $req{body} // {};
        my $u = $DB->selectrow_hashref("SELECT * FROM users WHERE username=?", undef, $body->{username}//"");
        if ($u && $u->{password_hash} eq sha256_hex($body->{password}//"")) {
            return created({ token => generate_token($u->{id}), user => {id=>$u->{id},username=>$u->{username},role=>$u->{role}} });
        }
        return err(401, "Invalid credentials");
    }
    
    if ($path eq "/auth/me" && $method eq "GET") {
        return err(401, "Not authenticated") unless $user;
        return ok({id=>$user->{id},username=>$user->{username},email=>$user->{email},role=>$user->{role}});
    }
    
    # Products
    if ($path eq "/products" && $method eq "GET") {
        my $page = ($req{params}{page}//1)+0;
        my $pp   = ($req{params}{per_page}//10)+0;
        my $q    = $req{params}{q}//"";
        my ($where,@b) = ("WHERE p.active=1");
        if ($q) { $where .= " AND (p.name LIKE ? OR p.description LIKE ?)"; push @b, "%$q%","%$q%" }
        my $total    = $DB->selectrow_array("SELECT COUNT(*) FROM products p $where", undef, @b)+0;
        my $products = $DB->selectall_arrayref("SELECT * FROM products p $where ORDER BY p.id LIMIT ? OFFSET ?",{Slice=>{}},@b,$pp,($page-1)*$pp);
        return ok(paginate($products,$total,$page,$pp));
    }
    
    if ($path =~ m{/products/(\d+)} && $method eq "GET") {
        my $p = $DB->selectrow_hashref("SELECT * FROM products WHERE id=? AND active=1",undef,$1);
        return $p ? ok($p) : err(404,"Product not found");
    }
    
    if ($path eq "/products" && $method eq "POST") {
        return err(403,"Admin only") unless $user && $user->{role} eq "admin";
        my $b = $req{body}//{};
        return err(400,"name required") unless $b->{name};
        return err(400,"price required") unless defined $b->{price} && $b->{price} > 0;
        $DB->do("INSERT INTO products (name,description,price,stock) VALUES (?,?,?,?)",undef,
            $b->{name},$b->{description}//"", $b->{price}+0, $b->{stock}//0);
        return created($DB->selectrow_hashref("SELECT * FROM products WHERE id=?",undef,$DB->last_insert_id));
    }
    
    # Wishlist
    if ($path eq "/wishlist" && $method eq "GET") {
        return err(401,"Auth required") unless $user;
        my $items = $DB->selectall_arrayref(
            "SELECT p.* FROM wishlist w JOIN products p ON p.id=w.product_id WHERE w.user_id=? ORDER BY w.created_at DESC",
            {Slice=>{}}, $user->{id});
        return ok($items);
    }
    
    if ($path =~ m{/wishlist/(\d+)} && $method eq "POST") {
        return err(401,"Auth required") unless $user;
        my $pid = $1;
        my $exists = $DB->selectrow_array("SELECT COUNT(*) FROM wishlist WHERE user_id=? AND product_id=?",undef,$user->{id},$pid);
        return err(409,"Already in wishlist") if $exists;
        $DB->do("INSERT INTO wishlist (user_id,product_id) VALUES (?,?)",undef,$user->{id},$pid);
        return created({user_id=>$user->{id}, product_id=>$pid+0});
    }
    
    return err(404, "Not found: $method $path");
}

# Tests
printf "=== Full REST API Tests ===\n\n";

# Login
my $login = dispatch("POST","/auth/login",body=>{username=>"admin",password=>"password"});
my $data  = eval{$JSON->decode($login->{body})};
my $admin_token = $data->{data}{token};
printf "Login admin: %d token=%s...\n", $login->{status}, substr($admin_token//"",-8);

my $login2 = dispatch("POST","/auth/login",body=>{username=>"alice",password=>"alice123"});
my $alice_token = eval{$JSON->decode($login2->{body})}->{data}{token};
printf "Login alice: %d\n", $login2->{status};

# Auth me
my $me = dispatch("GET","/auth/me",headers=>{Authorization=>"Bearer $admin_token"});
printf "\nMe: %s\n", $JSON->decode($me->{body})->{data}{username};

# Products
my $prod_list = dispatch("GET","/products");
my $pl = $JSON->decode($prod_list->{body});
printf "\nProducts: total=%d\n", $pl->{data}{meta}{total};

my $search = dispatch("GET","/products",params=>{q=>"Widget"});
my $sl = $JSON->decode($search->{body});
printf "Search 'Widget': %d results\n", scalar @{$sl->{data}{data}};

# Create product (admin)
my $new_prod = dispatch("POST","/products",
    headers => {Authorization=>"Bearer $admin_token"},
    body    => {name=>"New Gadget",price=>29.99,stock=>50});
printf "Create product: %d\n", $new_prod->{status};

# Create product (unauthorized)
my $unauth = dispatch("POST","/products",
    headers => {Authorization=>"Bearer $alice_token"},
    body    => {name=>"X",price=>1});
printf "Create (alice): %d — %s\n", $unauth->{status}, $JSON->decode($unauth->{body})->{error};

# Wishlist
my $wl_add = dispatch("POST","/wishlist/2",headers=>{Authorization=>"Bearer $alice_token"});
printf "\nAdd to wishlist: %d\n", $wl_add->{status};
dispatch("POST","/wishlist/3",headers=>{Authorization=>"Bearer $alice_token"});

my $wl = dispatch("GET","/wishlist",headers=>{Authorization=>"Bearer $alice_token"});
my $wl_data = $JSON->decode($wl->{body});
printf "Wishlist items: %d\n", scalar @{$wl_data->{data}};
printf "  - %s (\$%.2f)\n", $_->{name}, $_->{price} for @{$wl_data->{data}};

# Duplicate wishlist
my $dup = dispatch("POST","/wishlist/2",headers=>{Authorization=>"Bearer $alice_token"});
printf "Duplicate wishlist: %d — %s\n", $dup->{status}, $JSON->decode($dup->{body})->{error};
```

---

## สรุป Part 38 — REST API Design Patterns

### สิ่งที่เรียนรู้:
- **REST::Response** — Factory methods: ok/created/not_found/unprocessable + pagination metadata
- **Resource Router** — CRUD routing with resource() helper, middleware, path generation
- **Request Handling** — Validation, bearer tokens, query/body parsing
- **API Versioning** — URL prefix, Accept header, X-API-Version detection
- **HATEOAS** — Hypermedia links embedded in responses
- **Authentication** — JWT tokens, API keys, Basic Auth
- **Rate Limiting** — Sliding window + token bucket algorithms
- **OpenAPI Docs** — Builder pattern for Swagger/OpenAPI 3.0
- **Testing** — TestClient + assertion helpers
- **Capstone** — Full REST API with auth, products, wishlist

**ถัดไป: [Part 39 — WebSockets & Real-time Perl](part_39.md)**
