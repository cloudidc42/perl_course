# Part 18: Advanced CGI และ Web Applications
## Steps 171-180: การพัฒนาเว็บขั้นสูงด้วย CGI

---

## Step 171: MVC Pattern ใน CGI

```perl
#!/usr/bin/perl
# app.pl — MVC Web App
use strict;
use warnings;
use CGI;
use DBI;

my $cgi = CGI->new;
print $cgi->header(-type => 'text/html', -charset => 'utf-8');

# =====================
# Model
# =====================

package Model::Todo;

my $DB;

sub connect_db {
    $DB = DBI->connect("dbi:SQLite:dbname=/tmp/todo.db","","",
        { RaiseError => 1, AutoCommit => 1 });
    $DB->do(<<'SQL');
CREATE TABLE IF NOT EXISTS todos (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    title      TEXT NOT NULL,
    done       INTEGER DEFAULT 0,
    priority   TEXT DEFAULT 'normal',
    created_at DATETIME DEFAULT (datetime('now'))
)
SQL
}

sub all {
    return @{$DB->selectall_arrayref(
        "SELECT * FROM todos ORDER BY done, priority DESC, created_at DESC",
        { Slice => {} }
    )};
}

sub find {
    my $id = shift;
    return $DB->selectrow_hashref("SELECT * FROM todos WHERE id = ?", undef, $id);
}

sub create {
    my (%data) = @_;
    $DB->do("INSERT INTO todos (title, priority) VALUES (?, ?)",
        undef, $data{title}, $data{priority} // 'normal');
    return $DB->last_insert_id;
}

sub update {
    my ($id, %data) = @_;
    $DB->do("UPDATE todos SET title=?, done=?, priority=? WHERE id=?",
        undef, $data{title}, $data{done}//'', $data{priority}//'normal', $id);
}

sub toggle {
    my $id = shift;
    $DB->do("UPDATE todos SET done = NOT done WHERE id = ?", undef, $id);
}

sub delete_todo {
    my $id = shift;
    $DB->do("DELETE FROM todos WHERE id = ?", undef, $id);
}

sub stats {
    return {
        total  => ($DB->selectrow_array("SELECT COUNT(*) FROM todos"))[0],
        done   => ($DB->selectrow_array("SELECT COUNT(*) FROM todos WHERE done=1"))[0],
        active => ($DB->selectrow_array("SELECT COUNT(*) FROM todos WHERE done=0"))[0],
    };
}

# =====================
# View
# =====================

package View;

sub esc {
    my $s = shift // "";
    $s =~ s/&/&amp;/g;
    $s =~ s/</&lt;/g;
    $s =~ s/>/&gt;/g;
    $s =~ s/"/&quot;/g;
    $s;
}

sub layout {
    my ($title, $content) = @_;
    return <<HTML;
<!DOCTYPE html>
<html lang="th">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>$title</title>
<style>
* { box-sizing: border-box; }
body { font-family: -apple-system, sans-serif; max-width: 600px; margin: 30px auto; padding: 0 20px; color: #333; }
h1 { color: #2c5aa0; }
.card { background: #fff; border: 1px solid #ddd; border-radius: 8px; padding: 15px; margin: 10px 0; }
.todo-item { display: flex; align-items: center; gap: 10px; padding: 10px; border-bottom: 1px solid #eee; }
.todo-item:last-child { border-bottom: none; }
.done-title { text-decoration: line-through; color: #aaa; }
.priority-high   { border-left: 4px solid #e74c3c; }
.priority-normal { border-left: 4px solid #3498db; }
.priority-low    { border-left: 4px solid #95a5a6; }
input[type=text] { padding: 8px 12px; border: 1px solid #ddd; border-radius: 4px; font-size: 1em; }
button, .btn { padding: 8px 16px; border: none; border-radius: 4px; cursor: pointer; font-size: 0.9em; }
.btn-primary { background: #2c5aa0; color: white; }
.btn-success { background: #27ae60; color: white; }
.btn-danger  { background: #e74c3c; color: white; }
.stats { display: flex; gap: 20px; margin: 15px 0; }
.stat { text-align: center; padding: 10px; background: #f8f9fa; border-radius: 4px; flex: 1; }
.stat-num { font-size: 2em; font-weight: bold; color: #2c5aa0; }
</style>
</head>
<body>
$content
</body>
</html>
HTML
}

sub render_list {
    my (@todos) = @_;
    my $stats = Model::Todo::stats();
    
    my $stats_html = <<HTML;
<div class="stats">
  <div class="stat"><div class="stat-num">$stats->{total}</div><div>ทั้งหมด</div></div>
  <div class="stat"><div class="stat-num">$stats->{active}</div><div>ยังทำ</div></div>
  <div class="stat"><div class="stat-num">$stats->{done}</div><div>เสร็จแล้ว</div></div>
</div>
HTML
    
    my $list = join "", map {
        my $t = $_;
        my $done_class = $t->{done} ? "done-title" : "";
        my $pri = esc($t->{priority});
        <<HTML;
<div class="todo-item priority-$pri">
  <form method="post" style="margin:0">
    <input type="hidden" name="action" value="toggle">
    <input type="hidden" name="id" value="$t->{id}">
    <input type="checkbox" onchange="this.form.submit()" ${\($t->{done} ? 'checked' : '')}>
  </form>
  <span class="$done_class" style="flex:1">@{[esc($t->{title})]}</span>
  <small style="color:#999">$pri</small>
  <form method="post" style="margin:0">
    <input type="hidden" name="action" value="delete">
    <input type="hidden" name="id" value="$t->{id}">
    <button class="btn-danger" style="padding:4px 8px">×</button>
  </form>
</div>
HTML
    } @todos;
    
    return <<HTML;
<h1>📋 Todo List</h1>
$stats_html
<div class="card">
<form method="post">
  <input type="hidden" name="action" value="create">
  <div style="display:flex;gap:10px;flex-wrap:wrap">
    <input type="text" name="title" placeholder="งานใหม่..." style="flex:1;min-width:200px" required>
    <select name="priority" style="padding:8px;border:1px solid #ddd;border-radius:4px">
      <option value="high">สูง</option>
      <option value="normal" selected>ปกติ</option>
      <option value="low">ต่ำ</option>
    </select>
    <button class="btn-primary">+ เพิ่ม</button>
  </div>
</form>
</div>
<div class="card">
$list
</div>
HTML
}

# =====================
# Controller
# =====================

package Controller;

sub dispatch {
    my $cgi = shift;
    Model::Todo::connect_db();
    
    my $action = $cgi->param('action') // 'list';
    my $id     = $cgi->param('id');
    
    if ($action eq 'create' && $cgi->request_method eq 'POST') {
        Model::Todo::create(
            title    => $cgi->param('title'),
            priority => $cgi->param('priority') // 'normal',
        );
    } elsif ($action eq 'toggle' && $id) {
        Model::Todo::toggle($id);
    } elsif ($action eq 'delete' && $id) {
        Model::Todo::delete_todo($id);
    }
    
    my @todos = Model::Todo::all();
    return View::layout("Todo App", View::render_list(@todos));
}

# =====================
# Run
# =====================

package main;

print Controller::dispatch($cgi);
```

---

## Step 172: REST API ด้วย CGI

```perl
#!/usr/bin/perl
# api.pl — Simple REST API
use strict;
use warnings;
use CGI;
use DBI;
use JSON::PP;

my $cgi    = CGI->new;
my $json   = JSON::PP->new->utf8->canonical;
my $method = $ENV{REQUEST_METHOD} // 'GET';
my $path   = $ENV{PATH_INFO} // '/';

# =====================
# Router
# =====================

my $DB = DBI->connect("dbi:SQLite:dbname=/tmp/api.db","","",
    { RaiseError => 1, AutoCommit => 1 });

$DB->do(<<'SQL');
CREATE TABLE IF NOT EXISTS items (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    name        TEXT NOT NULL,
    description TEXT,
    price       REAL DEFAULT 0,
    stock       INTEGER DEFAULT 0,
    created_at  DATETIME DEFAULT (datetime('now'))
)
SQL

# Seed
unless (($DB->selectrow_array("SELECT COUNT(*) FROM items"))[0]) {
    $DB->do("INSERT INTO items (name,description,price,stock) VALUES
        ('Perl Book','Learn Perl Programming',49.99,100),
        ('Python Book','Learn Python',44.99,150),
        ('CGI Scripts','Web development',29.99,50)");
}

# =====================
# Response helpers
# =====================

sub json_response {
    my ($status, $data) = @_;
    my $body = $json->encode($data);
    print "Status: $status\n";
    print "Content-Type: application/json\n";
    print "Access-Control-Allow-Origin: *\n";
    print "\n";
    print $body;
}

sub error_response {
    my ($status, $message) = @_;
    json_response($status, { error => $message, status => $status });
}

# =====================
# Read request body
# =====================

sub get_body {
    my $len = $ENV{CONTENT_LENGTH} // 0;
    return {} unless $len > 0;
    my $body;
    read STDIN, $body, $len;
    return eval { $json->decode($body) } // {};
}

# =====================
# Route handlers
# =====================

# GET /items
sub list_items {
    my $search = $cgi->param('q') // '';
    my $limit  = int($cgi->param('limit') // 20);
    my $offset = int($cgi->param('offset') // 0);
    
    $limit = 100 if $limit > 100;
    
    my $sql = "SELECT * FROM items";
    my @params;
    if ($search) {
        $sql .= " WHERE name LIKE ? OR description LIKE ?";
        push @params, "%$search%", "%$search%";
    }
    $sql .= " ORDER BY id LIMIT ? OFFSET ?";
    push @params, $limit, $offset;
    
    my $items = $DB->selectall_arrayref($sql, { Slice => {} }, @params);
    my ($total) = $DB->selectrow_array("SELECT COUNT(*) FROM items");
    
    json_response(200, {
        items  => $items,
        total  => $total,
        limit  => $limit,
        offset => $offset,
    });
}

# GET /items/:id
sub get_item {
    my $id = shift;
    my $item = $DB->selectrow_hashref("SELECT * FROM items WHERE id = ?", undef, $id);
    return error_response(404, "Item not found") unless $item;
    json_response(200, $item);
}

# POST /items
sub create_item {
    my $data = get_body();
    
    unless ($data->{name}) {
        return error_response(422, "name is required");
    }
    
    $DB->do("INSERT INTO items (name,description,price,stock) VALUES (?,?,?,?)",
        undef, $data->{name}, $data->{description}//'',
        $data->{price}//0, $data->{stock}//0);
    
    my $id = $DB->last_insert_id;
    my $item = $DB->selectrow_hashref("SELECT * FROM items WHERE id = ?", undef, $id);
    
    json_response(201, $item);
}

# PUT /items/:id
sub update_item {
    my ($id, $data) = @_;
    
    my $item = $DB->selectrow_hashref("SELECT * FROM items WHERE id = ?", undef, $id);
    return error_response(404, "Item not found") unless $item;
    
    $DB->do("UPDATE items SET name=?, description=?, price=?, stock=? WHERE id=?",
        undef,
        $data->{name}        // $item->{name},
        $data->{description} // $item->{description},
        $data->{price}       // $item->{price},
        $data->{stock}       // $item->{stock},
        $id);
    
    my $updated = $DB->selectrow_hashref("SELECT * FROM items WHERE id = ?", undef, $id);
    json_response(200, $updated);
}

# DELETE /items/:id
sub delete_item {
    my $id = shift;
    my $item = $DB->selectrow_hashref("SELECT * FROM items WHERE id = ?", undef, $id);
    return error_response(404, "Item not found") unless $item;
    
    $DB->do("DELETE FROM items WHERE id = ?", undef, $id);
    json_response(200, { message => "Deleted", id => int($id) });
}

# =====================
# Dispatch
# =====================

if ($path =~ m{^/items/(\d+)$}) {
    my $id = $1;
    if    ($method eq 'GET')    { get_item($id) }
    elsif ($method eq 'PUT')    { update_item($id, get_body()) }
    elsif ($method eq 'DELETE') { delete_item($id) }
    else  { error_response(405, "Method not allowed") }
} elsif ($path eq '/items' || $path eq '/items/') {
    if    ($method eq 'GET')  { list_items() }
    elsif ($method eq 'POST') { create_item() }
    else  { error_response(405, "Method not allowed") }
} else {
    error_response(404, "Not found: $path");
}

$DB->disconnect;
```

---

## Step 173: Template Engine

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Simple Template Engine
# =====================

package Template;

sub new {
    my ($class, %opts) = @_;
    return bless {
        base_dir   => $opts{base_dir} // 'templates',
        cache      => {},
        auto_escape => $opts{auto_escape} // 1,
    }, $class;
}

sub escape {
    my $s = shift // "";
    $s =~ s/&/&amp;/g;
    $s =~ s/</&lt;/g;
    $s =~ s/>/&gt;/g;
    $s =~ s/"/&quot;/g;
    return $s;
}

sub render_string {
    my ($self, $template, $vars) = @_;
    $vars //= {};
    
    my $output = $template;
    
    # {{= var }} — escaped output
    $output =~ s/\{\{=\s*(\w+)\s*\}\}/
        defined $vars->{$1} ? escape($vars->{$1}) : ""/ge;
    
    # {{! var }} — raw output (no escaping)
    $output =~ s/\{\{!\s*(\w+)\s*\}\}/
        defined $vars->{$1} ? $vars->{$1} : ""/ge;
    
    # {{#if condition}} ... {{/if}}
    $output =~ s/\{\{#if\s+(\w+)\s*\}\}(.*?)\{\{\/if\}\}/
        $vars->{$1} ? $2 : ""/gse;
    
    # {{#unless condition}} ... {{/unless}}
    $output =~ s/\{\{#unless\s+(\w+)\s*\}\}(.*?)\{\{\/unless\}\}/
        $vars->{$1} ? "" : $2/gse;
    
    # {{#each items}} ... {{/each}}
    $output =~ s/\{\{#each\s+(\w+)\s*\}\}(.*?)\{\{\/each\}\}/
        do {
            my ($key, $block) = ($1, $2);
            my $items = $vars->{$key} // [];
            my $result = "";
            for my $i (0..$#$items) {
                my $item_block = $block;
                my $item = $items->[$i];
                
                # Replace {{.}} with scalar item
                $item_block =~ s/\{\{\.\}\}/${\(ref $item ? "" : escape($item))}/g;
                
                # Replace {{key}} with hash values
                if (ref $item eq 'HASH') {
                    $item_block =~ s/\{\{=\s*(\w+)\s*\}\}/
                        defined $item->{$1} ? escape($item->{$1}) : ""/ge;
                }
                
                # {{@index}}
                $item_block =~ s/\{\{@index\}\}/$i/g;
                $result .= $item_block;
            }
            $result;
        }/gse;
    
    return $output;
}

sub render_file {
    my ($self, $file, $vars) = @_;
    my $path = "$self->{base_dir}/$file";
    
    unless (exists $self->{cache}{$file}) {
        open(my $fh, '<:utf8', $path) or die "Template not found: $path";
        local $/;
        $self->{cache}{$file} = <$fh>;
        close $fh;
    }
    
    return $self->render_string($self->{cache}{$file}, $vars);
}

package main;

# =====================
# Test templates
# =====================

my $tmpl = Template->new;

# Basic substitution
my $hello_tmpl = "Hello, {{= name }}! You have {{= count }} messages.";
print $tmpl->render_string($hello_tmpl, { name => "Alice", count => 5 }), "\n";

# XSS safety test
my $xss_tmpl = "Input: {{= input }}";
print $tmpl->render_string($xss_tmpl, { input => "<script>alert(1)</script>" }), "\n";

# Conditionals
my $cond_tmpl = <<'T';
{{#if logged_in}}
<p>Welcome back, {{= username }}!</p>
{{/if}}
{{#unless logged_in}}
<p>Please <a href="/login">login</a>.</p>
{{/unless}}
T

print $tmpl->render_string($cond_tmpl, { logged_in => 1, username => "Alice" });
print $tmpl->render_string($cond_tmpl, { logged_in => 0 });

# Loop
my $list_tmpl = <<'T';
<ul>
{{#each items}}
  <li>{{@index}}. {{= .}}</li>
{{/each}}
</ul>
T

print $tmpl->render_string($list_tmpl, {
    items => [qw(Apple Banana Cherry Date)]
});

# Loop with objects
my $table_tmpl = <<'T';
<table>
{{#each users}}
  <tr><td>{{= id}}</td><td>{{= name}}</td><td>{{= email}}</td></tr>
{{/each}}
</table>
T

print $tmpl->render_string($table_tmpl, {
    users => [
        { id => 1, name => "Alice", email => "alice\@example.com" },
        { id => 2, name => "Bob",   email => "bob\@example.com" },
    ]
});
```

---

## Step 174: Middleware Pattern

```perl
#!/usr/bin/perl
use strict;
use warnings;
use CGI;
use POSIX qw(strftime);

# =====================
# Request/Response objects
# =====================

package Request;

sub new {
    my ($class, $cgi) = @_;
    return bless {
        cgi     => $cgi,
        method  => $ENV{REQUEST_METHOD} // 'GET',
        path    => $ENV{PATH_INFO}      // '/',
        query   => {},
        body    => {},
        headers => {},
        attrs   => {},   # for middleware to store data
    }, $class;
}

sub method { $_[0]->{method} }
sub path   { $_[0]->{path} }
sub param  { $_[0]->{cgi}->param($_[1]) }
sub attr   { 
    my ($self, $key, $val) = @_;
    $self->{attrs}{$key} = $val if defined $val;
    return $self->{attrs}{$key};
}

package Response;

sub new {
    my ($class) = @_;
    return bless {
        status  => 200,
        headers => { 'Content-Type' => 'text/html; charset=utf-8' },
        body    => '',
    }, $class;
}

sub status  {
    my ($self, $val) = @_;
    $self->{status} = $val if defined $val;
    return $self->{status};
}

sub header  {
    my ($self, $key, $val) = @_;
    $self->{headers}{$key} = $val if defined $val;
    return $self->{headers}{$key};
}

sub body    {
    my ($self, $val) = @_;
    $self->{body} = $val if defined $val;
    return $self->{body};
}

sub json {
    my ($self, $data) = @_;
    require JSON::PP;
    $self->header('Content-Type', 'application/json');
    $self->body(JSON::PP->new->utf8->encode($data));
}

sub send {
    my $self = shift;
    printf "Status: %d\n", $self->{status};
    printf "%s: %s\n", $_, $self->{headers}{$_} for sort keys %{$self->{headers}};
    print "\n";
    print $self->{body};
}

# =====================
# Middleware stack
# =====================

package App;

sub new {
    my $class = shift;
    return bless {
        middlewares => [],
        routes      => {},
    }, $class;
}

sub use {
    my ($self, $mw) = @_;
    push @{$self->{middlewares}}, $mw;
}

sub route {
    my ($self, $method, $path, $handler) = @_;
    $self->{routes}{"$method:$path"} = $handler;
}

sub get  { my ($self, $path, $h) = @_; $self->route('GET',    $path, $h) }
sub post { my ($self, $path, $h) = @_; $self->route('POST',   $path, $h) }

sub handle {
    my ($self, $req, $res) = @_;
    
    my @stack = @{$self->{middlewares}};
    
    # Add route handler as last middleware
    push @stack, sub {
        my ($req, $res, $next) = @_;
        my $key = $req->method . ":" . $req->path;
        my $handler = $self->{routes}{$key};
        
        if ($handler) {
            $handler->($req, $res);
        } else {
            $res->status(404);
            $res->body("<h1>404 Not Found</h1>");
        }
    };
    
    my $i = 0;
    my $dispatch;
    $dispatch = sub {
        return if $i >= @stack;
        my $mw = $stack[$i++];
        $mw->($req, $res, $dispatch);
    };
    $dispatch->();
}

# =====================
# Middleware implementations
# =====================

sub logger_middleware {
    return sub {
        my ($req, $res, $next) = @_;
        my $start = time();
        $next->();
        printf STDERR "[%s] %s %s %d (%.3fs)\n",
            strftime("%H:%M:%S", localtime),
            $req->method, $req->path,
            $res->status, time() - $start;
    };
}

sub auth_middleware {
    my %protected = map { $_ => 1 } @_;
    return sub {
        my ($req, $res, $next) = @_;
        if ($protected{$req->path}) {
            my $token = $req->param('token') // "";
            unless ($token eq "secret_token") {
                $res->status(401);
                $res->json({ error => "Unauthorized" });
                return;
            }
            $req->attr('user', { username => 'admin', role => 'admin' });
        }
        $next->();
    };
}

sub cors_middleware {
    return sub {
        my ($req, $res, $next) = @_;
        $res->header('Access-Control-Allow-Origin', '*');
        $res->header('Access-Control-Allow-Methods', 'GET,POST,PUT,DELETE');
        $res->header('Access-Control-Allow-Headers', 'Content-Type,Authorization');
        $next->();
    };
}

# =====================
# Application setup
# =====================

package main;

my $cgi = CGI->new;
my $req = Request->new($cgi);
my $res = Response->new;

my $app = App->new;

$app->use(App::cors_middleware());
$app->use(App::logger_middleware());
$app->use(App::auth_middleware('/admin', '/api/secret'));

$app->get('/', sub {
    my ($req, $res) = @_;
    $res->body("<h1>Hello from Perl App!</h1><p>Path: " . $req->path . "</p>");
});

$app->get('/api/items', sub {
    my ($req, $res) = @_;
    $res->json({ items => ["item1", "item2"], total => 2 });
});

$app->get('/admin', sub {
    my ($req, $res) = @_;
    my $user = $req->attr('user') // {};
    $res->json({ message => "Admin area", user => $user });
});

$app->handle($req, $res);
$res->send;
```

---

## Step 175: Caching

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(strftime);
use Storable qw(freeze thaw);
use Digest::MD5 qw(md5_hex);

# =====================
# Memory Cache
# =====================

package Cache::Memory;

sub new {
    my ($class, %opts) = @_;
    return bless {
        store   => {},
        ttl     => $opts{ttl}      // 300,      # 5 minutes
        max     => $opts{max_size} // 1000,
        stats   => { hits => 0, misses => 0, evictions => 0 },
    }, $class;
}

sub set {
    my ($self, $key, $value, $ttl) = @_;
    $ttl //= $self->{ttl};
    
    # Evict if full
    if (keys %{$self->{store}} >= $self->{max}) {
        $self->_evict;
    }
    
    $self->{store}{$key} = {
        value   => $value,
        expires => time() + $ttl,
        hits    => 0,
    };
}

sub get {
    my ($self, $key) = @_;
    
    my $entry = $self->{store}{$key};
    unless ($entry) {
        $self->{stats}{misses}++;
        return undef;
    }
    
    if (time() > $entry->{expires}) {
        delete $self->{store}{$key};
        $self->{stats}{misses}++;
        return undef;
    }
    
    $entry->{hits}++;
    $self->{stats}{hits}++;
    return $entry->{value};
}

sub del     { delete $_[0]->{store}{$_[1]} }
sub exists  { exists $_[0]->{store}{$_[1]} && time() <= $_[0]->{store}{$_[1]}{expires} }
sub flush   { $_[0]->{store} = {} }
sub size    { scalar keys %{$_[0]->{store}} }

sub stats {
    my $self = shift;
    my $s = $self->{stats};
    my $total = $s->{hits} + $s->{misses};
    return {
        %$s,
        hit_rate => $total > 0 ? $s->{hits} / $total : 0,
    };
}

sub _evict {
    my $self = shift;
    # LRU-ish: remove expired first, then least accessed
    my $now = time();
    
    # Remove expired
    for my $key (keys %{$self->{store}}) {
        if ($now > $self->{store}{$key}{expires}) {
            delete $self->{store}{$key};
            $self->{stats}{evictions}++;
        }
    }
    
    # If still too many, remove least accessed
    if (keys %{$self->{store}} >= $self->{max}) {
        my @by_hits = sort { $self->{store}{$a}{hits} <=> $self->{store}{$b}{hits} }
                          keys %{$self->{store}};
        my $remove = int($self->{max} * 0.1);  # remove 10%
        for my $key (@by_hits[0..$remove-1]) {
            delete $self->{store}{$key};
            $self->{stats}{evictions}++;
        }
    }
}

# =====================
# File Cache
# =====================

package Cache::File;

sub new {
    my ($class, %opts) = @_;
    my $dir = $opts{dir} // '/tmp/cache';
    mkdir $dir unless -d $dir;
    return bless { dir => $dir, ttl => $opts{ttl} // 300 }, $class;
}

sub _path { "$_[0]->{dir}/" . md5_hex($_[1]) . ".cache" }

sub set {
    my ($self, $key, $value, $ttl) = @_;
    $ttl //= $self->{ttl};
    
    my $data = freeze({ value => $value, expires => time() + $ttl });
    open(my $fh, '>', $self->_path($key)) or return;
    print $fh $data;
    close $fh;
}

sub get {
    my ($self, $key) = @_;
    my $path = $self->_path($key);
    return undef unless -e $path;
    
    open(my $fh, '<', $path) or return undef;
    local $/;
    my $data = thaw(<$fh>);
    close $fh;
    
    return undef unless $data && time() <= $data->{expires};
    return $data->{value};
}

sub del   { unlink $_[0]->_path($_[1]) }

# =====================
# Cache decorator
# =====================

package main;

# Memoize with cache
sub cached {
    my ($cache, $fn, $key_fn) = @_;
    $key_fn //= sub { md5_hex(join("\0", @_)) };
    
    return sub {
        my @args = @_;
        my $key = $key_fn->(@args);
        
        my $cached = $cache->get($key);
        return $cached if defined $cached;
        
        my $result = $fn->(@args);
        $cache->set($key, $result);
        return $result;
    };
}

my $cache = Cache::Memory->new(ttl => 60);

# Expensive database query simulation
my $fetch_user = cached($cache, sub {
    my $id = shift;
    print "  [DB] Fetching user $id...\n";
    sleep(0);   # simulate delay
    return { id => $id, name => "User $id", email => "user$id\@example.com" };
});

print "First fetch:\n";
my $u1 = $fetch_user->(1);
printf "User: %s\n", $u1->{name};

print "\nSecond fetch (should be cached):\n";
my $u2 = $fetch_user->(1);
printf "User: %s\n", $u2->{name};

my $stats = $cache->stats;
printf "\nCache stats: hits=%d, misses=%d, rate=%.0f%%\n",
    $stats->{hits}, $stats->{misses}, $stats->{hit_rate} * 100;
```

---

## Step 176: Email Sending

```perl
#!/usr/bin/perl
use strict;
use warnings;
use MIME::Lite;   # หรือ Email::MIME + Email::Sender
use MIME::Base64 qw(encode_base64);

# =====================
# MIME::Lite — simple email
# =====================

sub send_simple_email {
    my (%opts) = @_;
    
    my $msg = MIME::Lite->new(
        From    => $opts{from}    // 'noreply@example.com',
        To      => $opts{to},
        Subject => $opts{subject} // 'No Subject',
        Type    => 'text/plain; charset=utf-8',
        Data    => $opts{body}    // '',
    );
    
    # Add CC/BCC
    $msg->add('Cc',  $opts{cc})  if $opts{cc};
    $msg->add('Bcc', $opts{bcc}) if $opts{bcc};
    
    # Send via SMTP or sendmail
    if ($opts{smtp}) {
        $msg->send('smtp', $opts{smtp},
            AuthUser => $opts{smtp_user},
            AuthPass => $opts{smtp_pass},
        );
    } else {
        # Simulate print
        print $msg->as_string;
    }
}

# =====================
# HTML email with attachment
# =====================

sub send_html_email {
    my (%opts) = @_;
    
    my $msg = MIME::Lite->new(
        From    => $opts{from} // 'noreply@example.com',
        To      => $opts{to},
        Subject => $opts{subject},
        Type    => 'multipart/mixed',
    );
    
    # HTML body
    $msg->attach(
        Type     => 'text/html; charset=utf-8',
        Data     => $opts{html_body},
    );
    
    # Plain text fallback
    if ($opts{text_body}) {
        $msg->attach(
            Type => 'text/plain; charset=utf-8',
            Data => $opts{text_body},
        );
    }
    
    # Attachments
    for my $att (@{$opts{attachments} // []}) {
        $msg->attach(
            Type        => $att->{type} // 'application/octet-stream',
            FileName    => $att->{name},
            Path        => $att->{path},
            Disposition => 'attachment',
        );
    }
    
    return $msg;
}

# =====================
# Email templates
# =====================

sub welcome_email {
    my (%user) = @_;
    
    my $html = <<HTML;
<!DOCTYPE html>
<html>
<body style="font-family:sans-serif;max-width:600px;margin:0 auto">
<h1 style="color:#2c5aa0">ยินดีต้อนรับ!</h1>
<p>สวัสดี <strong>$user{name}</strong>,</p>
<p>บัญชีของคุณได้ถูกสร้างเรียบร้อยแล้ว</p>
<p><a href="https://example.com/verify?token=$user{token}" 
   style="background:#2c5aa0;color:white;padding:10px 20px;text-decoration:none;border-radius:4px">
   ยืนยันอีเมล
</a></p>
<p>ขอบคุณ,<br>ทีมงาน</p>
</body>
</html>
HTML
    
    return (
        to      => $user{email},
        subject => "ยินดีต้อนรับ $user{name}!",
        html_body => $html,
        text_body => "สวัสดี $user{name},\nบัญชีของคุณถูกสร้างแล้ว\nhttp://example.com/verify?token=$user{token}",
    );
}

# Demo
my $msg = send_html_email(
    welcome_email(
        name  => "Alice",
        email => "alice\@example.com",
        token => "abc123xyz456",
    )
);

print "Email message:\n";
print "-" x 60, "\n";
print substr($msg->as_string, 0, 500), "\n...\n";
```

---

## Step 177: Web Scraping

```perl
#!/usr/bin/perl
use strict;
use warnings;
use LWP::UserAgent;
use HTML::Parser;
use HTML::TreeBuilder;

# =====================
# LWP::UserAgent — HTTP client
# =====================

my $ua = LWP::UserAgent->new(
    agent   => 'PerlBot/1.0',
    timeout => 30,
);

# GET request
sub http_get {
    my ($url, %opts) = @_;
    
    my $req = HTTP::Request->new('GET', $url);
    $req->header('Accept-Language' => 'th,en;q=0.9');
    $req->header('Accept-Encoding' => 'gzip, deflate');
    $req->header($_, $opts{headers}{$_}) for keys %{$opts{headers} // {}};
    
    my $res = $ua->request($req);
    
    return {
        status   => $res->code,
        ok       => $res->is_success,
        body     => $res->decoded_content,
        headers  => { map { $_ => $res->header($_) } $res->header_field_names },
    };
}

# POST request
sub http_post {
    my ($url, $data, %opts) = @_;
    
    my $res;
    if (ref $data eq 'HASH' && !$opts{json}) {
        # Form POST
        $res = $ua->post($url, $data);
    } else {
        # JSON POST
        require JSON::PP;
        my $req = HTTP::Request->new('POST', $url);
        $req->content_type('application/json');
        $req->content(JSON::PP->new->encode($data));
        $res = $ua->request($req);
    }
    
    return {
        status => $res->code,
        ok     => $res->is_success,
        body   => $res->decoded_content,
    };
}

# =====================
# HTML Parser
# =====================

sub extract_links {
    my $html = shift;
    my @links;
    
    my $p = HTML::Parser->new(
        start_h => [sub {
            my ($tag, $attr) = @_;
            if ($tag eq 'a' && $attr->{href}) {
                push @links, {
                    href  => $attr->{href},
                    title => $attr->{title} // "",
                };
            }
        }, "tag, attr"]
    );
    
    $p->parse($html);
    $p->eof;
    
    return @links;
}

sub extract_text {
    my $html = shift;
    $html =~ s/<script[^>]*>.*?<\/script>//gsi;
    $html =~ s/<style[^>]*>.*?<\/style>//gsi;
    $html =~ s/<[^>]+>//g;
    $html =~ s/&nbsp;/ /g;
    $html =~ s/&lt;/</g;
    $html =~ s/&gt;/>/g;
    $html =~ s/&amp;/&/g;
    $html =~ s/\s+/ /g;
    $html =~ s/^\s+|\s+$//g;
    return $html;
}

# =====================
# HTML::TreeBuilder
# =====================

sub parse_table {
    my $html = shift;
    
    my $tree = HTML::TreeBuilder->new_from_content($html);
    my @tables;
    
    for my $table ($tree->look_down(_tag => 'table')) {
        my @rows;
        for my $tr ($table->look_down(_tag => 'tr')) {
            my @cells = map { $_->as_trimmed_text } 
                        $tr->look_down(_tag => qr/^t[dh]$/);
            push @rows, \@cells if @cells;
        }
        push @tables, \@rows if @rows;
    }
    
    $tree->delete;
    return @tables;
}

# =====================
# Demo (using example HTML)
# =====================

my $sample_html = <<'HTML';
<html>
<body>
<h1>Test Page</h1>
<p>Hello <a href="http://example.com" title="Example">World</a></p>
<a href="/about">About</a>
<a href="/contact">Contact</a>

<table>
  <tr><th>Name</th><th>Age</th><th>City</th></tr>
  <tr><td>Alice</td><td>28</td><td>Bangkok</td></tr>
  <tr><td>Bob</td><td>35</td><td>Chiang Mai</td></tr>
</table>
</body>
</html>
HTML

# Extract links
my @links = extract_links($sample_html);
print "Links found:\n";
printf "  %s (%s)\n", $_->{href}, $_->{title} for @links;

# Extract text
my $text = extract_text($sample_html);
print "\nText: $text\n";

# Parse table
my @tables = parse_table($sample_html);
print "\nTables:\n";
for my $table (@tables) {
    for my $row (@$table) {
        print "  " . join(" | ", @$row) . "\n";
    }
    print "\n";
}
```

---

## Step 178: Rate Limiting

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(floor);

# =====================
# Token Bucket rate limiter
# =====================

package RateLimit::TokenBucket;

sub new {
    my ($class, %opts) = @_;
    return bless {
        rate     => $opts{rate}     // 10,   # tokens per second
        capacity => $opts{capacity} // 10,   # max burst
        tokens   => $opts{capacity} // 10,
        last_ts  => time(),
    }, $class;
}

sub allow {
    my ($self, $tokens) = @_;
    $tokens //= 1;
    
    $self->_refill;
    
    if ($self->{tokens} >= $tokens) {
        $self->{tokens} -= $tokens;
        return 1;
    }
    
    return 0;
}

sub _refill {
    my $self = shift;
    my $now = time();
    my $elapsed = $now - $self->{last_ts};
    
    $self->{tokens} += $elapsed * $self->{rate};
    $self->{tokens}  = $self->{capacity} if $self->{tokens} > $self->{capacity};
    $self->{last_ts} = $now;
}

sub tokens { $_[0]->_refill; $_[0]->{tokens} }

# =====================
# Fixed Window rate limiter
# =====================

package RateLimit::FixedWindow;

sub new {
    my ($class, %opts) = @_;
    return bless {
        limit    => $opts{limit}  // 100,   # requests per window
        window   => $opts{window} // 60,    # window size in seconds
        counts   => {},
    }, $class;
}

sub allow {
    my ($self, $key) = @_;
    $key //= 'global';
    
    my $now = time();
    my $window_key = $key . ':' . floor($now / $self->{window});
    
    $self->{counts}{$window_key} //= 0;
    $self->{counts}{$window_key}++;
    
    # Cleanup old windows
    my $cutoff = floor($now / $self->{window}) - 2;
    delete $self->{counts}{$_}
        for grep { /:\d+$/ && $_ =~ /:(\d+)$/ && $1 < $cutoff }
              keys %{$self->{counts}};
    
    return $self->{counts}{$window_key} <= $self->{limit};
}

sub count {
    my ($self, $key) = @_;
    $key //= 'global';
    my $window_key = $key . ':' . floor(time() / $self->{window});
    return $self->{counts}{$window_key} // 0;
}

# =====================
# IP-based rate limiter for CGI
# =====================

package RateLimit::CGI;

sub new {
    my ($class, %opts) = @_;
    return bless {
        limiter => RateLimit::FixedWindow->new(%opts),
    }, $class;
}

sub check {
    my ($self, $ip) = @_;
    $ip //= $ENV{REMOTE_ADDR} // '0.0.0.0';
    
    unless ($self->{limiter}->allow($ip)) {
        print "Status: 429 Too Many Requests\n";
        print "Content-Type: text/plain\n";
        print "Retry-After: 60\n";
        print "\n";
        print "Rate limit exceeded. Please wait.\n";
        return 0;
    }
    
    return 1;
}

# =====================
# Demo
# =====================

package main;

my $bucket = RateLimit::TokenBucket->new(rate => 2, capacity => 5);

print "Token bucket demo:\n";
for my $i (1..8) {
    my $allowed = $bucket->allow;
    printf "Request %d: %s (tokens left: %.1f)\n",
        $i, $allowed ? "ALLOWED" : "DENIED", $bucket->tokens;
}

my $window = RateLimit::FixedWindow->new(limit => 5, window => 10);

print "\nFixed window demo:\n";
for my $i (1..8) {
    my $allowed = $window->allow("user1");
    printf "Request %d: %s (count: %d/5)\n",
        $i, $allowed ? "ALLOWED" : "DENIED", $window->count("user1");
}
```

---

## Step 179: WebSocket Simulation

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# WebSocket ด้วย Perl
# Note: WebSocket จริงต้องใช้ AnyEvent::WebSocket::Server
#       หรือ Mojo::WebSocket
# นี่คือ simulation เพื่อเข้าใจ concept
# =====================

# =====================
# Server-Sent Events (SSE)
# =====================

sub sse_stream {
    my ($cgi) = @_;
    
    print "Content-Type: text/event-stream\n";
    print "Cache-Control: no-cache\n";
    print "Connection: keep-alive\n";
    print "\n";
    
    # Disable output buffering
    $| = 1;
    
    my $event_id = 0;
    while (1) {
        $event_id++;
        
        my $data = {
            time => time(),
            id   => $event_id,
            cpu  => int(rand 100),
            mem  => int(rand 100),
        };
        
        require JSON::PP;
        my $json = JSON::PP->new->encode($data);
        
        printf "id: %d\n", $event_id;
        printf "event: status\n";
        printf "data: %s\n\n", $json;
        
        sleep 1;
        
        last if $event_id >= 5;   # limit for demo
    }
    
    print "event: close\ndata: {}\n\n";
}

# =====================
# Long polling
# =====================

package LongPoll;

my @message_queue;

sub add_message {
    my ($channel, $data) = @_;
    push @message_queue, {
        id      => time() . rand(),
        channel => $channel,
        data    => $data,
        ts      => time(),
    };
    # Keep only last 100
    @message_queue = @message_queue[-100..-1] if @message_queue > 100;
}

sub poll {
    my ($channel, $since, $timeout) = @_;
    $timeout //= 30;
    
    my $end = time() + $timeout;
    
    while (time() < $end) {
        my @new = grep {
            $_->{channel} eq $channel && $_->{ts} > $since
        } @message_queue;
        
        return @new if @new;
        
        # In real app: sleep briefly and check again
        # Here we just return empty after one check
        last;
    }
    
    return ();
}

# =====================
# Chat server simulation
# =====================

package main;

my @chat_history;
my %online_users;

sub broadcast {
    my (%msg) = @_;
    $msg{ts} = time();
    push @chat_history, \%msg;
    @chat_history = @chat_history[-50..-1] if @chat_history > 50;
}

sub user_join {
    my ($user) = @_;
    $online_users{$user} = time();
    broadcast(type => "join", user => $user, message => "$user joined");
}

sub user_leave {
    my ($user) = @_;
    delete $online_users{$user};
    broadcast(type => "leave", user => $user, message => "$user left");
}

sub send_message {
    my ($user, $message) = @_;
    broadcast(type => "message", user => $user, message => $message);
}

# Demo
user_join("Alice");
user_join("Bob");
send_message("Alice", "Hello Bob!");
send_message("Bob", "Hi Alice! How are you?");
send_message("Alice", "I'm learning Perl CGI");
send_message("Bob", "Cool! Me too");
user_leave("Alice");

print "Chat history:\n";
for my $msg (@chat_history) {
    my $prefix = $msg->{type} eq 'message' ? "$msg->{user}: " : "*** ";
    printf "[%s] %s%s\n", 
        scalar localtime $msg->{ts},
        $prefix, $msg->{message};
}

printf "\nOnline users: %s\n", join(", ", sort keys %online_users);
```

---

## Step 180: โปรแกรมสรุป — Full Web Application

```perl
#!/usr/bin/perl
#
# webapp.pl — Complete Web Application Framework
#

use strict;
use warnings;
use CGI;
use DBI;
use JSON::PP;
use Digest::SHA qw(sha256_hex);
use POSIX qw(strftime);

my $cgi = CGI->new;

# =====================
# Configuration
# =====================

my %CONFIG = (
    db_path    => '/tmp/webapp.db',
    session_ttl => 3600,
    max_upload  => 5 * 1024 * 1024,
    app_name    => 'Perl WebApp',
);

# =====================
# Database setup
# =====================

my $DB = DBI->connect("dbi:SQLite:dbname=$CONFIG{db_path}","","",
    { RaiseError => 1, AutoCommit => 1, sqlite_unicode => 1 });

for my $sql (
    "CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE NOT NULL,
        password_hash TEXT NOT NULL,
        email TEXT,
        role TEXT DEFAULT 'user',
        created_at DATETIME DEFAULT (datetime('now'))
    )",
    "CREATE TABLE IF NOT EXISTS sessions (
        id TEXT PRIMARY KEY,
        user_id INTEGER,
        data TEXT DEFAULT '{}',
        expires_at INTEGER,
        created_at DATETIME DEFAULT (datetime('now'))
    )",
    "CREATE TABLE IF NOT EXISTS posts (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        title TEXT NOT NULL,
        content TEXT,
        published INTEGER DEFAULT 0,
        created_at DATETIME DEFAULT (datetime('now'))
    )",
) {
    $DB->do($sql);
}

# =====================
# Auth helpers
# =====================

sub hash_password {
    my ($pwd, $salt) = @_;
    $salt //= join "", map { ('a'..'z','0'..'9')[rand 36] } 1..8;
    return (sha256_hex($salt . $pwd), $salt);
}

sub verify_password {
    my ($pwd, $hash, $salt) = @_;
    return sha256_hex($salt . $pwd) eq $hash;
}

sub get_session {
    my $sid = $cgi->cookie('session') // return undef;
    my $row = $DB->selectrow_hashref(
        "SELECT * FROM sessions WHERE id=? AND expires_at>?",
        undef, $sid, time()
    );
    return undef unless $row;
    return {
        id   => $sid,
        user => $DB->selectrow_hashref("SELECT * FROM users WHERE id=?", undef, $row->{user_id}),
        data => eval { JSON::PP->new->decode($row->{data}) } // {},
    };
}

sub create_session {
    my ($user_id) = @_;
    my $sid = sha256_hex(time() . $$ . rand());
    $DB->do("INSERT INTO sessions (id, user_id, expires_at) VALUES (?,?,?)",
        undef, $sid, $user_id, time() + $CONFIG{session_ttl});
    return $sid;
}

sub destroy_session {
    my $sid = shift;
    $DB->do("DELETE FROM sessions WHERE id=?", undef, $sid);
}

# =====================
# Router
# =====================

my $path   = $ENV{PATH_INFO} // '/';
my $method = $ENV{REQUEST_METHOD} // 'GET';
my $sess   = get_session();

my %headers;
my $status = 200;
my $body   = "";

sub redirect {
    my $url = shift;
    print "Status: 302\n";
    print "Location: $url\n";
    print "\n";
    exit;
}

sub require_auth {
    redirect("?page=login") unless $sess;
    return $sess;
}

# Handle actions
my $page = $cgi->param('page') // 'home';

if ($page eq 'register' && $method eq 'POST') {
    my $username = $cgi->param('username');
    my $password = $cgi->param('password');
    my $email    = $cgi->param('email');
    
    if ($username && $password) {
        my ($hash, $salt) = hash_password($password);
        eval {
            $DB->do("INSERT INTO users (username,password_hash,email) VALUES (?,?,?)",
                undef, $username, "$hash:$salt", $email);
        };
        if ($@) {
            $page = "register";
            $body = "<p class='error'>Username already taken</p>";
        } else {
            redirect("?page=login&registered=1");
        }
    }
}

if ($page eq 'login' && $method eq 'POST') {
    my $username = $cgi->param('username');
    my $password = $cgi->param('password');
    
    my $user = $DB->selectrow_hashref("SELECT * FROM users WHERE username=?", undef, $username);
    if ($user && $user->{password_hash} =~ /^([^:]+):(.+)$/) {
        if (verify_password($password, $1, $2)) {
            my $sid = create_session($user->{id});
            print "Status: 302\n";
            print "Set-Cookie: session=$sid; Path=/; HttpOnly\n";
            print "Location: ?\n\n";
            exit;
        }
    }
    $page = "login";
    $body = "<p class='error'>Invalid credentials</p>";
}

if ($page eq 'logout') {
    destroy_session($cgi->cookie('session')) if $sess;
    print "Status: 302\n";
    print "Set-Cookie: session=; Path=/; Expires=Thu, 01 Jan 1970 00:00:00 GMT\n";
    print "Location: ?\n\n";
    exit;
}

# Output
print "Status: $status\nContent-Type: text/html; charset=utf-8\n\n";

# Layout
print <<HTML;
<!DOCTYPE html>
<html lang="th">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>$CONFIG{app_name}</title>
<style>
* { box-sizing: border-box; }
body { font-family: -apple-system, sans-serif; margin: 0; background: #f5f5f5; }
nav { background: #2c5aa0; padding: 0 20px; display: flex; align-items: center; }
nav a { color: white; padding: 15px 10px; text-decoration: none; }
nav a:hover { background: rgba(255,255,255,.1); }
.nav-right { margin-left: auto; }
main { max-width: 900px; margin: 20px auto; padding: 0 20px; }
.card { background: white; padding: 20px; border-radius: 8px; margin: 15px 0; box-shadow: 0 2px 4px rgba(0,0,0,.08); }
.error { color: #e74c3c; }
.success { color: #27ae60; }
input[type=text], input[type=email], input[type=password], textarea {
  width: 100%; padding: 10px; border: 1px solid #ddd; border-radius: 4px; margin: 5px 0 15px; font-size: 1em;
}
button[type=submit] {
  background: #2c5aa0; color: white; padding: 10px 24px; border: none; border-radius: 4px; cursor: pointer; font-size: 1em;
}
</style>
</head>
<body>
<nav>
  <a href="?" style="font-size:1.2em;font-weight:bold">$CONFIG{app_name}</a>
  <a href="?page=home">หน้าแรก</a>
HTML

if ($sess) {
    printf "<span class='nav-right'><a href='?'>👤 %s</a> <a href='?page=logout'>ออก</a></span>\n",
        $sess->{user}{username};
} else {
    print "<span class='nav-right'><a href='?page=login'>เข้าสู่ระบบ</a> <a href='?page=register'>สมัคร</a></span>\n";
}

print "</nav>\n<main>\n";

# Page content
if ($page eq 'login') {
    print "<div class='card'><h2>เข้าสู่ระบบ</h2>$body";
    print "<p class='success'>สมัครสมาชิกสำเร็จ! กรุณาเข้าสู่ระบบ</p>" if $cgi->param('registered');
    print <<FORM;
<form method="post">
<input type="hidden" name="page" value="login">
<label>Username: <input type="text" name="username" required></label>
<label>Password: <input type="password" name="password" required></label>
<button type="submit">เข้าสู่ระบบ</button>
</form>
</div>
FORM

} elsif ($page eq 'register') {
    print "<div class='card'><h2>สมัครสมาชิก</h2>$body";
    print <<FORM;
<form method="post">
<input type="hidden" name="page" value="register">
<label>Username: <input type="text" name="username" required></label>
<label>Email: <input type="email" name="email"></label>
<label>Password: <input type="password" name="password" required></label>
<button type="submit">สมัครสมาชิก</button>
</form>
</div>
FORM

} else {
    # Home page
    print "<div class='card'><h2>ยินดีต้อนรับ</h2>";
    if ($sess) {
        printf "<p>สวัสดี <strong>%s</strong>!</p>\n", $sess->{user}{username};
        printf "<p>เข้าสู่ระบบในฐานะ: <strong>%s</strong></p>\n", $sess->{user}{role};
    } else {
        print "<p>กรุณา <a href='?page=login'>เข้าสู่ระบบ</a> หรือ <a href='?page=register'>สมัครสมาชิก</a></p>\n";
    }
    
    # User count
    my ($count) = $DB->selectrow_array("SELECT COUNT(*) FROM users");
    printf "<p>มีผู้ใช้ทั้งหมด: <strong>%d</strong> คน</p>\n", $count;
    print "</div>\n";
}

print "</main></body></html>\n";
$DB->disconnect;
```

---

## สรุป Part 18

ใน Part นี้คุณได้เรียนรู้:
- ✅ MVC Pattern ใน CGI
- ✅ REST API
- ✅ Template Engine
- ✅ Middleware Pattern
- ✅ Caching (Memory, File)
- ✅ Email sending
- ✅ Web scraping (LWP, HTML::Parser)
- ✅ Rate Limiting
- ✅ Server-Sent Events
- ✅ Full Web Application

**ถัดไป: [Part 19 — Database Programming](part_19.md)**
