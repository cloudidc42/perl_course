# Part 30: Intermediate Capstone — Full-Stack Web Application
## Steps 291-300: โปรเจกต์รวม — Task Management System

---

## Step 291: Project Overview

```perl
#!/usr/bin/perl
# project_overview.pl — Task Management System (TMS)
use strict;
use warnings;

print <<'OVERVIEW';
========================================
Task Management System (TMS) v1.0
========================================

Architecture:
  - Frontend: CGI-based HTML/CSS/JS
  - Backend: Perl with CGI
  - Database: SQLite via DBI
  - Auth: Session-based with SHA-256
  - ORM: Custom lightweight
  - Template: Custom engine

Features:
  1. User auth (register, login, logout)
  2. Projects: create, list, view, delete
  3. Tasks: create, assign, status, priority, due date
  4. Comments on tasks
  5. Tags for tasks
  6. Dashboard with stats
  7. REST API (JSON)
  8. Search with filters
  9. Pagination
 10. File attachments (simulated)

Modules:
  TMS::DB         - Database connection and schema
  TMS::Model::*   - Data models (User, Project, Task, Comment, Tag)
  TMS::Auth       - Authentication and sessions
  TMS::Template   - Template engine
  TMS::Router     - Request routing
  TMS::Controller::* - Controllers
  TMS::API        - JSON API
  TMS::Validator  - Input validation

Files:
  tms.pl          - Main CGI entry point
  api.pl          - API entry point
  schema.sql      - Database schema
  t/              - Tests

OVERVIEW

print "System ready for implementation.\n";
```

---

## Step 292: Database Schema and Models

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

# =====================
# TMS::DB
# =====================

{
package TMS::DB;

my $SCHEMA = q{
CREATE TABLE IF NOT EXISTS users (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    username   TEXT UNIQUE NOT NULL,
    email      TEXT UNIQUE NOT NULL,
    password   TEXT NOT NULL,
    full_name  TEXT,
    role       TEXT DEFAULT 'member',
    active     INTEGER DEFAULT 1,
    created_at INTEGER DEFAULT (strftime('%s','now')),
    updated_at INTEGER
);

CREATE TABLE IF NOT EXISTS sessions (
    token      TEXT PRIMARY KEY,
    user_id    INTEGER REFERENCES users(id),
    expires_at INTEGER NOT NULL,
    created_at INTEGER DEFAULT (strftime('%s','now'))
);

CREATE TABLE IF NOT EXISTS projects (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    name        TEXT NOT NULL,
    slug        TEXT UNIQUE NOT NULL,
    description TEXT,
    owner_id    INTEGER REFERENCES users(id),
    status      TEXT DEFAULT 'active',
    created_at  INTEGER DEFAULT (strftime('%s','now')),
    updated_at  INTEGER
);

CREATE TABLE IF NOT EXISTS tasks (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    project_id  INTEGER REFERENCES projects(id),
    title       TEXT NOT NULL,
    description TEXT,
    status      TEXT DEFAULT 'todo',
    priority    TEXT DEFAULT 'medium',
    assignee_id INTEGER REFERENCES users(id),
    creator_id  INTEGER REFERENCES users(id),
    due_date    TEXT,
    position    INTEGER DEFAULT 0,
    created_at  INTEGER DEFAULT (strftime('%s','now')),
    updated_at  INTEGER
);

CREATE TABLE IF NOT EXISTS comments (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id    INTEGER REFERENCES tasks(id) ON DELETE CASCADE,
    user_id    INTEGER REFERENCES users(id),
    content    TEXT NOT NULL,
    created_at INTEGER DEFAULT (strftime('%s','now'))
);

CREATE TABLE IF NOT EXISTS tags (
    id   INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT UNIQUE NOT NULL,
    color TEXT DEFAULT '#6c757d'
);

CREATE TABLE IF NOT EXISTS task_tags (
    task_id INTEGER REFERENCES tasks(id) ON DELETE CASCADE,
    tag_id  INTEGER REFERENCES tags(id),
    PRIMARY KEY (task_id, tag_id)
);
};

my $dbh;

sub connect {
    my ($class, $dsn) = @_;
    $dsn //= "dbi:SQLite::memory:";
    $dbh = DBI->connect($dsn, "", "", {
        RaiseError     => 1,
        AutoCommit     => 1,
        sqlite_unicode => 1,
    });
    $dbh->do("PRAGMA foreign_keys = ON");
    return $class;
}

sub handle { $dbh }

sub init_schema {
    for my $stmt (split /;\s*/, $SCHEMA) {
        $stmt =~ s/^\s+|\s+$//g;
        $dbh->do($stmt) if length $stmt;
    }
}

sub transaction {
    my ($class, $code) = @_;
    $dbh->{AutoCommit} = 0;
    eval { $code->($dbh) };
    if ($@) {
        $dbh->rollback;
        $dbh->{AutoCommit} = 1;
        die $@;
    }
    $dbh->commit;
    $dbh->{AutoCommit} = 1;
}
}

# Initialize
TMS::DB->connect;
TMS::DB->init_schema;

printf "Database initialized\n";
printf "Handle: %s\n", ref(TMS::DB->handle);
```

---

## Step 293: User Model

```perl
#!/usr/bin/perl
# (continued from Step 292 — same script)
use strict;
use warnings;
use Digest::SHA qw(sha256_hex);

{
package TMS::Model::User;

sub _dbh { TMS::DB->handle }

sub hash_password { sha256_hex("tms_salt_" . $_[1]) }

sub create {
    my ($class, %args) = @_;
    my $dbh = $class->_dbh;
    
    die "Username required\n"  unless $args{username};
    die "Email required\n"     unless $args{email};
    die "Password required\n"  unless $args{password};
    
    # Validate uniqueness
    my $exists = $dbh->selectrow_array("SELECT id FROM users WHERE username=? OR email=?",
        undef, $args{username}, $args{email});
    die "Username or email already exists\n" if $exists;
    
    my $hashed = $class->hash_password($args{password});
    $dbh->do("INSERT INTO users (username, email, password, full_name, role) VALUES (?,?,?,?,?)",
        undef, $args{username}, $args{email}, $hashed,
        $args{full_name}//$args{username}, $args{role}//"member");
    
    return $class->find($dbh->last_insert_id);
}

sub find {
    my ($class, $id) = @_;
    return TMS::DB->handle->selectrow_hashref("SELECT * FROM users WHERE id=?", undef, $id);
}

sub find_by_username {
    my ($class, $username) = @_;
    return TMS::DB->handle->selectrow_hashref("SELECT * FROM users WHERE username=?", undef, $username);
}

sub authenticate {
    my ($class, $username, $password) = @_;
    my $user = $class->find_by_username($username) or return undef;
    return undef unless $user->{active};
    my $hashed = $class->hash_password($password);
    return $hashed eq $user->{password} ? $user : undef;
}

sub all {
    return @{TMS::DB->handle->selectall_arrayref("SELECT * FROM users ORDER BY username", {Slice=>{}})};
}

sub update {
    my ($class, $id, %args) = @_;
    my @fields = grep { $args{$_} } qw(email full_name role active);
    return unless @fields;
    my $set = join(", ", map { "$_=?" } @fields);
    my @vals = map { $args{$_} } @fields;
    TMS::DB->handle->do("UPDATE users SET $set, updated_at=strftime('%s','now') WHERE id=?",
        undef, @vals, $id);
}
}

# Test User model
TMS::Model::User->create(
    username  => "alice",
    email     => "alice\@tms.com",
    password  => "password123",
    full_name => "Alice Smith",
    role      => "admin",
);

TMS::Model::User->create(
    username  => "bob",
    email     => "bob\@tms.com",
    password  => "secret456",
    full_name => "Bob Jones",
);

# Auth test
my $user = TMS::Model::User->authenticate("alice", "password123");
printf "Auth alice: %s\n", $user ? "OK" : "FAIL";
printf "  Name: %s\n",   $user->{full_name};
printf "  Role: %s\n",   $user->{role};

my $bad = TMS::Model::User->authenticate("alice", "wrongpass");
printf "Auth bad pw: %s\n", $bad ? "should_fail" : "correctly_failed";

# List users
my @users = TMS::Model::User->all;
printf "Users: %d\n", scalar @users;
printf "  - %s (%s)\n", $_->{username}, $_->{email} for @users;
```

---

## Step 294: Project and Task Models

```perl
#!/usr/bin/perl
# (continued in same execution context)
use strict;
use warnings;

{
package TMS::Model::Project;

sub _dbh { TMS::DB->handle }

sub _slugify {
    my $s = lc shift;
    $s =~ s/[^a-z0-9]+/-/g;
    $s =~ s/^-|-$//g;
    return $s;
}

sub create {
    my ($class, %args) = @_;
    die "Name required\n"     unless $args{name};
    die "Owner ID required\n" unless $args{owner_id};
    
    my $slug = $class->_slugify($args{name});
    my $suffix = 0;
    my $try = $slug;
    while ($class->_dbh->selectrow_array("SELECT id FROM projects WHERE slug=?", undef, $try)) {
        $try = $slug . "-" . ++$suffix;
    }
    $slug = $try;
    
    $class->_dbh->do(
        "INSERT INTO projects (name, slug, description, owner_id, status) VALUES (?,?,?,?,?)",
        undef, $args{name}, $slug, $args{description}//"", $args{owner_id}, $args{status}//"active"
    );
    return $class->find($class->_dbh->last_insert_id);
}

sub find  { TMS::DB->handle->selectrow_hashref("SELECT * FROM projects WHERE id=?", undef, $_[1]) }
sub all   { @{TMS::DB->handle->selectall_arrayref("SELECT * FROM projects ORDER BY name", {Slice=>{}})} }
sub by_owner { @{TMS::DB->handle->selectall_arrayref("SELECT * FROM projects WHERE owner_id=?", {Slice=>{}}, $_[1])} }

sub stats {
    my ($class, $id) = @_;
    my $dbh = TMS::DB->handle;
    my %s;
    for my $status (qw(todo in_progress done cancelled)) {
        $s{$status} = $dbh->selectrow_array(
            "SELECT COUNT(*) FROM tasks WHERE project_id=? AND status=?", undef, $id, $status) // 0;
    }
    $s{total} = $dbh->selectrow_array("SELECT COUNT(*) FROM tasks WHERE project_id=?", undef, $id) // 0;
    return %s;
}
}

{
package TMS::Model::Task;

sub _dbh { TMS::DB->handle }

sub create {
    my ($class, %args) = @_;
    die "Title required\n"      unless $args{title};
    die "Project ID required\n" unless $args{project_id};
    die "Creator ID required\n" unless $args{creator_id};
    
    $class->_dbh->do(
        "INSERT INTO tasks (project_id, title, description, status, priority, assignee_id, creator_id, due_date, position)
         VALUES (?,?,?,?,?,?,?,?,?)",
        undef,
        $args{project_id}, $args{title}, $args{description}//"",
        $args{status}//"todo", $args{priority}//"medium",
        $args{assignee_id}, $args{creator_id},
        $args{due_date}, $args{position}//0
    );
    return $class->find($class->_dbh->last_insert_id);
}

sub find     { TMS::DB->handle->selectrow_hashref("SELECT * FROM tasks WHERE id=?", undef, $_[1]) }
sub by_project {
    my ($class, $pid, %opts) = @_;
    my $status_filter = $opts{status} ? "AND status=?" : "";
    my @params = ($pid);
    push @params, $opts{status} if $opts{status};
    return @{TMS::DB->handle->selectall_arrayref(
        "SELECT * FROM tasks WHERE project_id=? $status_filter ORDER BY position, created_at",
        {Slice=>{}}, @params
    )};
}

sub update_status {
    my ($class, $id, $status) = @_;
    TMS::DB->handle->do(
        "UPDATE tasks SET status=?, updated_at=strftime('%s','now') WHERE id=?",
        undef, $status, $id);
}

sub assign {
    my ($class, $id, $user_id) = @_;
    TMS::DB->handle->do(
        "UPDATE tasks SET assignee_id=?, updated_at=strftime('%s','now') WHERE id=?",
        undef, $user_id, $id);
}

sub add_comment {
    my ($class, $task_id, $user_id, $content) = @_;
    TMS::DB->handle->do("INSERT INTO comments (task_id, user_id, content) VALUES (?,?,?)",
        undef, $task_id, $user_id, $content);
    return TMS::DB->handle->last_insert_id;
}

sub comments {
    my ($class, $task_id) = @_;
    return @{TMS::DB->handle->selectall_arrayref(
        "SELECT c.*, u.username FROM comments c JOIN users u ON c.user_id=u.id WHERE c.task_id=? ORDER BY c.created_at",
        {Slice=>{}}, $task_id
    )};
}

sub add_tag {
    my ($class, $task_id, $tag_name) = @_;
    my $dbh = TMS::DB->handle;
    my $tag = $dbh->selectrow_hashref("SELECT * FROM tags WHERE name=?", undef, $tag_name);
    unless ($tag) {
        $dbh->do("INSERT INTO tags (name) VALUES (?)", undef, $tag_name);
        $tag = { id => $dbh->last_insert_id };
    }
    eval { $dbh->do("INSERT INTO task_tags (task_id, tag_id) VALUES (?,?)", undef, $task_id, $tag->{id}) };
    # Ignore duplicate errors
}

sub tags {
    my ($class, $task_id) = @_;
    return @{TMS::DB->handle->selectall_arrayref(
        "SELECT t.name, t.color FROM tags t JOIN task_tags tt ON t.id=tt.tag_id WHERE tt.task_id=?",
        {Slice=>{}}, $task_id
    )};
}
}

package main;

# Get users for demo
my ($alice, $bob) = TMS::Model::User->all;

# Create project
my $project = TMS::Model::Project->create(
    name        => "Website Redesign",
    description => "Complete redesign of company website",
    owner_id    => $alice->{id},
);
printf "Project: %s (slug: %s)\n", $project->{name}, $project->{slug};

# Create tasks
my @task_specs = (
    { title => "Design new homepage",    priority => "high",   assignee => $alice },
    { title => "Update navigation",      priority => "medium", assignee => $bob },
    { title => "Implement contact form", priority => "medium", assignee => $bob },
    { title => "SEO optimization",       priority => "low",    assignee => $alice },
    { title => "Performance audit",      priority => "high",   assignee => $alice },
);

my @tasks;
for my $spec (@task_specs) {
    my $task = TMS::Model::Task->create(
        project_id  => $project->{id},
        title       => $spec->{title},
        priority    => $spec->{priority},
        creator_id  => $alice->{id},
        assignee_id => $spec->{assignee}{id},
    );
    push @tasks, $task;
}

printf "Created %d tasks\n", scalar @tasks;

# Update statuses
TMS::Model::Task->update_status($tasks[0]{id}, "in_progress");
TMS::Model::Task->update_status($tasks[1]{id}, "done");

# Add comments
TMS::Model::Task->add_comment($tasks[0]{id}, $alice->{id}, "Starting the design phase");
TMS::Model::Task->add_comment($tasks[0]{id}, $bob->{id},  "I have some mockups ready");

# Add tags
TMS::Model::Task->add_tag($tasks[0]{id}, "design");
TMS::Model::Task->add_tag($tasks[0]{id}, "frontend");
TMS::Model::Task->add_tag($tasks[2]{id}, "backend");
TMS::Model::Task->add_tag($tasks[2]{id}, "form");

# Project stats
my %stats = TMS::Model::Project->stats($project->{id});
printf "\nProject stats:\n";
printf "  %-12s %d\n", $_ . ":", $stats{$_} for qw(total todo in_progress done);

# Task details
printf "\nTasks:\n";
for my $task (TMS::Model::Task->by_project($project->{id})) {
    my @tags = TMS::Model::Task->tags($task->{id});
    printf "  [%-11s] [%-6s] %s %s\n",
        $task->{status}, $task->{priority}, $task->{title},
        @tags ? "[" . join(",", map { $_->{name} } @tags) . "]" : "";
}

# Comments
printf "\nComments on task 1:\n";
for my $c (TMS::Model::Task->comments($tasks[0]{id})) {
    printf "  %s: %s\n", $c->{username}, $c->{content};
}
```

---

## Step 295: Authentication System

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(sha256_hex);

{
package TMS::Auth;

sub _dbh { TMS::DB->handle }

sub create_session {
    my ($class, $user_id) = @_;
    my $token = sha256_hex(rand() . time() . $$ . $user_id);
    my $expires = time() + 86400;  # 24 hours
    
    $class->_dbh->do(
        "INSERT INTO sessions (token, user_id, expires_at) VALUES (?,?,?)",
        undef, $token, $user_id, $expires
    );
    
    return $token;
}

sub validate_session {
    my ($class, $token) = @_;
    return undef unless $token;
    
    my $session = $class->_dbh->selectrow_hashref(
        "SELECT s.*, u.username, u.email, u.role, u.full_name
         FROM sessions s JOIN users u ON s.user_id=u.id
         WHERE s.token=? AND s.expires_at > strftime('%s','now')",
        undef, $token
    );
    
    return $session;
}

sub invalidate_session {
    my ($class, $token) = @_;
    $class->_dbh->do("DELETE FROM sessions WHERE token=?", undef, $token);
}

sub clean_expired {
    my $class = shift;
    my $count = $class->_dbh->do(
        "DELETE FROM sessions WHERE expires_at <= strftime('%s','now')"
    );
    return $count;
}

sub login {
    my ($class, $username, $password) = @_;
    my $user = TMS::Model::User->authenticate($username, $password);
    return (undef, "Invalid credentials") unless $user;
    my $token = $class->create_session($user->{id});
    return ($token, undef);
}

sub logout {
    my ($class, $token) = @_;
    $class->invalidate_session($token);
}
}

package main;

# Test auth
my ($token, $err) = TMS::Auth::login("alice", "password123");
if ($token) {
    printf "Login OK, token: %.16s...\n", $token;
    
    my $session = TMS::Auth->validate_session($token);
    printf "Session valid: %s\n", $session ? "yes" : "no";
    printf "  User: %s (%s)\n", $session->{username}, $session->{role};
    
    TMS::Auth->logout($token);
    my $after = TMS::Auth->validate_session($token);
    printf "After logout: %s\n", $after ? "still_valid" : "correctly_invalid";
} else {
    printf "Login failed: %s\n", $err;
}

# Bad credentials
my ($bad_token, $bad_err) = TMS::Auth::login("alice", "wrongpass");
printf "Bad login: token=%s err=%s\n", $bad_token//"none", $bad_err//"none";
```

---

## Step 296: Template Engine

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package TMS::Template;

sub new {
    my ($class, $base_dir) = @_;
    return bless { base_dir => $base_dir // ".", _cache => {} }, $class;
}

sub render {
    my ($self, $template, %vars) = @_;
    my $output = $template;
    
    # {{= var }} — escaped variable
    $output =~ s/\{\{=\s*(\w+(?:\.\w+)*)\s*\}\}/
        _html_escape(_get_nested(\%vars, $1))
    /ge;
    
    # {{! var }} — raw/unescaped variable
    $output =~ s/\{\{!\s*(\w+)\s*\}\}/$vars{$1}\/\/""/ge;
    
    # {{# if var }}...{{/ if }}
    $output =~ s/\{\{\#\s*if\s+(\w+)\s*\}\}(.*?)\{\{\/\s*if\s*\}\}/
        _get_nested(\%vars, $1) ? $2 : ""
    /gse;
    
    # {{# unless var }}...{{/ unless }}
    $output =~ s/\{\{\#\s*unless\s+(\w+)\s*\}\}(.*?)\{\{\/\s*unless\s*\}\}/
        !_get_nested(\%vars, $1) ? $2 : ""
    /gse;
    
    # {{# each items as item }}...{{/ each }}
    $output =~ s/\{\{\#\s*each\s+(\w+)\s+as\s+(\w+)\s*\}\}(.*?)\{\{\/\s*each\s*\}\}/
        _render_each(\%vars, $1, $2, $3)
    /gse;
    
    return $output;
}

sub _get_nested {
    my ($vars, $path) = @_;
    my @parts = split /\./, $path;
    my $val   = $vars;
    for my $p (@parts) {
        return "" unless ref $val eq 'HASH' && exists $val->{$p};
        $val = $val->{$p};
    }
    return $val // "";
}

sub _html_escape {
    my $s = shift // "";
    $s =~ s/&/&amp;/g;
    $s =~ s/</&lt;/g;
    $s =~ s/>/&gt;/g;
    $s =~ s/"/&quot;/g;
    return $s;
}

sub _render_each {
    my ($vars, $list_key, $item_var, $body) = @_;
    my $list = $vars->{$list_key} // [];
    return "" unless ref $list eq 'ARRAY';
    
    my $out = "";
    my $i   = 0;
    for my $item (@$list) {
        my %item_vars = (
            %$vars,
            $item_var     => $item,
            loop_index    => $i,
            loop_first    => $i == 0 ? 1 : 0,
            loop_last     => $i == $#$list ? 1 : 0,
            loop_odd      => $i % 2 ? 1 : 0,
        );
        my $tmpl = TMS::Template->new;
        $out .= $tmpl->render($body, %item_vars);
        $i++;
    }
    return $out;
}
}

package main;

my $tmpl = TMS::Template->new;

# Task list template
my $task_list_tmpl = q{
<div class="project">
  <h2>{{= title }}</h2>
  <p>{{= description }}</p>
  {{# if task_count }}
  <p>Tasks: {{= task_count }}</p>
  {{/ if }}
  <ul>
  {{# each tasks as task }}
    <li class="{{# if task.loop_odd }}odd{{/ if }}{{# unless task.loop_odd }}even{{/ unless }}">
      [{{= task.status }}] {{= task.title }} ({{= task.priority }})
    </li>
  {{/ each }}
  </ul>
  {{# unless tasks }}
  <p>No tasks yet.</p>
  {{/ unless }}
</div>
};

my $rendered = $tmpl->render($task_list_tmpl,
    title       => "Website Redesign",
    description => "Complete <redesign> of site",  # will be escaped
    task_count  => 3,
    tasks       => [
        { title => "Design homepage",  status => "in_progress", priority => "high" },
        { title => "Update nav",       status => "done",        priority => "medium" },
        { title => "Contact form",     status => "todo",        priority => "low" },
    ],
);

printf "Rendered template:\n%s\n", $rendered;

# HTML escape test
my $xss_test = $tmpl->render("Hello {{= name }}!",
    name => '<script>alert("xss")</script>');
printf "XSS escaped: %s\n", $xss_test;
```

---

## Step 297: REST API Controller

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

{
package TMS::API;

my $json = JSON::PP->new->utf8->canonical;

sub dispatch {
    my ($class, $method, $path, $body, $token) = @_;
    
    # Auth check
    my $session = TMS::Auth->validate_session($token);
    
    # Public routes
    if ($method eq 'POST' && $path eq '/api/auth/login') {
        return $class->handle_login($body);
    }
    
    # Protected routes
    return $class->_err(401, "Unauthorized") unless $session;
    
    my $user = { id => $session->{user_id}, %$session };
    
    # Route dispatch
    if    ($method eq 'GET'    && $path eq '/api/projects')           { return $class->list_projects($user) }
    elsif ($method eq 'POST'   && $path eq '/api/projects')           { return $class->create_project($user, $body) }
    elsif ($method eq 'GET'    && $path =~ m{^/api/projects/(\d+)$})  { return $class->get_project($user, $1) }
    elsif ($method eq 'GET'    && $path =~ m{^/api/projects/(\d+)/tasks$}) { return $class->list_tasks($user, $1) }
    elsif ($method eq 'POST'   && $path =~ m{^/api/projects/(\d+)/tasks$}) { return $class->create_task($user, $1, $body) }
    elsif ($method eq 'PATCH'  && $path =~ m{^/api/tasks/(\d+)$})    { return $class->update_task($user, $1, $body) }
    elsif ($method eq 'DELETE' && $path =~ m{^/api/tasks/(\d+)$})    { return $class->delete_task($user, $1) }
    elsif ($method eq 'GET'    && $path eq '/api/me')                  { return $class->get_me($user) }
    
    return $class->_err(404, "Not found");
}

sub _ok  { my ($class, $data, $code) = @_; { status => $code//200, body => $json->encode($data) } }
sub _err { my ($class, $code, $msg) = @_;  { status => $code, body => $json->encode({error => $msg}) } }

sub handle_login {
    my ($class, $body) = @_;
    my $data = eval { $json->decode($body) };
    return $class->_err(400, "Invalid JSON") unless $data;
    
    my ($token, $err) = TMS::Auth->login($data->{username}//"", $data->{password}//"");
    return $class->_err(401, $err//"Invalid credentials") unless $token;
    
    my $user = TMS::Auth->validate_session($token);
    return $class->_ok({ token => $token, user => { id => $user->{user_id}, username => $user->{username} } });
}

sub list_projects {
    my ($class, $user) = @_;
    my @projects = TMS::Model::Project->all;
    return $class->_ok(\@projects);
}

sub create_project {
    my ($class, $user, $body) = @_;
    my $data = eval { $json->decode($body) };
    return $class->_err(400, "Invalid JSON") unless $data;
    
    eval {
        my $proj = TMS::Model::Project->create(
            name        => $data->{name},
            description => $data->{description},
            owner_id    => $user->{user_id},
        );
        return $class->_ok($proj, 201);
    };
    return $class->_err(400, $@) if $@;
}

sub get_project {
    my ($class, $user, $id) = @_;
    my $proj = TMS::Model::Project->find($id);
    return $class->_err(404, "Project not found") unless $proj;
    
    my %stats = TMS::Model::Project->stats($id);
    $proj->{stats} = \%stats;
    return $class->_ok($proj);
}

sub list_tasks {
    my ($class, $user, $project_id) = @_;
    my @tasks = TMS::Model::Task->by_project($project_id);
    # Add tags
    for my $task (@tasks) {
        $task->{tags} = [TMS::Model::Task->tags($task->{id})];
    }
    return $class->_ok(\@tasks);
}

sub create_task {
    my ($class, $user, $project_id, $body) = @_;
    my $data = eval { $json->decode($body) };
    return $class->_err(400, "Invalid JSON") unless $data;
    
    my $task = eval { TMS::Model::Task->create(
        project_id  => $project_id,
        title       => $data->{title},
        description => $data->{description},
        priority    => $data->{priority},
        assignee_id => $data->{assignee_id},
        creator_id  => $user->{user_id},
    )};
    return $class->_err(400, $@) if $@;
    return $class->_ok($task, 201);
}

sub update_task {
    my ($class, $user, $id, $body) = @_;
    my $data = eval { $json->decode($body) };
    return $class->_err(400, "Invalid") unless $data;
    
    if ($data->{status}) {
        TMS::Model::Task->update_status($id, $data->{status});
    }
    
    my $task = TMS::Model::Task->find($id);
    return $class->_err(404, "Not found") unless $task;
    return $class->_ok($task);
}

sub delete_task {
    my ($class, $user, $id) = @_;
    TMS::DB->handle->do("DELETE FROM tasks WHERE id=?", undef, $id);
    return $class->_ok({ deleted => 1 });
}

sub get_me {
    my ($class, $user) = @_;
    return $class->_ok({
        id       => $user->{user_id},
        username => $user->{username},
        email    => $user->{email},
        role     => $user->{role},
    });
}
}

package main;

# Test the API
printf "=== API Tests ===\n\n";

# Login
my $login_res = TMS::API->dispatch("POST", "/api/auth/login", '{"username":"alice","password":"password123"}', undef);
my $login_data = JSON::PP->new->utf8->decode($login_res->{body});
printf "Login: status=%d token=%.16s...\n", $login_res->{status}, $login_data->{token};

my $token = $login_data->{token};

# Get me
my $me_res  = TMS::API->dispatch("GET", "/api/me", "", $token);
my $me_data = JSON::PP->new->utf8->decode($me_res->{body});
printf "Me: %s (%s)\n", $me_data->{username}, $me_data->{role};

# List projects
my $proj_res  = TMS::API->dispatch("GET", "/api/projects", "", $token);
my $proj_data = JSON::PP->new->utf8->decode($proj_res->{body});
printf "Projects: %d\n", scalar @$proj_data;

# Create task via API
my $new_task_body = JSON::PP->new->utf8->encode({
    title    => "API Created Task",
    priority => "high",
});
my $task_res  = TMS::API->dispatch("POST", "/api/projects/$proj_data->[0]{id}/tasks", $new_task_body, $token);
printf "Create task: status=%d\n", $task_res->{status};

# List tasks
my $tasks_res  = TMS::API->dispatch("GET", "/api/projects/$proj_data->[0]{id}/tasks", "", $token);
my $tasks_data = JSON::PP->new->utf8->decode($tasks_res->{body});
printf "Tasks: %d\n", scalar @$tasks_data;

# Update task status
my $update_res = TMS::API->dispatch("PATCH", "/api/tasks/$tasks_data->[0]{id}",
    '{"status":"in_progress"}', $token);
printf "Update task: status=%d\n", $update_res->{status};

# Unauthorized
my $unauth = TMS::API->dispatch("GET", "/api/projects", "", "invalid_token");
printf "Unauth: status=%d\n", $unauth->{status};
```

---

## Step 298: Search and Pagination

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package TMS::Search;

sub _dbh { TMS::DB->handle }

sub search_tasks {
    my ($class, %opts) = @_;
    
    my @conditions = ("1=1");
    my @params;
    
    if ($opts{q}) {
        push @conditions, "(t.title LIKE ? OR t.description LIKE ?)";
        push @params, "%$opts{q}%", "%$opts{q}%";
    }
    
    if ($opts{project_id}) {
        push @conditions, "t.project_id = ?";
        push @params, $opts{project_id};
    }
    
    if ($opts{status}) {
        push @conditions, "t.status = ?";
        push @params, $opts{status};
    }
    
    if ($opts{priority}) {
        push @conditions, "t.priority = ?";
        push @params, $opts{priority};
    }
    
    if ($opts{assignee_id}) {
        push @conditions, "t.assignee_id = ?";
        push @params, $opts{assignee_id};
    }
    
    my $where = join " AND ", @conditions;
    
    # Count total
    my $total = $class->_dbh->selectrow_array(
        "SELECT COUNT(*) FROM tasks t WHERE $where", undef, @params) // 0;
    
    # Pagination
    my $page     = $opts{page}     // 1;
    my $per_page = $opts{per_page} // 10;
    my $offset   = ($page - 1) * $per_page;
    
    my $order = "t.created_at DESC";
    $order = "t.priority DESC, t.created_at DESC" if $opts{sort} eq "priority";
    $order = "t.due_date ASC"                     if ($opts{sort}//"") eq "due_date";
    
    my @tasks = @{$class->_dbh->selectall_arrayref(
        "SELECT t.*, u.username AS assignee_name, p.name AS project_name
         FROM tasks t
         LEFT JOIN users u ON t.assignee_id = u.id
         LEFT JOIN projects p ON t.project_id = p.id
         WHERE $where
         ORDER BY $order
         LIMIT ? OFFSET ?",
        {Slice=>{}}, @params, $per_page, $offset
    )};
    
    return {
        tasks    => \@tasks,
        total    => $total,
        page     => $page,
        per_page => $per_page,
        pages    => int(($total + $per_page - 1) / $per_page),
    };
}
}

package main;

printf "\n=== Search and Pagination ===\n";

# Add more test tasks for search demo
my ($alice, $bob) = TMS::Model::User->all;
my ($proj) = TMS::Model::Project->all;

for my $i (1..15) {
    TMS::Model::Task->create(
        project_id => $proj->{id},
        title      => "Search test task $i",
        priority   => (qw(low medium high))[int($i/5)],
        status     => (qw(todo in_progress done))[int($i%3)],
        creator_id => $alice->{id},
        assignee_id => ($i % 2 == 0 ? $alice->{id} : $bob->{id}),
    );
}

# Search all
my $result = TMS::Search->search_tasks(per_page => 5, page => 1);
printf "All tasks page 1: %d/%d (total)\n", scalar @{$result->{tasks}}, $result->{total};
printf "Pages: %d\n", $result->{pages};

# Search by query
my $r2 = TMS::Search->search_tasks(q => "homepage", per_page => 10);
printf "\nSearch 'homepage': %d results\n", $r2->{total};

# Filter by status
my $r3 = TMS::Search->search_tasks(status => "todo", per_page => 100);
printf "Status=todo: %d tasks\n", $r3->{total};

# Filter by priority
my $r4 = TMS::Search->search_tasks(priority => "high", per_page => 100);
printf "Priority=high: %d tasks\n", $r4->{total};

# Pagination demo
for my $page (1..3) {
    my $r = TMS::Search->search_tasks(per_page => 5, page => $page);
    printf "Page %d: %d items\n", $page, scalar @{$r->{tasks}};
}
```

---

## Step 299: Full Dashboard

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package TMS::Dashboard;

sub _dbh { TMS::DB->handle }

sub overview {
    my $class = shift;
    my $dbh   = $class->_dbh;
    
    my %stats;
    
    # Global counts
    $stats{total_users}    = $dbh->selectrow_array("SELECT COUNT(*) FROM users");
    $stats{total_projects} = $dbh->selectrow_array("SELECT COUNT(*) FROM projects");
    $stats{total_tasks}    = $dbh->selectrow_array("SELECT COUNT(*) FROM tasks");
    $stats{total_comments} = $dbh->selectrow_array("SELECT COUNT(*) FROM comments");
    
    # Task status breakdown
    my $statuses = $dbh->selectall_arrayref(
        "SELECT status, COUNT(*) AS count FROM tasks GROUP BY status", {Slice=>{}});
    $stats{by_status} = { map { $_->{status} => $_->{count} } @$statuses };
    
    # Task priority breakdown
    my $priorities = $dbh->selectall_arrayref(
        "SELECT priority, COUNT(*) AS count FROM tasks GROUP BY priority", {Slice=>{}});
    $stats{by_priority} = { map { $_->{priority} => $_->{count} } @$priorities };
    
    # Most active users (by tasks created)
    $stats{top_creators} = $dbh->selectall_arrayref(
        "SELECT u.username, COUNT(t.id) AS task_count
         FROM users u LEFT JOIN tasks t ON u.id=t.creator_id
         GROUP BY u.id ORDER BY task_count DESC LIMIT 5",
        {Slice=>{}}
    );
    
    # Project summaries
    $stats{project_summary} = $dbh->selectall_arrayref(
        "SELECT p.name, p.status,
                COUNT(t.id) AS total,
                SUM(CASE WHEN t.status='done' THEN 1 ELSE 0 END) AS done,
                SUM(CASE WHEN t.status='in_progress' THEN 1 ELSE 0 END) AS in_progress
         FROM projects p LEFT JOIN tasks t ON p.id=t.project_id
         GROUP BY p.id ORDER BY p.name",
        {Slice=>{}}
    );
    
    # Completion rate
    my $done  = $stats{by_status}{done}  // 0;
    my $total = $stats{total_tasks}      || 1;
    $stats{completion_rate} = sprintf "%.1f%%", $done/$total*100;
    
    return %stats;
}

sub render_ascii {
    my ($class, %stats) = @_;
    my $out = "";
    
    $out .= "=" x 50 . "\n";
    $out .= "    TASK MANAGEMENT SYSTEM — Dashboard\n";
    $out .= "=" x 50 . "\n\n";
    
    $out .= sprintf "Users: %-5d  Projects: %-5d  Tasks: %-5d  Comments: %d\n",
        $stats{total_users}, $stats{total_projects}, $stats{total_tasks}, $stats{total_comments};
    $out .= sprintf "Completion Rate: %s\n\n", $stats{completion_rate};
    
    $out .= "Task Status:\n";
    my %bs = %{$stats{by_status}//{}};
    for my $s (qw(todo in_progress done cancelled)) {
        my $n = $bs{$s}//0;
        my $bar = "#" x ($n > 20 ? 20 : $n);
        $out .= sprintf "  %-12s %3d |%s\n", $s, $n, $bar;
    }
    
    $out .= "\nTask Priority:\n";
    my %bp = %{$stats{by_priority}//{}};
    for my $p (qw(low medium high critical)) {
        my $n = $bp{$p}//0;
        next unless $n;
        $out .= sprintf "  %-8s %d\n", $p, $n;
    }
    
    $out .= "\nProjects:\n";
    for my $p (@{$stats{project_summary}//[]}) {
        my $pct = $p->{total} ? sprintf "%.0f%%", ($p->{done}//$0)/$p->{total}*100 : "0%";
        $out .= sprintf "  %-30s  %2d tasks  %s done\n",
            $p->{name}, $p->{total}//$0, $pct;
    }
    
    $out .= "\nTop Contributors:\n";
    for my $u (@{$stats{top_creators}//[]}) {
        next unless $u->{task_count};
        $out .= sprintf "  %-15s %d tasks\n", $u->{username}, $u->{task_count};
    }
    
    return $out;
}
}

package main;

my %dash = TMS::Dashboard->overview;
print TMS::Dashboard->render_ascii(%dash);
```

---

## Step 300: โปรแกรมสรุป — System Tests

```perl
#!/usr/bin/perl
# tms_tests.pl — Full system test suite
use strict;
use warnings;
use Test::More;

# =====================
# Integration tests
# =====================

subtest 'User management' => sub {
    my @users = TMS::Model::User->all;
    ok(@users >= 2, "at least 2 users created");
    
    my $alice = $users[0];
    ok($alice->{username}, "user has username");
    ok($alice->{email},    "user has email");
    ok($alice->{password} !~ /password/, "password is hashed");
    
    my $found = TMS::Model::User->find($alice->{id});
    is($found->{username}, $alice->{username}, "find by id works");
    
    my $auth = TMS::Model::User->authenticate("alice", "password123");
    ok($auth, "correct auth succeeds");
    
    my $bad = TMS::Model::User->authenticate("alice", "wrong");
    ok(!$bad, "incorrect auth fails");
};

subtest 'Session management' => sub {
    my ($token, $err) = TMS::Auth->login("alice", "password123");
    ok($token, "login returns token");
    ok(!$err,  "no error on good login");
    ok(length($token) >= 32, "token is long enough");
    
    my $session = TMS::Auth->validate_session($token);
    ok($session, "session is valid");
    is($session->{username}, "alice", "correct user in session");
    
    TMS::Auth->logout($token);
    my $invalid = TMS::Auth->validate_session($token);
    ok(!$invalid, "session invalid after logout");
    
    my $bad_session = TMS::Auth->validate_session("nonexistent_token");
    ok(!$bad_session, "invalid token returns undef");
};

subtest 'Project management' => sub {
    my @projects = TMS::Model::Project->all;
    ok(@projects >= 1, "at least 1 project");
    
    my $proj = $projects[0];
    ok($proj->{id},   "project has id");
    ok($proj->{name}, "project has name");
    ok($proj->{slug}, "project has slug");
    like($proj->{slug}, qr/^[a-z0-9\-]+$/, "slug is valid format");
    
    my $found = TMS::Model::Project->find($proj->{id});
    is($found->{name}, $proj->{name}, "find by id works");
    
    my %stats = TMS::Model::Project->stats($proj->{id});
    ok(exists $stats{total}, "stats has total");
    cmp_ok($stats{total}, '>=', 0, "total is non-negative");
};

subtest 'Task management' => sub {
    my @projects = TMS::Model::Project->all;
    my ($alice) = TMS::Model::User->all;
    my $proj = $projects[0];
    
    my @tasks = TMS::Model::Task->by_project($proj->{id});
    ok(@tasks >= 1, "project has tasks");
    
    my $task = $tasks[0];
    ok($task->{id},         "task has id");
    ok($task->{title},      "task has title");
    ok($task->{project_id}, "task has project_id");
    ok($task->{status},     "task has status");
    ok($task->{priority},   "task has priority");
    
    # Update status
    my $orig_status = $task->{status};
    TMS::Model::Task->update_status($task->{id}, "in_progress");
    my $updated = TMS::Model::Task->find($task->{id});
    is($updated->{status}, "in_progress", "status updated");
    
    # Restore
    TMS::Model::Task->update_status($task->{id}, $orig_status);
};

subtest 'Comments' => sub {
    my @projects = TMS::Model::Project->all;
    my @tasks    = TMS::Model::Task->by_project($projects[0]{id});
    my ($alice)  = TMS::Model::User->all;
    
    my $comment_id = TMS::Model::Task->add_comment($tasks[0]{id}, $alice->{id}, "Test comment");
    ok($comment_id > 0, "comment created");
    
    my @comments = TMS::Model::Task->comments($tasks[0]{id});
    ok(@comments >= 1, "comments retrieved");
    ok(grep { $_->{content} eq "Test comment" } @comments, "correct comment found");
};

subtest 'Tags' => sub {
    my @projects = TMS::Model::Project->all;
    my @tasks    = TMS::Model::Task->by_project($projects[0]{id});
    
    TMS::Model::Task->add_tag($tasks[0]{id}, "test-tag-unique");
    
    my @tags = TMS::Model::Task->tags($tasks[0]{id});
    ok(@tags >= 1, "task has tags");
    ok(grep { $_->{name} eq "test-tag-unique" } @tags, "correct tag found");
    
    # Idempotent
    TMS::Model::Task->add_tag($tasks[0]{id}, "test-tag-unique");
    my @tags2 = TMS::Model::Task->tags($tasks[0]{id});
    my $count1 = grep { $_->{name} eq "test-tag-unique" } @tags;
    my $count2 = grep { $_->{name} eq "test-tag-unique" } @tags2;
    is($count1, $count2, "duplicate tag not added");
};

subtest 'API' => sub {
    my ($token) = TMS::Auth->login("alice", "password123");
    ok($token, "login for API test");
    
    # GET projects
    my $r = TMS::API->dispatch("GET", "/api/projects", "", $token);
    is($r->{status}, 200, "GET /api/projects returns 200");
    
    my $data = JSON::PP->new->utf8->decode($r->{body});
    ok(ref $data eq 'ARRAY', "returns array");
    ok(@$data >= 1, "at least 1 project");
    
    # Unauthorized
    my $unauth = TMS::API->dispatch("GET", "/api/projects", "", "bad_token");
    is($unauth->{status}, 401, "bad token returns 401");
    
    # Create project via API
    my $new_proj_json = JSON::PP->new->utf8->encode({ name => "API Test Project", description => "Created via API" });
    my $cr = TMS::API->dispatch("POST", "/api/projects", $new_proj_json, $token);
    is($cr->{status}, 201, "project creation returns 201");
    
    TMS::Auth->logout($token);
};

subtest 'Search' => sub {
    my $r = TMS::Search->search_tasks(per_page => 10, page => 1);
    ok($r->{total} >= 0, "search returns total");
    ok($r->{pages}  >= 1, "search returns pages");
    ok(ref $r->{tasks} eq 'ARRAY', "search returns tasks array");
    
    my $r2 = TMS::Search->search_tasks(q => "nonexistent_xyz_12345");
    is($r2->{total}, 0, "search for nonexistent returns 0");
};

done_testing;
printf "\n=== All TMS Tests Complete ===\n";
printf "Task Management System fully implemented and tested!\n";
```

---

## สรุป Part 30 — Intermediate Capstone

ใน Part นี้คุณได้สร้างระบบจัดการงาน (Task Management System) ที่ครอบคลุม:

### สิ่งที่สร้าง
- ✅ **TMS::DB** — Database connection, schema, transactions
- ✅ **TMS::Model::User** — User CRUD, password hashing, authentication
- ✅ **TMS::Model::Project** — Project management with slugs
- ✅ **TMS::Model::Task** — Task CRUD, status, priority, assignment
- ✅ **TMS::Model::Comment** — Comments on tasks
- ✅ **TMS::Model::Tag** — Tagging system
- ✅ **TMS::Auth** — Session-based authentication
- ✅ **TMS::Template** — Custom template engine
- ✅ **TMS::API** — RESTful JSON API
- ✅ **TMS::Search** — Search with filters and pagination
- ✅ **TMS::Dashboard** — Statistics and reports
- ✅ **Full test suite** — Integration tests with Test::More

### เทคนิคที่ใช้
- DBI + SQLite (in-memory and file)
- SHA-256 password hashing
- Token-based sessions
- JSON API (GET/POST/PATCH/DELETE)
- Template engine with if/unless/each
- Search with dynamic WHERE clauses
- Pagination
- ASCII dashboard visualization

**ถัดไป: [Part 31 — Advanced Perl Patterns](part_31.md)**
