# Part 34: Dancer2 Web Framework
## Steps 331-340: Dancer2 — Modern Perl Web Framework

---

## Step 331: Dancer2 Basics

```perl
#!/usr/bin/perl
# dancer2_demo.pl — Simulated Dancer2-style framework
# Note: Real Dancer2 requires 'cpan Dancer2' install
# This demonstrates the patterns and architecture

use strict;
use warnings;

# Simulate Dancer2 DSL in pure Perl
{
package Dancer2::Lite;

my @routes;
my %config = (
    appname  => "MyApp",
    port     => 3000,
    host     => "0.0.0.0",
    logger   => "console",
    charset  => "UTF-8",
    session  => "Simple",
);

my %sessions;
my %stash;
my @before_hooks;
my @after_hooks;

# DSL functions
sub get    { push @routes, { method => "GET",    path => $_[0], code => $_[1] } }
sub post   { push @routes, { method => "POST",   path => $_[0], code => $_[1] } }
sub put    { push @routes, { method => "PUT",    path => $_[0], code => $_[1] } }
sub del    { push @routes, { method => "DELETE", path => $_[0], code => $_[1] } }
sub any    {
    my ($methods, $path, $code) = @_;
    for my $m (ref $methods ? @$methods : ($methods)) {
        push @routes, { method => uc($m), path => $path, code => $code };
    }
}
sub before { push @before_hooks, $_[0] }
sub after  { push @after_hooks, $_[0] }

sub config { @_ > 1 ? ($config{$_[0]} = $_[1]) : $config{$_[0]} }
sub set    { $config{$_[0]} = $_[1] }

# Response helpers
sub status { $_[0] }
sub send_file { "FILE:$_[0]" }
sub redirect { "REDIRECT:$_[0]" }
sub halt     { die "HALT:$_[0]" }

# Session
my $current_session;
sub session {
    $current_session //= do {
        my $id = sprintf "%016x", rand(0xFFFFFFFFFFFFFFFF);
        $sessions{$id} = {};
        $id;
    };
    return @_ > 1 ? ($sessions{$current_session}{$_[0]} = $_[1])
         : @_ == 1 ? $sessions{$current_session}{$_[0]}
         : $sessions{$current_session};
}

# Request simulation
my %current_request;
sub request  { \%current_request }
sub params   { %current_request{qw(body_params query_params)} }
sub param    { $current_request{params}{$_[0]} }
sub body_params { $current_request{body_params} }

sub dispatch {
    my ($class, $method, $path, %opts) = @_;
    $method = uc($method);
    %current_request = (method => $method, path => $path, %opts,
                         params => $opts{params}//{});
    $current_session = undef;
    
    # Run before hooks
    for my $hook (@before_hooks) {
        eval { $hook->() };
    }
    
    # Find route
    for my $route (@routes) {
        next unless $route->{method} eq $method;
        my $regex = $route->{path};
        $regex =~ s{:(\w+)}{(?<$1>[^/]+)}g;
        $regex = qr{^$regex$};
        
        if ($path =~ $regex) {
            $current_request{route_params} = {%+};
            my $result = eval { $route->{code}->() };
            if ($@) {
                return { status => 500, body => "Error: $@" } unless $@ =~ /^HALT:/;
                return { status => 200, body => $@ =~ s/^HALT://r };
            }
            
            # Run after hooks
            for my $hook (@after_hooks) { eval { $hook->() } }
            
            return { status => 200, body => $result };
        }
    }
    
    return { status => 404, body => "Not Found: $method $path" };
}

# Template rendering (simple)
sub template {
    my ($name, $vars) = @_;
    $vars //= {};
    my $t = "<!-- template: $name -->\n";
    $t   .= "<ul>";
    $t   .= "<li>$_: $vars->{$_}</li>" for sort keys %$vars;
    $t   .= "</ul>";
    return $t;
}

sub to_json {
    require JSON::PP;
    return JSON::PP->new->utf8->encode($_[0]);
}
}

# ---- Application routes ----
package main;
use JSON::PP;

# Config
Dancer2::Lite::set("appname", "TMS API");

# Before hook (auth)
Dancer2::Lite::before(sub {
    my $req = Dancer2::Lite::request();
    # Log every request (simulated)
    # print "  [LOG] $req->{method} $req->{path}\n";
});

# Routes
Dancer2::Lite::get("/", sub {
    return "Welcome to " . Dancer2::Lite::config("appname");
});

Dancer2::Lite::get("/hello/:name", sub {
    my $name = Dancer2::Lite::request()->{route_params}{name};
    return "Hello, $name!";
});

Dancer2::Lite::get("/api/status", sub {
    return Dancer2::Lite::to_json({ status => "ok", app => Dancer2::Lite::config("appname") });
});

Dancer2::Lite::post("/api/echo", sub {
    my $req = Dancer2::Lite::request();
    return Dancer2::Lite::to_json({ echo => $req->{params} });
});

Dancer2::Lite::get("/session/set", sub {
    Dancer2::Lite::session("user", "alice");
    Dancer2::Lite::session("role", "admin");
    return "Session set";
});

Dancer2::Lite::get("/session/get", sub {
    my $user = Dancer2::Lite::session("user");
    return $user ? "Logged in as $user" : "Not logged in";
});

# Test
printf "=== Dancer2-Style Framework ===\n\n";

my @tests = (
    ["GET",  "/",                 {}],
    ["GET",  "/hello/Alice",      {}],
    ["GET",  "/hello/World",      {}],
    ["GET",  "/api/status",       {}],
    ["POST", "/api/echo",         { params => { name => "Bob", age => 25 } }],
    ["GET",  "/not-found",        {}],
);

for my $t (@tests) {
    my ($method, $path, %opts) = @$t;
    my $res = Dancer2::Lite::dispatch($method, $path, %opts);
    printf "%s %-20s => %d: %s\n", $method, $path, $res->{status},
        length($res->{body}) > 50 ? substr($res->{body},0,50)."..." : $res->{body};
}
```

---

## Step 332: Middleware and Hooks

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Middleware;

sub logger {
    my $next = shift;
    return sub {
        my ($method, $path) = @_;
        my $start = time();
        my $res   = $next->($method, $path, @_[2..$#_]);
        printf "  [%s] %s %s => %d (%.3fs)\n",
            scalar(localtime()), $method, $path, $res->{status}, time()-$start;
        return $res;
    };
}

sub cors {
    my $next   = shift;
    my %opts   = @_;
    my $origin = $opts{allow_origin} // "*";
    return sub {
        my $res = $next->(@_);
        $res->{headers}{"Access-Control-Allow-Origin"} = $origin;
        $res->{headers}{"Access-Control-Allow-Methods"} = "GET, POST, PUT, DELETE";
        $res->{headers}{"Access-Control-Allow-Headers"} = "Content-Type, Authorization";
        return $res;
    };
}

sub rate_limit {
    my $next   = shift;
    my %opts   = @_;
    my %counts;
    return sub {
        my ($method, $path, %ctx) = @_;
        my $ip  = $ctx{ip} // "127.0.0.1";
        my $key = "$ip";
        $counts{$key}++;
        if ($counts{$key} > ($opts{max}//100)) {
            return { status => 429, body => "Too Many Requests", headers => {} };
        }
        return $next->($method, $path, %ctx);
    };
}

sub auth {
    my $next = shift;
    my %opts = @_;
    my $skip = $opts{skip} // [];
    return sub {
        my ($method, $path, %ctx) = @_;
        # Skip public paths
        return $next->($method, $path, %ctx) if grep { $path eq $_ } @$skip;
        
        my $token = $ctx{token};
        unless ($token && $token eq "valid_token") {
            return { status => 401, body => "Unauthorized", headers => {} };
        }
        return $next->($method, $path, %ctx);
    };
}

sub compress {
    my $next = shift;
    return sub {
        my $res = $next->(@_);
        if (length($res->{body}) > 1024) {
            $res->{headers}{"Content-Encoding"} = "gzip (simulated)";
        }
        return $res;
    };
}
}

# Simple app for middleware demo
{
package App;

my %handlers = (
    "GET /"        => sub { { status => 200, body => "Home", headers => {} } },
    "GET /api/data"=> sub { { status => 200, body => '{"data":"secret"}', headers => {} } },
    "GET /public"  => sub { { status => 200, body => "Public page", headers => {} } },
);

sub handle {
    my ($method, $path, %ctx) = @_;
    my $key = "$method $path";
    return $handlers{$key} ? $handlers{$key}->() : { status => 404, body => "Not Found", headers => {} };
}
}

package main;

printf "=== Middleware Composition ===\n\n";

# Build middleware chain
my $app = \&App::handle;
$app    = Middleware::compress($app);
$app    = Middleware::cors($app, allow_origin => "https://frontend.example.com");
$app    = Middleware::auth($app, skip => ["/", "/public"]);
$app    = Middleware::rate_limit($app, max => 5);

my @requests = (
    ["GET", "/",         { ip => "1.1.1.1" }],
    ["GET", "/public",   { ip => "1.1.1.1" }],
    ["GET", "/api/data", { ip => "1.1.1.1" }],               # No token
    ["GET", "/api/data", { ip => "1.1.1.1", token => "valid_token" }],  # With token
    ["GET", "/api/data", { ip => "2.2.2.2", token => "valid_token" }],
);

for my $req (@requests) {
    my ($method, $path, $ctx) = @$req;
    my $res = $app->($method, $path, %$ctx);
    printf "%s %-15s token=%-14s => %d: %s %s\n",
        $method, $path,
        ($ctx->{token}//"(none)"),
        $res->{status},
        $res->{body},
        $res->{headers}{"Access-Control-Allow-Origin"} ? "[CORS set]" : "";
}

# Rate limit demo
printf "\n--- Rate limit ---\n";
for my $i (1..7) {
    my $res = $app->("GET", "/", { ip => "3.3.3.3" });
    printf "Request %d: %d %s\n", $i, $res->{status}, $res->{body};
}
```

---

## Step 333: Database Integration with Dancer2

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;
use JSON::PP;

{
package App::DB;

my $dbh;

sub connect {
    $dbh = DBI->connect("dbi:SQLite::memory:", "", "", {
        RaiseError     => 1,
        AutoCommit     => 1,
        sqlite_unicode => 1,
    });
    $dbh->do("PRAGMA foreign_keys = ON");
    _init_schema();
    _seed_data();
    return $dbh;
}

sub handle { $dbh }

sub _init_schema {
    $dbh->do(q{
        CREATE TABLE IF NOT EXISTS articles (
            id         INTEGER PRIMARY KEY AUTOINCREMENT,
            title      TEXT NOT NULL,
            slug       TEXT UNIQUE,
            body       TEXT,
            author     TEXT DEFAULT 'admin',
            published  INTEGER DEFAULT 0,
            views      INTEGER DEFAULT 0,
            created_at INTEGER DEFAULT (strftime('%s','now'))
        )
    });
    $dbh->do(q{
        CREATE TABLE IF NOT EXISTS categories (
            id   INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT UNIQUE,
            slug TEXT UNIQUE
        )
    });
    $dbh->do(q{
        CREATE TABLE IF NOT EXISTS article_categories (
            article_id  INTEGER REFERENCES articles(id),
            category_id INTEGER REFERENCES categories(id),
            PRIMARY KEY (article_id, category_id)
        )
    });
}

sub _seed_data {
    my @cats = (["Technology", "technology"], ["Programming", "programming"], ["Tutorial", "tutorial"]);
    $dbh->do("INSERT INTO categories (name, slug) VALUES (?,?)", undef, @$_) for @cats;
    
    my @arts = (
        ["Intro to Perl", "intro-perl", "Perl is a great language.", "alice", 1, 150],
        ["DBI in Perl",   "dbi-perl",   "Using DBI for databases.", "alice", 1, 89],
        ["CGI Basics",    "cgi-basics", "CGI web development.",     "bob",   1, 67],
        ["Dancer2 Guide", "dancer2",    "The Dancer2 framework.",   "alice", 1, 203],
        ["Draft Post",    "draft",      "Not ready yet.",           "bob",   0, 0],
    );
    for my $a (@arts) {
        $dbh->do("INSERT INTO articles (title,slug,body,author,published,views) VALUES (?,?,?,?,?,?)",
            undef, @$a);
    }
    
    # Associate articles with categories
    $dbh->do("INSERT INTO article_categories VALUES (1,1)");
    $dbh->do("INSERT INTO article_categories VALUES (1,2)");
    $dbh->do("INSERT INTO article_categories VALUES (2,2)");
    $dbh->do("INSERT INTO article_categories VALUES (2,3)");
    $dbh->do("INSERT INTO article_categories VALUES (3,1)");
    $dbh->do("INSERT INTO article_categories VALUES (4,1)");
    $dbh->do("INSERT INTO article_categories VALUES (4,2)");
    $dbh->do("INSERT INTO article_categories VALUES (4,3)");
}
}

{
package App::Controller::Articles;

sub index {
    my ($class, %params) = @_;
    my $page     = $params{page}     // 1;
    my $per_page = $params{per_page} // 3;
    my $dbh      = App::DB->handle;
    
    my $total = $dbh->selectrow_array("SELECT COUNT(*) FROM articles WHERE published=1");
    my $items = $dbh->selectall_arrayref(
        "SELECT * FROM articles WHERE published=1 ORDER BY created_at DESC LIMIT ? OFFSET ?",
        {Slice=>{}}, $per_page, ($page-1)*$per_page
    );
    
    return { items => $items, total => $total, page => $page, per_page => $per_page };
}

sub show {
    my ($class, $slug) = @_;
    my $dbh  = App::DB->handle;
    my $art  = $dbh->selectrow_hashref("SELECT * FROM articles WHERE slug=? AND published=1", undef, $slug);
    return undef unless $art;
    
    # Increment views
    $dbh->do("UPDATE articles SET views=views+1 WHERE id=?", undef, $art->{id});
    
    # Get categories
    $art->{categories} = $dbh->selectall_arrayref(
        "SELECT c.name, c.slug FROM categories c
         JOIN article_categories ac ON c.id=ac.category_id
         WHERE ac.article_id=?", {Slice=>{}}, $art->{id}
    );
    
    return $art;
}

sub by_category {
    my ($class, $cat_slug) = @_;
    my $dbh = App::DB->handle;
    return $dbh->selectall_arrayref(q{
        SELECT a.* FROM articles a
        JOIN article_categories ac ON a.id=ac.article_id
        JOIN categories c ON ac.category_id=c.id
        WHERE c.slug=? AND a.published=1
        ORDER BY a.views DESC
    }, {Slice=>{}}, $cat_slug);
}

sub create {
    my ($class, %data) = @_;
    my $dbh = App::DB->handle;
    $dbh->do("INSERT INTO articles (title,slug,body,author,published) VALUES (?,?,?,?,?)",
        undef, $data{title}, $data{slug}//_slugify($data{title}),
        $data{body}, $data{author}//"admin", $data{published}//0);
    return $dbh->selectrow_hashref("SELECT * FROM articles WHERE id=?", undef, $dbh->last_insert_id);
}

sub _slugify { my $s = lc(shift); $s =~ s/[^a-z0-9]+/-/g; $s =~ s/^-|-$//g; $s }

sub stats {
    my $dbh = App::DB->handle;
    return {
        total_articles  => $dbh->selectrow_array("SELECT COUNT(*) FROM articles"),
        published       => $dbh->selectrow_array("SELECT COUNT(*) FROM articles WHERE published=1"),
        total_views     => $dbh->selectrow_array("SELECT SUM(views) FROM articles") // 0,
        categories      => $dbh->selectrow_array("SELECT COUNT(*) FROM categories"),
        top_article     => $dbh->selectrow_hashref("SELECT title, views FROM articles ORDER BY views DESC LIMIT 1"),
    };
}
}

package main;

App::DB->connect;
my $json = JSON::PP->new->utf8;

printf "=== Article API ===\n\n";

# List
my $list = App::Controller::Articles->index(page => 1, per_page => 3);
printf "Articles (page 1): %d/%d\n", scalar @{$list->{items}}, $list->{total};
printf "  [%s] %s (%d views)\n", $_->{slug}, $_->{title}, $_->{views} for @{$list->{items}};

# Single article
printf "\nArticle: dancer2\n";
my $art = App::Controller::Articles->show("dancer2");
printf "  Title:  %s\n", $art->{title};
printf "  Author: %s\n", $art->{author};
printf "  Views:  %d\n", $art->{views};
printf "  Categories: %s\n", join(", ", map { $_->{name} } @{$art->{categories}});

# By category
my @prog = @{App::Controller::Articles->by_category("programming")};
printf "\nProgramming articles: %d\n", scalar @prog;
printf "  %s (%d views)\n", $_->{title}, $_->{views} for @prog;

# Stats
my $stats = App::Controller::Articles->stats;
printf "\nStats:\n";
printf "  %-20s %s\n", $_, $stats->{$_} for grep { !ref($stats->{$_}) } keys %$stats;
printf "  top_article: %s (%d views)\n", $stats->{top_article}{title}, $stats->{top_article}{views};
```

---

## Step 334: JSON API with Dancer2

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

{
package API::Response;

my $json = JSON::PP->new->utf8->canonical;

sub ok {
    my ($class, $data, $code) = @_;
    return { status => $code//200, body => $json->encode($data),
             headers => { "Content-Type" => "application/json" } };
}

sub created    { API::Response->ok($_[1], 201) }
sub no_content { { status => 204, body => "", headers => {} } }
sub bad_request { API::Response->error($_[1], 400) }
sub unauthorized { API::Response->error($_[1]//"Unauthorized", 401) }
sub not_found   { API::Response->error($_[1]//"Not Found", 404) }

sub error {
    my ($class, $msg, $code) = @_;
    return { status => $code//500,
             body   => $json->encode({ error => $msg, status => $code//500 }),
             headers => { "Content-Type" => "application/json" } };
}

sub paginated {
    my ($class, $items, $total, $page, $per_page) = @_;
    return $class->ok({
        data  => $items,
        meta  => {
            total    => $total+0,
            page     => $page+0,
            per_page => $per_page+0,
            pages    => int(($total + $per_page - 1) / $per_page) || 1,
        },
    });
}
}

{
package API::Router;

my @routes;

sub add { push @routes, { method => uc($_[1]), path => $_[2], handler => $_[3] } }

sub dispatch {
    my ($class, $method, $path, %ctx) = @_;
    $method = uc($method);
    
    for my $r (@routes) {
        next unless $r->{method} eq $method;
        my $regex = $r->{path};
        $regex =~ s{:(\w+)}{(?<$1>[^/]+)}g;
        $regex = qr{^$regex$};
        next unless $path =~ $regex;
        
        my %params = %+;
        eval { return $r->{handler}->({ %ctx, path_params => \%params }) };
        return API::Response->error("$@") if $@;
    }
    
    return API::Response->not_found("$method $path");
}
}

package main;

my $json  = JSON::PP->new->utf8;
App::DB->connect if !eval { App::DB->handle->ping };

# Register API routes
API::Router->add("API", "GET",  "/api/articles",     sub {
    my $ctx  = shift;
    my $page = $ctx->{params}{page} // 1;
    my $list = App::Controller::Articles->index(page => $page, per_page => 3);
    return API::Response->paginated($list->{items}, $list->{total}, $page, 3);
});

API::Router->add("API", "GET",  "/api/articles/:slug", sub {
    my $ctx  = shift;
    my $slug = $ctx->{path_params}{slug};
    my $art  = App::Controller::Articles->show($slug);
    return $art ? API::Response->ok($art) : API::Response->not_found("Article not found");
});

API::Router->add("API", "POST", "/api/articles", sub {
    my $ctx  = shift;
    my $body = eval { $json->decode($ctx->{body}//"{}") };
    return API::Response->bad_request("Invalid JSON") if $@;
    
    return API::Response->bad_request("title required") unless $body->{title};
    
    my $art = App::Controller::Articles->create(%$body);
    return API::Response->created($art);
});

API::Router->add("API", "GET", "/api/stats", sub {
    return API::Response->ok(App::Controller::Articles->stats);
});

# Test API
printf "=== JSON API Tests ===\n\n";

my @api_tests = (
    ["GET",  "/api/articles",          {}],
    ["GET",  "/api/articles/perl-tips", {}],
    ["GET",  "/api/articles/dancer2",   {}],
    ["GET",  "/api/articles/not-exist", {}],
    ["POST", "/api/articles",           { body => '{"title":"New Post","body":"Content here."}' }],
    ["GET",  "/api/stats",              {}],
);

for my $t (@api_tests) {
    my ($method, $path, $ctx) = @$t;
    my $res  = API::Router->dispatch($method, $path, %$ctx);
    my $data = eval { $json->decode($res->{body}) };
    printf "%s %-30s %d: %s\n", $method, $path, $res->{status},
        $@ ? $res->{body} : do {
            if (ref $data eq 'HASH' && $data->{data}) {
                "items=" . scalar(@{$data->{data}}) . " total=$data->{meta}{total}"
            } elsif (ref $data eq 'HASH' && $data->{error}) {
                "error=$data->{error}"
            } elsif (ref $data eq 'HASH' && $data->{title}) {
                "title=$data->{title}"
            } else {
                substr($json->encode($data),0,60)
            }
        };
}
```

---

## Step 335: Authentication with Dancer2

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(sha256_hex hmac_sha256_hex);
use MIME::Base64 qw(encode_base64url decode_base64url);
use JSON::PP;

{
package JWT;

my $SECRET = "jwt_secret_key_change_this";
my $json   = JSON::PP->new->utf8->canonical;

sub encode {
    my ($class, %payload) = @_;
    $payload{iat} = time();
    $payload{exp} //= time() + 3600;
    
    my $header = encode_base64url('{"alg":"HS256","typ":"JWT"}');
    my $body   = encode_base64url($json->encode(\%payload));
    my $sig    = encode_base64url(hmac_sha256_hex("$header.$body", $SECRET));
    
    return "$header.$body.$sig";
}

sub decode {
    my ($class, $token) = @_;
    return undef unless $token;
    
    my ($header, $body, $sig) = split /\./, $token;
    return undef unless $header && $body && $sig;
    
    # Verify signature
    my $expected = encode_base64url(hmac_sha256_hex("$header.$body", $SECRET));
    return undef unless _const_cmp($sig, $expected);
    
    # Decode payload
    my $payload = eval { $json->decode(decode_base64url($body)) };
    return undef if $@;
    
    # Check expiry
    return undef if $payload->{exp} && time() > $payload->{exp};
    
    return $payload;
}

sub _const_cmp {
    my ($a, $b) = @_;
    return 0 if length($a) != length($b);
    my $diff = 0;
    $diff |= ord(substr($a,$_,1)) ^ ord(substr($b,$_,1)) for 0..length($a)-1;
    return $diff == 0;
}
}

{
package Auth::Controller;

my %users = (
    "alice" => { id => 1, password => sha256_hex("password123"), role => "admin" },
    "bob"   => { id => 2, password => sha256_hex("password456"), role => "user" },
);

sub login {
    my ($class, $username, $password) = @_;
    my $user = $users{$username};
    return (undef, "Invalid credentials")
        unless $user && $user->{password} eq sha256_hex($password);
    
    my $token = JWT->encode(
        sub  => $user->{id},
        user => $username,
        role => $user->{role},
    );
    
    return ($token, undef);
}

sub require_auth {
    my ($class, $token) = @_;
    my $payload = JWT->decode($token);
    return (undef, "Token invalid or expired") unless $payload;
    return ($payload, undef);
}

sub require_role {
    my ($class, $token, $required_role) = @_;
    my ($payload, $err) = $class->require_auth($token);
    return (undef, $err) if $err;
    
    my %role_levels = (user => 1, moderator => 2, admin => 3);
    my $user_level  = $role_levels{$payload->{role}} // 0;
    my $req_level   = $role_levels{$required_role}   // 99;
    
    return (undef, "Insufficient permissions") unless $user_level >= $req_level;
    return ($payload, undef);
}
}

package main;

printf "=== JWT Authentication ===\n\n";

# Login
my ($token, $err) = Auth::Controller->login("alice", "password123");
printf "Login alice: %s\n", $token ? "OK (token=" . substr($token,0,30) . "...)" : "FAIL: $err";

my ($bad_token) = Auth::Controller->login("alice", "wrong");
printf "Login wrong: %s\n", $bad_token ? "bypassed" : "correctly rejected";

# Decode
if ($token) {
    my ($payload) = Auth::Controller->require_auth($token);
    printf "Token valid: user=%s role=%s\n", $payload->{user}, $payload->{role};
    
    # Role check
    my ($admin_ok, $admin_err) = Auth::Controller->require_role($token, "admin");
    printf "Admin access: %s\n", $admin_ok ? "granted" : "denied: $admin_err";
    
    my ($super_ok, $super_err) = Auth::Controller->require_role($token, "superuser");
    printf "Superuser access: %s\n", $super_ok ? "granted" : "denied: $super_err";
}

# Bob (user role)
my ($bob_token) = Auth::Controller->login("bob", "password456");
if ($bob_token) {
    my ($admin_ok, $err) = Auth::Controller->require_role($bob_token, "admin");
    printf "\nBob admin access: %s\n", $admin_ok ? "granted (bad)" : "correctly denied";
    
    my ($user_ok) = Auth::Controller->require_role($bob_token, "user");
    printf "Bob user access: %s\n", $user_ok ? "granted (correct)" : "denied";
}

# Invalid token
my ($inv) = Auth::Controller->require_auth("not.a.valid.token");
printf "\nInvalid token: %s\n", $inv ? "bypassed (bad)" : "correctly rejected";
```

---

## Step 336: Dancer2 Plugins Pattern

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Simulate plugin system
{
package Plugin::Base;

sub register {
    my ($class, $app, %config) = @_;
    $class->_install_hooks($app, %config);
    printf "  Plugin %s registered\n", $class;
}

sub _install_hooks { }
}

{
package Plugin::CSRF;
use parent -norequire, 'Plugin::Base';
use Digest::SHA qw(hmac_sha256_hex);

my $secret = "csrf_secret";

sub _install_hooks {
    my ($class, $app, %config) = @_;
    my @safe_methods = qw(GET HEAD OPTIONS);
    
    $app->add_before_filter(sub {
        my $req = shift;
        return if grep { uc($req->{method}) eq $_ } @safe_methods;
        
        my $token    = $req->{params}{csrf_token}//"";
        my $session  = $req->{session_id}//"";
        my $expected = hmac_sha256_hex($session, $secret);
        
        unless (length($token) > 10) {
            return { status => 403, body => "CSRF check failed" };
        }
    });
}
}

{
package Plugin::RateLimit;
use parent -norequire, 'Plugin::Base';

my %hits;

sub _install_hooks {
    my ($class, $app, %config) = @_;
    my $max    = $config{max}    // 100;
    my $window = $config{window} // 60;
    
    $app->add_before_filter(sub {
        my $req = shift;
        my $ip  = $req->{ip} // "unknown";
        my $now = time();
        
        $hits{$ip} //= [];
        @{$hits{$ip}} = grep { $now - $_ < $window } @{$hits{$ip}};
        push @{$hits{$ip}}, $now;
        
        if (@{$hits{$ip}} > $max) {
            return { status => 429, body => "Too Many Requests" };
        }
        return undef;  # continue
    });
}
}

{
package Plugin::Logger;
use parent -norequire, 'Plugin::Base';

sub _install_hooks {
    my ($class, $app, %config) = @_;
    my $level = $config{level} // "info";
    
    $app->add_before_filter(sub {
        my $req = shift;
        printf "  [LOG-%s] %s %s\n", uc($level), $req->{method}//"?", $req->{path}//"?";
        return undef;  # continue
    });
}
}

{
package MiniApp;

sub new {
    my ($class) = @_;
    return bless {
        routes  => [],
        filters => [],
    }, $class;
}

sub add_before_filter { push @{$_[0]->{filters}}, $_[1] }

sub use_plugin {
    my ($self, $plugin, %config) = @_;
    $plugin->register($self, %config);
}

sub get  { push @{$_[0]->{routes}}, { method => "GET",  path => $_[1], code => $_[2] } }
sub post { push @{$_[0]->{routes}}, { method => "POST", path => $_[1], code => $_[2] } }

sub dispatch {
    my ($self, $method, $path, %ctx) = @_;
    my $req = { method => $method, path => $path, %ctx };
    
    # Run before filters
    for my $f (@{$self->{filters}}) {
        my $result = $f->($req);
        return $result if $result;
    }
    
    # Find route
    for my $r (@{$self->{routes}}) {
        next unless uc($r->{method}) eq uc($method);
        my $regex = $r->{path};
        $regex =~ s{:(\w+)}{(?<$1>[^/]+)}g;
        next unless $path =~ qr{^$regex$};
        return { status => 200, body => $r->{code}->($req) };
    }
    
    return { status => 404, body => "Not Found" };
}
}

package main;

printf "=== Plugin System ===\n\n";

printf "Registering plugins:\n";
my $app = MiniApp->new;
$app->use_plugin("Plugin::Logger",    level  => "info");
$app->use_plugin("Plugin::RateLimit", max    => 3, window => 60);
$app->use_plugin("Plugin::CSRF");

$app->get("/",       sub { "Home page" });
$app->get("/public", sub { "Public page" });
$app->post("/data",  sub { "Data saved" });

printf "\nDispatching requests:\n";
my @tests = (
    ["GET",  "/",       { ip => "1.1.1.1" }],
    ["GET",  "/public", { ip => "1.1.1.1" }],
    ["GET",  "/public", { ip => "1.1.1.1" }],
    ["GET",  "/public", { ip => "1.1.1.1" }],
    ["GET",  "/public", { ip => "1.1.1.1" }],  # Rate limited
);

for my $t (@tests) {
    my ($method, $path, $ctx) = @$t;
    my $res = $app->dispatch($method, $path, %$ctx);
    printf "  => %d: %s\n", $res->{status}, $res->{body};
}
```

---

## Step 337: Dancer2 Testing

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Test::More;
use JSON::PP;

# Test helper for Dancer2-style apps
{
package Test::Dancer2;

sub new {
    my ($class, $app) = @_;
    return bless { app => $app }, $class;
}

sub request {
    my ($self, $method, $path, %opts) = @_;
    return $self->{app}->dispatch($method, $path, %opts);
}

sub get  { $_[0]->request("GET",  $_[1], %{$_[2]//{}}) }
sub post { $_[0]->request("POST", $_[1], %{$_[2]//{}}) }
sub put  { $_[0]->request("PUT",  $_[1], %{$_[2]//{}}) }
sub del  { $_[0]->request("DELETE",$_[1], %{$_[2]//{}}) }

sub json_body {
    my ($self, $res) = @_;
    return eval { JSON::PP->new->utf8->decode($res->{body}) };
}
}

# Create test app
my $app = MiniApp->new;
my $json = JSON::PP->new->utf8;

my %items = (1 => { id => 1, name => "Apple" }, 2 => { id => 2, name => "Banana" });
my $next_id = 3;

$app->get("/api/items", sub {
    $json->encode([values %items])
});

$app->get("/api/items/:id", sub {
    my $req = shift;
    (my $path = $req->{path}) =~ m{/(\d+)$};
    my $id = $1;
    return $items{$id} ? $json->encode($items{$id}) : do { "{\"error\":\"not found\"}" };
});

$app->post("/api/items", sub {
    my $req  = shift;
    my $data = eval { $json->decode($req->{body}//"{}") };
    return '{"error":"invalid"}' unless $data && $data->{name};
    my $id   = $next_id++;
    $items{$id} = { id => $id, name => $data->{name} };
    return $json->encode($items{$id});
});

# Tests
my $t = Test::Dancer2->new($app);

subtest "GET /api/items" => sub {
    my $res = $t->get("/api/items");
    is($res->{status}, 200, "status 200");
    my $data = $t->json_body($res);
    ok(ref $data eq 'ARRAY', "returns array");
    cmp_ok(scalar @$data, ">=", 2, "at least 2 items");
};

subtest "GET /api/items/:id" => sub {
    my $res = $t->get("/api/items/1");
    is($res->{status}, 200, "existing item returns 200");
    my $data = $t->json_body($res);
    is($data->{id},   1,       "correct id");
    is($data->{name}, "Apple", "correct name");
};

subtest "GET /api/items/:id 404" => sub {
    my $res = $t->get("/api/items/999");
    my $data = $t->json_body($res);
    ok($data->{error}, "has error key");
};

subtest "POST /api/items" => sub {
    my $res = $t->post("/api/items", { body => '{"name":"Cherry"}' });
    is($res->{status}, 200, "post ok");
    my $data = $t->json_body($res);
    ok($data->{id},   "has id");
    is($data->{name}, "Cherry", "correct name");
    
    # Verify it's in the list now
    my $list_res = $t->get("/api/items");
    my $list     = $t->json_body($list_res);
    ok(grep { $_->{name} eq "Cherry" } @$list, "item in list");
};

subtest "POST /api/items — invalid" => sub {
    my $res  = $t->post("/api/items", { body => '{"bad":"data"}' });
    my $data = $t->json_body($res);
    ok($data->{error}, "validation error returned");
};

done_testing;
printf "\nDancer2-style app tests complete!\n";
```

---

## Step 338: Configuration Management

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package App::Config;

my %defaults = (
    database => {
        dsn      => "dbi:SQLite::memory:",
        user     => "",
        pass     => "",
        pool     => 5,
    },
    app => {
        name    => "MyApp",
        debug   => 0,
        port    => 3000,
        host    => "0.0.0.0",
        secret  => "change_this_secret",
    },
    session => {
        driver  => "memory",
        ttl     => 3600,
        secure  => 1,
    },
    cache => {
        driver  => "memory",
        ttl     => 300,
    },
    mail => {
        host    => "localhost",
        port    => 25,
        from    => 'noreply@localhost',
    },
);

my %config;

sub load_defaults { %config = _deep_merge(\%defaults, {}) }

sub load_env {
    # Override from environment variables: APP_DB_DSN, APP_PORT, etc.
    for my $key (keys %ENV) {
        next unless $key =~ /^APP_(.+)$/;
        my @parts = map { lc } split /_/, $1;
        my $section = shift @parts;
        my $name    = join("_", @parts);
        $config{$section}{$name} = $ENV{$key} if @parts;
    }
}

sub load_file {
    my ($class, $file) = @_;
    return unless -f $file;
    open my $fh, "<", $file or return;
    while (<$fh>) {
        next if /^\s*#/;
        next unless /^\s*(\w+)\.(\w+)\s*=\s*(.+?)\s*$/;
        $config{$1}{$2} = $3;
    }
    close $fh;
}

sub get {
    my ($class, $key) = @_;
    if ($key =~ /\./) {
        my ($section, $name) = split /\./, $key, 2;
        return $config{$section}{$name};
    }
    return $config{$key};
}

sub set {
    my ($class, $key, $value) = @_;
    if ($key =~ /\./) {
        my ($section, $name) = split /\./, $key, 2;
        $config{$section}{$name} = $value;
    } else {
        $config{$key} = $value;
    }
}

sub all { %config }

sub _deep_merge {
    my ($base, $override) = @_;
    my %result = %$base;
    for my $k (keys %$override) {
        if (ref($result{$k}) eq 'HASH' && ref($override->{$k}) eq 'HASH') {
            $result{$k} = _deep_merge($result{$k}, $override->{$k});
        } else {
            $result{$k} = $override->{$k};
        }
    }
    return \%result;
}
}

package main;

App::Config->load_defaults;

# Override some settings
App::Config->set("app.name",    "Blog Application");
App::Config->set("app.debug",   1);
App::Config->set("app.port",    8080);
App::Config->set("session.ttl", 7200);

printf "=== Configuration ===\n\n";
printf "App name:    %s\n",   App::Config->get("app.name");
printf "App port:    %d\n",   App::Config->get("app.port");
printf "Debug mode:  %s\n",   App::Config->get("app.debug") ? "on" : "off";
printf "Session TTL: %d\n",   App::Config->get("session.ttl");
printf "DB DSN:      %s\n",   App::Config->get("database.dsn");
printf "Mail from:   %s\n",   App::Config->get("mail.from");
printf "Secret:      %s***\n", substr(App::Config->get("app.secret"),0,6);
```

---

## Step 339: Error Handling and Logging

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(strftime);

{
package Logger;

my $log_level  = 2;  # 0=error, 1=warn, 2=info, 3=debug
my @log_buffer;
my $max_buffer = 100;

my %levels = (error => 0, warn => 1, info => 2, debug => 3);
my %colors = (error => "\e[31m", warn => "\e[33m", info => "\e[32m", debug => "\e[36m");
my $reset  = "\e[0m";

sub set_level { $log_level = $levels{$_[1]} // 2 }

sub _log {
    my ($level, $msg, @ctx) = @_;
    return if $levels{$level} > $log_level;
    
    my $ts   = strftime("%H:%M:%S", localtime);
    my $line = "[$ts] [" . uc($level) . "] $msg";
    $line   .= " {" . join(", ", map { "$_=$ctx[$_*2+1]" } 0..$#ctx/2) . "}"
        if @ctx;
    
    push @log_buffer, $line;
    shift @log_buffer if @log_buffer > $max_buffer;
    
    printf "%s%s%s\n", ($colors{$level}//""), $line, $reset;
}

sub error { _log("error", @_[1..$#_]) }
sub warn  { _log("warn",  @_[1..$#_]) }
sub info  { _log("info",  @_[1..$#_]) }
sub debug { _log("debug", @_[1..$#_]) }

sub recent { @log_buffer[-10..-1] }
}

{
package ErrorHandler;

my %error_pages = (
    400 => "Bad Request",
    401 => "Unauthorized",
    403 => "Forbidden",
    404 => "Not Found",
    429 => "Too Many Requests",
    500 => "Internal Server Error",
    503 => "Service Unavailable",
);

sub handle {
    my ($class, $status, $message, %ctx) = @_;
    
    Logger->error($message, status => $status, %ctx);
    
    my $title = $error_pages{$status} // "Error";
    
    return {
        status  => $status,
        headers => { "Content-Type" => "text/html" },
        body    => <<HTML,
<!DOCTYPE html><html><head><title>$status $title</title></head>
<body><h1>$status $title</h1><p>$message</p></body></html>
HTML
    };
}

sub json_error {
    my ($class, $status, $message) = @_;
    require JSON::PP;
    return {
        status  => $status,
        headers => { "Content-Type" => "application/json" },
        body    => JSON::PP->new->utf8->encode({ error => $message, code => $status }),
    };
}
}

package main;

printf "=== Error Handling and Logging ===\n\n";

Logger->set_level("debug");

Logger->info("Application starting", version => "1.0.0", env => "development");
Logger->debug("Config loaded", db => "sqlite");
Logger->info("Server listening", port => 3000);

# Simulate request handling
Logger->info("Request received", method => "GET", path => "/api/data");
Logger->debug("DB query executed", table => "articles", rows => 5);
Logger->warn("Slow query detected", query_ms => 450);

# Error handling
my $err_res = ErrorHandler->handle(404, "The page you requested was not found");
printf "\nError response: %d\n", $err_res->{status};

my $json_err = ErrorHandler->json_error(400, "Invalid request parameter: age must be numeric");
printf "JSON error:     %s\n", $json_err->{body};

Logger->error("Unhandled exception", exception => "Connection refused", db_host => "db.example.com");
```

---

## Step 340: Capstone — Dancer2-Style REST API

```perl
#!/usr/bin/perl
# dancer2_api.pl — Complete Dancer2-style REST API
use strict;
use warnings;
use DBI;
use JSON::PP;
use Digest::SHA qw(sha256_hex);

# Initialize everything
App::DB->connect;
App::Config->load_defaults;
App::Config->set("app.name", "TodoAPI");

printf "=== %s ===\n\n", App::Config->get("app.name");

# Create the API app
my $api = MiniApp->new;
$api->use_plugin("Plugin::Logger", level => "info");
$api->use_plugin("Plugin::RateLimit", max => 20, window => 60);

my $json = JSON::PP->new->utf8->canonical;

# ---- Todo Resource ----
my %todos;
my $todo_id = 1;

sub _create_todo {
    my (%args) = @_;
    my $id = $todo_id++;
    $todos{$id} = {
        id          => $id,
        title       => $args{title}       // "Untitled",
        completed   => $args{completed}   // 0,
        priority    => $args{priority}    // "medium",
        created_at  => time(),
    };
    return $todos{$id};
}

# Seed
_create_todo(title => "Buy groceries",   priority => "low");
_create_todo(title => "Write Perl code", priority => "high");
_create_todo(title => "Read book",       priority => "medium");
_create_todo(title => "Exercise",        priority => "high");

# Routes
$api->get("/api/todos", sub {
    my $req    = shift;
    my @items  = values %todos;
    
    # Filter by completed
    if (defined $req->{params}{completed}) {
        my $c = $req->{params}{completed};
        @items = grep { $_->{completed} == $c } @items;
    }
    
    # Filter by priority
    if ($req->{params}{priority}) {
        @items = grep { $_->{priority} eq $req->{params}{priority} } @items;
    }
    
    @items = sort { $a->{id} <=> $b->{id} } @items;
    
    $json->encode({
        items => \@items,
        count => scalar @items,
    })
});

$api->get("/api/todos/:id", sub {
    my $req = shift;
    (my $path = $req->{path}) =~ m{/(\d+)$};
    my $id = $1;
    return $todos{$id} ? $json->encode($todos{$id}) : $json->encode({ error => "Not found" });
});

$api->post("/api/todos", sub {
    my $req  = shift;
    my $data = eval { $json->decode($req->{body}//"{}") };
    return $json->encode({ error => "Invalid JSON" }) if $@;
    return $json->encode({ error => "title required" }) unless $data->{title};
    
    my $todo = _create_todo(%$data);
    return $json->encode($todo);
});

$api->put("/api/todos/:id", sub {
    my $req  = shift;
    (my $path = $req->{path}) =~ m{/(\d+)$};
    my $id   = $1;
    return $json->encode({ error => "Not found" }) unless $todos{$id};
    
    my $data = eval { $json->decode($req->{body}//"{}") } // {};
    
    $todos{$id}{title}     = $data->{title}     if defined $data->{title};
    $todos{$id}{completed} = $data->{completed} if defined $data->{completed};
    $todos{$id}{priority}  = $data->{priority}  if defined $data->{priority};
    $todos{$id}{updated_at} = time();
    
    return $json->encode($todos{$id});
});

$api->del("/api/todos/:id", sub {
    my $req = shift;
    (my $path = $req->{path}) =~ m{/(\d+)$};
    my $id  = $1;
    return $json->encode({ error => "Not found" }) unless $todos{$id};
    delete $todos{$id};
    return $json->encode({ deleted => \1, id => $id+0 });
});

$api->get("/api/stats", sub {
    my @all       = values %todos;
    my $completed = grep { $_->{completed} } @all;
    return $json->encode({
        total     => scalar @all,
        completed => $completed,
        pending   => scalar(@all) - $completed,
        by_priority => {
            high   => scalar(grep { $_->{priority} eq "high" } @all),
            medium => scalar(grep { $_->{priority} eq "medium" } @all),
            low    => scalar(grep { $_->{priority} eq "low" } @all),
        },
    });
});

# Integration tests
printf "Running API tests:\n\n";

sub test_req {
    my ($method, $path, $body, $params) = @_;
    my $res  = $api->dispatch($method, $path,
        body => $body, params => $params//{}, ip => "127.0.0.1");
    my $data = eval { $json->decode($res->{body}) };
    return ($res->{status}, $data);
}

# GET all
my ($s, $d) = test_req("GET", "/api/todos", undef);
printf "GET /todos: %d — count=%d\n", $s, $d->{count};

# Filter
($s, $d) = test_req("GET", "/api/todos", undef, { priority => "high" });
printf "GET /todos?priority=high: count=%d\n", $d->{count};

# Get one
($s, $d) = test_req("GET", "/api/todos/1", undef);
printf "GET /todos/1: %s\n", $d->{title};

# Create
($s, $d) = test_req("POST", "/api/todos", '{"title":"New task","priority":"high"}');
printf "POST /todos: created id=%d title=%s\n", $d->{id}, $d->{title};
my $new_id = $d->{id};

# Update
($s, $d) = test_req("PUT", "/api/todos/$new_id", '{"completed":1}');
printf "PUT /todos/$new_id: completed=%d\n", $d->{completed};

# Delete
($s, $d) = test_req("DELETE", "/api/todos/$new_id", undef);
printf "DELETE /todos/$new_id: deleted=%s\n", $d->{deleted} ? "yes" : "no";

# Stats
($s, $d) = test_req("GET", "/api/stats", undef);
printf "\nStats: total=%d completed=%d pending=%d\n",
    $d->{total}, $d->{completed}, $d->{pending};
printf "  By priority: high=%d medium=%d low=%d\n",
    $d->{by_priority}{high}, $d->{by_priority}{medium}, $d->{by_priority}{low};

printf "\nDancer2-style REST API complete!\n";
```

---

## สรุป Part 34 — Dancer2 Web Framework

ใน Part นี้คุณได้เรียนรู้:

### Framework Patterns
- **DSL Routing** — get/post/put/del/any routes
- **Middleware** — Logger, CORS, RateLimit, Auth, Compress
- **Database Integration** — Repository + Controller patterns
- **JSON API** — Structured responses, pagination metadata
- **JWT Authentication** — Token generation, validation, role-based access
- **Plugin System** — Pluggable before/after hooks
- **Testing** — Test::Dancer2-style request testing
- **Configuration** — Multi-source config (defaults/env/file)
- **Error Handling** — Structured error pages and JSON errors
- **Capstone** — Full CRUD REST API for Todo list

**ถัดไป: [Part 35 — Mojolicious Basics](part_35.md)**
