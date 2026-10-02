# Part 50: Capstone Project — Full-Stack Perl Application
## Steps 491-500: REST API + Auth + Caching + Background Jobs + Email

---

## Step 491: Project Architecture

```
TaskFlow — Project Management REST API
=======================================

Architecture:
  ┌─────────────────────────────────────────────┐
  │  HTTP Router (Part 46 HTTP::Router)          │
  ├─────────────────────────────────────────────┤
  │  Middleware Chain                            │
  │  ├── Auth (JWT — Part 40)                   │
  │  ├── Rate Limiting (Part 42)                │
  │  └── Request Logging                        │
  ├─────────────────────────────────────────────┤
  │  Controllers                                │
  │  ├── AuthController    (register/login)     │
  │  ├── ProjectController (CRUD)               │
  │  ├── TaskController    (CRUD + assign)      │
  │  └── UserController    (profile/teams)      │
  ├─────────────────────────────────────────────┤
  │  Services                                   │
  │  ├── AuthService  (JWT, password hashing)   │
  │  ├── CacheService (L1 memory + L2 file)     │
  │  ├── EmailService (SMTP + templates)        │
  │  └── JobService   (background jobs)         │
  ├─────────────────────────────────────────────┤
  │  Data Layer (in-memory for demo)            │
  │  ├── UserRepository                         │
  │  ├── ProjectRepository                      │
  │  └── TaskRepository                         │
  └─────────────────────────────────────────────┘
```

---

## Step 492: Data Models

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Model::Base;

sub new {
    my ($class, %data) = @_;
    return bless { %data }, $class;
}

sub to_hash {
    my $self = shift;
    return { map { $_ => $self->{$_} } grep { !/^_/ } keys %$self };
}

sub as_json {
    require JSON::PP;
    return JSON::PP->new->encode($_[0]->to_hash);
}
}

{
package Model::User;
our @ISA = ("Model::Base");

sub new {
    my ($class, %data) = @_;
    my $self = $class->SUPER::new(%data);
    $self->{id}         //= _uuid();
    $self->{created_at} //= time();
    $self->{role}       //= "member";
    $self->{active}     //= 1;
    $self->{projects}   //= [];
    return $self;
}

sub id         { $_[0]->{id} }
sub name       { $_[0]->{name} }
sub email      { $_[0]->{email} }
sub role       { $_[0]->{role} }
sub is_admin   { $_[0]->{role} eq "admin" }
sub is_active  { $_[0]->{active} }

sub to_public {
    my $self = shift;
    return { map { $_ => $self->{$_} } qw(id name email role created_at) };
}
}

{
package Model::Project;
our @ISA = ("Model::Base");

sub new {
    my ($class, %data) = @_;
    my $self = $class->SUPER::new(%data);
    $self->{id}          //= _uuid();
    $self->{created_at}  //= time();
    $self->{status}      //= "active";
    $self->{members}     //= [];
    $self->{task_count}  //= 0;
    return $self;
}

sub id      { $_[0]->{id} }
sub name    { $_[0]->{name} }
sub status  { $_[0]->{status} }

sub add_member {
    my ($self, $user_id) = @_;
    push @{$self->{members}}, $user_id unless grep { $_ eq $user_id } @{$self->{members}};
}

sub has_member {
    my ($self, $user_id) = @_;
    return grep { $_ eq $user_id } @{$self->{members}};
}
}

{
package Model::Task;
our @ISA = ("Model::Base");

use constant STATUSES  => [qw(todo in_progress review done)];
use constant PRIORITIES=> [qw(low medium high critical)];

sub new {
    my ($class, %data) = @_;
    my $self = $class->SUPER::new(%data);
    $self->{id}          //= _uuid();
    $self->{created_at}  //= time();
    $self->{status}      //= "todo";
    $self->{priority}    //= "medium";
    $self->{tags}        //= [];
    $self->{comments}    //= [];
    return $self;
}

sub id       { $_[0]->{id} }
sub title    { $_[0]->{title} }
sub status   { $_[0]->{status} }
sub priority { $_[0]->{priority} }
sub assignee { $_[0]->{assignee_id} }

sub assign {
    my ($self, $user_id) = @_;
    $self->{assignee_id} = $user_id;
    $self->{assigned_at} = time();
}

sub move_to {
    my ($self, $status) = @_;
    die "Invalid status: $status" unless grep { $_ eq $status } @{+STATUSES};
    my $old = $self->{status};
    $self->{status} = $status;
    $self->{updated_at} = time();
    $self->{completed_at} = time() if $status eq "done";
    return $old;
}

sub add_comment {
    my ($self, $user_id, $text) = @_;
    push @{$self->{comments}}, {
        id      => _uuid(),
        user_id => $user_id,
        text    => $text,
        at      => time(),
    };
}
}

{
package Repository;

sub new {
    my ($class, %opts) = @_;
    return bless {
        store   => {},
        indices => {},
        _seq    => 0,
    }, $class;
}

sub save {
    my ($self, $obj) = @_;
    $self->{store}{$obj->id} = $obj;
    return $obj;
}

sub find { $_[0]->{store}{$_[1]} }

sub find_all {
    my ($self, %filter) = @_;
    my @all = values %{$self->{store}};
    for my $key (keys %filter) {
        @all = grep { defined $_->{$key} && $_->{$key} eq $filter{$key} } @all;
    }
    return @all;
}

sub delete {
    my ($self, $id) = @_;
    delete $self->{store}{$id};
}

sub count { scalar keys %{$_[0]->{store}} }

sub all { values %{$_[0]->{store}} }
}

sub _uuid {
    my @chars = ("a".."f", 0..9);
    return join "-", map { join("", map { $chars[rand @chars] } 1..$_) } (8,4,4,4,12);
}

package main;

printf "=== Data Models ===\n\n";

my $users    = Repository->new;
my $projects = Repository->new;
my $tasks    = Repository->new;

# Create users
my $alice = Model::User->new(name=>"Alice Smith",  email=>"alice\@test.com",  role=>"admin");
my $bob   = Model::User->new(name=>"Bob Jones",    email=>"bob\@test.com");
my $carol = Model::User->new(name=>"Carol White",  email=>"carol\@test.com");

$users->save($_) for ($alice, $bob, $carol);

printf "Users created: %d\n", $users->count;

# Create project
my $proj = Model::Project->new(
    name    => "TaskFlow Development",
    owner   => $alice->id,
    desc    => "Build the TaskFlow application",
);
$proj->add_member($alice->id);
$proj->add_member($bob->id);
$projects->save($proj);

printf "Project: %s (members: %d)\n", $proj->name, scalar @{$proj->{members}};

# Create tasks
my @task_defs = (
    { title=>"Design database schema", priority=>"high",     assignee=>$alice },
    { title=>"Implement auth module",  priority=>"critical", assignee=>$bob },
    { title=>"Build REST API",         priority=>"high",     assignee=>$carol },
    { title=>"Write unit tests",       priority=>"medium",   assignee=>$bob },
    { title=>"Deploy to staging",      priority=>"low",      assignee=>$alice },
);

for my $td (@task_defs) {
    my $task = Model::Task->new(
        title      => $td->{title},
        priority   => $td->{priority},
        project_id => $proj->id,
    );
    $task->assign($td->{assignee}->id);
    $task->add_comment($alice->id, "Initial task created");
    $tasks->save($task);
}

printf "Tasks created: %d\n\n", $tasks->count;

# Workflow simulation
my @project_tasks = $tasks->find_all(project_id => $proj->id);
printf "Task workflow:\n";
printf "%-35s %-12s %s\n", "Title", "Priority", "Status";
printf "%s\n", "-" x 60;
printf "%-35s %-12s %s\n", $_->title, $_->priority, $_->status for @project_tasks;

# Move some tasks
printf "\nMoving tasks through workflow:\n";
my ($task1) = grep { $_->title =~ /schema/ } @project_tasks;
my ($task2) = grep { $_->title =~ /auth/   } @project_tasks;
$task1->move_to("in_progress"); printf "  '%s' -> in_progress\n", $task1->title;
$task2->move_to("in_progress"); printf "  '%s' -> in_progress\n", $task2->title;
$task1->move_to("review");      printf "  '%s' -> review\n",      $task1->title;
$task1->move_to("done");        printf "  '%s' -> done\n",        $task1->title;

my $done = scalar grep { $_->status eq "done" } @project_tasks;
printf "\nCompleted: %d/%d tasks\n", $done, scalar @project_tasks;
```

---

## Step 493-500: REST API + Auth + Full Capstone

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(hmac_sha256 sha256_hex);
use MIME::Base64 qw(encode_base64url decode_base64url encode_base64 decode_base64);

# ===== JWT Service =====
{
package JWTService;

my $SECRET = "taskflow-secret-key-2024";

sub _b64url { my $b = encode_base64($_[0],""); $b=~tr|+/=|-_|d; $b }
sub _b64dec {
    my $s = shift; $s=~tr|-_|+/|;
    my $pad = (4 - length($s) % 4) % 4;
    return decode_base64($s . "=" x $pad);
}

sub generate {
    my ($class, $payload, $ttl) = @_;
    $ttl //= 3600;
    $payload->{iat} = time();
    $payload->{exp} = time() + $ttl;
    
    require JSON::PP;
    my $j = JSON::PP->new;
    my $header = $class->_b64url($j->encode({ alg=>"HS256", typ=>"JWT" }));
    my $claims = $class->_b64url($j->encode($payload));
    my $sig    = $class->_b64url(hmac_sha256("$header.$claims", $SECRET));
    
    return "$header.$claims.$sig";
}

sub verify {
    my ($class, $token) = @_;
    my ($header, $claims, $sig) = split /\./, $token, 3;
    return undef unless $header && $claims && $sig;
    
    my $expected = $class->_b64url(hmac_sha256("$header.$claims", $SECRET));
    return undef unless $sig eq $expected;
    
    require JSON::PP;
    my $payload = eval { JSON::PP->new->decode($class->_b64dec($claims)) };
    return undef unless $payload;
    return undef if $payload->{exp} < time();
    
    return $payload;
}
}

# ===== Auth Service =====
{
package AuthService;

my %_users_db;  # email -> user record

sub register {
    my ($class, %args) = @_;
    die "Email already registered\n" if $_users_db{lc $args{email}};
    die "Password too short\n" unless length($args{password}) >= 8;
    
    my $id = "user_" . sprintf("%04d", scalar keys %_users_db + 1);
    my $hash = sha256_hex($args{password} . "salt_$id");
    
    my $user = {
        id         => $id,
        name       => $args{name},
        email      => lc $args{email},
        password   => $hash,
        role       => "member",
        created_at => time(),
    };
    $_users_db{$user->{email}} = $user;
    return $user;
}

sub login {
    my ($class, $email, $password) = @_;
    my $user = $_users_db{lc $email} or die "Invalid credentials\n";
    my $expected = sha256_hex($password . "salt_$user->{id}");
    die "Invalid credentials\n" unless $user->{password} eq $expected;
    
    my $token = JWTService->generate({
        sub   => $user->{id},
        email => $user->{email},
        role  => $user->{role},
    });
    my $refresh = JWTService->generate({ sub=>$user->{id}, type=>"refresh" }, 7*24*3600);
    
    return { user => $user, token => $token, refresh_token => $refresh };
}

sub get_user { $_users_db{lc $_[1]} }
sub all_users { values %_users_db }
}

# ===== Rate Limiter =====
{
package RateLimiter;

my %_buckets;

sub new {
    my ($class, %opts) = @_;
    return bless {
        rate     => $opts{rate}     // 100,
        per      => $opts{per}      // 60,
        burst    => $opts{burst}    // 10,
    }, $class;
}

sub check {
    my ($self, $key) = @_;
    my $bucket = $_buckets{$key} //= {
        tokens => $self->{burst},
        last   => time(),
    };
    
    my $elapsed = time() - $bucket->{last};
    my $refill  = $elapsed * ($self->{rate} / $self->{per});
    $bucket->{tokens} = List::Util::min($self->{burst}, $bucket->{tokens} + $refill);
    $bucket->{last}   = time();
    
    if ($bucket->{tokens} >= 1) {
        $bucket->{tokens}--;
        return 1;
    }
    return 0;
}
}

# ===== Middleware =====
{
package Middleware;

sub jwt_auth {
    my ($handler) = @_;
    return sub {
        my ($req) = @_;
        my $auth = $req->{headers}{"authorization"} // "";
        
        unless ($auth =~ /^Bearer\s+(.+)/) {
            return { status=>401, body=>encode_json({error=>"No token"}) };
        }
        
        my $payload = JWTService->verify($1);
        unless ($payload) {
            return { status=>401, body=>encode_json({error=>"Invalid token"}) };
        }
        
        $req->{user} = $payload;
        return $handler->($req);
    };
}

sub rate_limit {
    my ($handler, $limiter) = @_;
    return sub {
        my ($req) = @_;
        my $key = $req->{user}{sub} // $req->{ip} // "anonymous";
        
        unless ($limiter->check($key)) {
            return { status=>429, body=>encode_json({error=>"Rate limit exceeded"}) };
        }
        return $handler->($req);
    };
}

sub request_logger {
    my ($handler) = @_;
    return sub {
        my ($req) = @_;
        my $t0 = time();
        my $resp = $handler->($req);
        printf "  [LOG] %s %s -> %d (%dms)\n",
            $req->{method}, $req->{path}, $resp->{status}, (time()-$t0)*1000;
        return $resp;
    };
}

sub encode_json {
    require JSON::PP;
    JSON::PP->new->encode($_[0]);
}
}

# ===== Router & Controllers =====
{
package App;
use List::Util qw(min);

my %_projects;
my %_tasks;
my %_seq = (project => 0, task => 0);

sub new {
    my ($class) = @_;
    my $self = bless { routes => [], limiter => RateLimiter->new(rate=>60, per=>60, burst=>5) }, $class;
    $self->_setup_routes;
    return $self;
}

sub _setup_routes {
    my $self = shift;
    
    # Auth routes (no JWT required)
    $self->route("POST", "/auth/register", \&_register);
    $self->route("POST", "/auth/login",    \&_login);
    
    # Protected routes
    my $protected = sub {
        my $h = shift;
        Middleware::rate_limit(
            Middleware::jwt_auth($h),
            $self->{limiter}
        );
    };
    
    $self->route("GET",    "/api/me",             $protected->(\&_me));
    $self->route("GET",    "/api/projects",       $protected->(\&_list_projects));
    $self->route("POST",   "/api/projects",       $protected->(\&_create_project));
    $self->route("GET",    "/api/projects/:id",   $protected->(\&_get_project));
    $self->route("PUT",    "/api/projects/:id",   $protected->(\&_update_project));
    $self->route("DELETE", "/api/projects/:id",   $protected->(\&_delete_project));
    $self->route("GET",    "/api/tasks",          $protected->(\&_list_tasks));
    $self->route("POST",   "/api/tasks",          $protected->(\&_create_task));
    $self->route("GET",    "/api/tasks/:id",      $protected->(\&_get_task));
    $self->route("PUT",    "/api/tasks/:id/status",$protected->(\&_update_task_status));
}

sub route {
    my ($self, $method, $path, $handler) = @_;
    my $regex = $path;
    my @params;
    $regex =~ s{:(\w+)}{push @params, $1; "([^/]+)"}ge;
    push @{$self->{routes}}, { method=>$method, path=>$path, re=>qr{^$regex$}, params=>\@params, handler=>$handler };
}

sub dispatch {
    my ($self, $method, $path, %opts) = @_;
    my $req = {
        method  => uc $method,
        path    => $path,
        headers => $opts{headers} // {},
        body    => $opts{body}    // "",
        params  => {},
        ip      => "127.0.0.1",
    };
    
    # Parse JSON body
    if ($req->{body} && ($req->{headers}{"content-type"}//"") =~ /json/i) {
        eval {
            require JSON::PP;
            $req->{json} = JSON::PP->new->decode($req->{body});
        };
    }
    
    for my $route (@{$self->{routes}}) {
        next unless $route->{method} eq $req->{method};
        if (my @caps = $path =~ $route->{re}) {
            @{$req->{params}}{@{$route->{params}}} = @caps;
            my $resp = eval { $route->{handler}->($req) };
            return $resp // { status=>500, body=>Middleware::encode_json({error=>"$@"}) };
        }
    }
    return { status=>404, body=>Middleware::encode_json({error=>"Not found: $path"}) };
}

# Controller handlers
sub _register {
    my $req = shift;
    my $data = $req->{json} // {};
    eval { my $user = AuthService->register(%$data); };
    return $@ ? { status=>400, body=>Middleware::encode_json({error=>"$@"}) }
              : { status=>201, body=>Middleware::encode_json({message=>"Registered"}) };
}

sub _login {
    my $req = shift;
    my $data = $req->{json} // {};
    my $result = eval { AuthService->login($data->{email}//"", $data->{password}//"") };
    return $@ ? { status=>401, body=>Middleware::encode_json({error=>"$@"}) }
              : { status=>200, body=>Middleware::encode_json({
                    token  => $result->{token},
                    user   => { map { $_ => $result->{user}{$_} } qw(id name email role) },
                }) };
}

sub _me {
    my $req = shift;
    my $user = AuthService->get_user($req->{user}{email}) or return {status=>404,body=>""};
    return { status=>200, body=>Middleware::encode_json({map{$_=>$user->{$_}}qw(id name email role)}) };
}

sub _list_projects {
    return { status=>200, body=>Middleware::encode_json([map{$_->{id},$_->{name}} values %_projects]) };
}

sub _create_project {
    my $req = shift;
    my $d   = $req->{json} // {};
    die "name required\n" unless $d->{name};
    my $id = "proj_" . ++$_seq{project};
    $_projects{$id} = { id=>$id, name=>$d->{name}, desc=>$d->{desc}//"",
                        owner=>$req->{user}{sub}, members=>[$req->{user}{sub}],
                        status=>"active", created_at=>time() };
    return { status=>201, body=>Middleware::encode_json($_projects{$id}) };
}

sub _get_project {
    my $req = shift;
    my $id  = $req->{params}{id};
    my $p   = $_projects{$id} or return { status=>404, body=>Middleware::encode_json({error=>"Not found"}) };
    return { status=>200, body=>Middleware::encode_json($p) };
}

sub _update_project {
    my $req = shift;
    my $id  = $req->{params}{id};
    my $p   = $_projects{$id} or return { status=>404, body=>Middleware::encode_json({error=>"Not found"}) };
    my $d   = $req->{json} // {};
    $p->{$_} = $d->{$_} for grep { exists $d->{$_} } qw(name desc status);
    return { status=>200, body=>Middleware::encode_json($p) };
}

sub _delete_project {
    my $req = shift;
    delete $_projects{$req->{params}{id}};
    return { status=>204, body=>"" };
}

sub _list_tasks {
    my $req = shift;
    my $pid = $req->{params}{project_id} // "";
    my @t   = values %_tasks;
    @t = grep { $_->{project_id} eq $pid } @t if $pid;
    return { status=>200, body=>Middleware::encode_json(\@t) };
}

sub _create_task {
    my $req = shift;
    my $d   = $req->{json} // {};
    die "title required\n" unless $d->{title};
    my $id = "task_" . ++$_seq{task};
    $_tasks{$id} = {
        id=>$id, title=>$d->{title}, status=>"todo", priority=>$d->{priority}//"medium",
        project_id=>$d->{project_id}//"", assignee=>$d->{assignee}//"",
        created_by=>$req->{user}{sub}, created_at=>time(),
    };
    return { status=>201, body=>Middleware::encode_json($_tasks{$id}) };
}

sub _get_task {
    my $req = shift;
    my $t   = $_tasks{$req->{params}{id}} or return {status=>404,body=>""};
    return { status=>200, body=>Middleware::encode_json($t) };
}

sub _update_task_status {
    my $req = shift;
    my $t   = $_tasks{$req->{params}{id}} or return {status=>404,body=>""};
    my $d   = $req->{json} // {};
    $t->{status} = $d->{status} // $t->{status};
    $t->{updated_at} = time();
    return { status=>200, body=>Middleware::encode_json($t) };
}
}

package main;

printf "=== TaskFlow REST API Demo ===\n\n";

my $app = App->new;

# 1. Register
printf "1. Register user:\n";
my $r = $app->dispatch("POST", "/auth/register",
    headers => {"content-type"=>"application/json"},
    body    => '{"name":"Alice","email":"alice@test.com","password":"secret123"}',
);
printf "  Status: %d\n\n", $r->{status};

# 2. Login
printf "2. Login:\n";
$r = $app->dispatch("POST", "/auth/login",
    headers => {"content-type"=>"application/json"},
    body    => '{"email":"alice@test.com","password":"secret123"}',
);
printf "  Status: %d\n", $r->{status};
require JSON::PP;
my $login_data = JSON::PP->new->decode($r->{body});
my $token = $login_data->{token};
printf "  Token (first 30): %s...\n\n", substr($token, 0, 30);

my %auth_header = (
    "authorization" => "Bearer $token",
    "content-type"  => "application/json",
);

# 3. Get profile
printf "3. GET /api/me:\n";
$r = $app->dispatch("GET", "/api/me", headers => \%auth_header);
printf "  Status: %d Body: %s\n\n", $r->{status}, $r->{body};

# 4. Create project
printf "4. POST /api/projects:\n";
$r = $app->dispatch("POST", "/api/projects",
    headers => \%auth_header,
    body    => '{"name":"TaskFlow Dev","desc":"Main project"}',
);
printf "  Status: %d\n", $r->{status};
my $proj = JSON::PP->new->decode($r->{body});
printf "  Created project: %s (id=%s)\n\n", $proj->{name}, $proj->{id};

# 5. Create tasks
printf "5. POST /api/tasks:\n";
my @task_bodies = (
    { title=>"Design API",    priority=>"high",     project_id=>$proj->{id} },
    { title=>"Implement auth",priority=>"critical", project_id=>$proj->{id} },
    { title=>"Write tests",   priority=>"medium",   project_id=>$proj->{id} },
);

my @task_ids;
for my $tb (@task_bodies) {
    $r = $app->dispatch("POST", "/api/tasks",
        headers => \%auth_header,
        body    => JSON::PP->new->encode($tb),
    );
    my $t = JSON::PP->new->decode($r->{body});
    push @task_ids, $t->{id};
    printf "  Created: %s (%s)\n", $t->{title}, $t->{id};
}

# 6. Update task status
printf "\n6. Move task to in_progress:\n";
$r = $app->dispatch("PUT", "/api/tasks/$task_ids[0]/status",
    headers => \%auth_header,
    body    => '{"status":"in_progress"}',
);
my $updated = JSON::PP->new->decode($r->{body});
printf "  Status: %d | Task '%s' is now: %s\n\n", $r->{status}, $updated->{title}, $updated->{status};

# 7. List tasks
printf "7. GET /api/tasks:\n";
$r = $app->dispatch("GET", "/api/tasks", headers => \%auth_header);
my $task_list = JSON::PP->new->decode($r->{body});
printf "  Total tasks: %d\n", scalar @$task_list;
printf "  %-20s %-12s %s\n", $_->{title}, $_->{priority}, $_->{status} for @$task_list;

# 8. Delete project
printf "\n8. DELETE /api/projects/%s:\n", $proj->{id};
$r = $app->dispatch("DELETE", "/api/projects/$proj->{id}", headers => \%auth_header);
printf "  Status: %d\n\n", $r->{status};

# 9. Rate limit test
printf "9. Rate limit test (6 rapid requests, burst=5):\n";
for my $i (1..6) {
    $r = $app->dispatch("GET", "/api/me", headers => \%auth_header);
    printf "  Request %d: %d %s\n", $i, $r->{status},
        $r->{status} == 429 ? "(rate limited!)" : "";
}
```

---

## สรุป Part 50 — Capstone: Full-Stack Perl Application

### สิ่งที่สร้าง:
- **Data Models** — User/Project/Task with Repository pattern
- **JWT Auth** — HMAC-SHA256 token generation/verification
- **Auth Service** — Register/login with password hashing
- **Rate Limiter** — Token bucket per-user
- **Middleware Chain** — jwt_auth → rate_limit → request_logger
- **REST Router** — Pattern matching with :params
- **Controllers** — CRUD for projects and tasks
- **Full API Flow** — Register → Login → JWT → CRUD → Rate limit

### หลักสูตรถึง Step 500 แล้ว!
Parts 1-50 ครอบคลุม:
- Perl fundamentals → OOP → CGI → Regex → Network
- Auth, Caching, Jobs, Email, Files, IPC
- Meta API, Types, Roles, CPAN modules
- Full REST API capstone

**ถัดไป: [Part 51 — AnyEvent & Async I/O](part_51.md)**
