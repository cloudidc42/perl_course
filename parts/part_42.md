# Part 42: Background Jobs & Task Queues
## Steps 411-420: Job Processing, Scheduling, Worker Patterns

---

## Step 411: Job Queue Fundamentals

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

my $JSON = JSON::PP->new->utf8->canonical;

{
package Job;

sub new {
    my ($class, %opts) = @_;
    return bless {
        id        => $opts{id}        // _gen_id(),
        type      => $opts{type}      // die("type required"),
        payload   => $opts{payload}   // {},
        priority  => $opts{priority}  // 0,
        status    => "pending",
        attempts  => 0,
        max_attempts => $opts{max_attempts} // 3,
        created_at => time(),
        scheduled_at => $opts{scheduled_at} // time(),
        started_at   => undef,
        finished_at  => undef,
        result       => undef,
        error        => undef,
        queue        => $opts{queue}  // "default",
    }, $class;
}

sub start  { my $s=shift; $s->{status}="running";  $s->{started_at}=time(); $s->{attempts}++ }
sub finish { my ($s,$r)=@_; $s->{status}="done"; $s->{finished_at}=time(); $s->{result}=$r }
sub fail   { my ($s,$e)=@_; $s->{status}="failed";$s->{finished_at}=time(); $s->{error}=$e }
sub retry  { my $s=shift; $s->{status}="pending"; $s->{scheduled_at}=time()+(2**$s->{attempts}) }

sub can_retry { $_[0]->{attempts} < $_[0]->{max_attempts} }
sub is_due    { $_[0]->{scheduled_at} <= time() }
sub duration  { $_[0]->{finished_at} && $_[0]->{started_at}
                ? $_[0]->{finished_at} - $_[0]->{started_at} : undef }

sub to_hash {
    my $self = shift;
    return { map { $_ => $self->{$_} } qw(id type status attempts queue priority) };
}

sub _gen_id { sprintf "job_%08x", int(rand(0xFFFFFFFF)) }
}

{
package JobQueue;

sub new {
    my ($class, %opts) = @_;
    return bless {
        jobs     => {},
        queues   => {},
        handlers => {},
        stats    => { enqueued=>0, completed=>0, failed=>0, retried=>0 },
    }, $class;
}

sub enqueue {
    my ($self, $type, $payload, %opts) = @_;
    my $job = Job->new(type => $type, payload => $payload, %opts);
    $self->{jobs}{$job->{id}} = $job;
    push @{$self->{queues}{$job->{queue}}}, $job->{id};
    $self->{stats}{enqueued}++;
    return $job->{id};
}

sub register {
    my ($self, $type, $handler) = @_;
    $self->{handlers}{$type} = $handler;
    return $self;
}

sub process_one {
    my ($self, $queue) = @_;
    $queue //= "default";
    
    my $queue_list = $self->{queues}{$queue} or return undef;
    
    my ($job_id) = grep {
        my $j = $self->{jobs}{$_};
        $j && $j->{status} eq "pending" && $j->is_due;
    } @$queue_list;
    
    return undef unless $job_id;
    my $job = $self->{jobs}{$job_id};
    
    $job->start;
    
    my $handler = $self->{handlers}{$job->{type}};
    unless ($handler) {
        $job->fail("No handler for job type '$job->{type}'");
        $self->{stats}{failed}++;
        return $job;
    }
    
    eval { $handler->($job->{payload}, $job) };
    if ($@) {
        if ($job->can_retry) {
            $job->retry;
            $self->{stats}{retried}++;
        } else {
            $job->fail($@);
            $self->{stats}{failed}++;
        }
    } else {
        $job->finish($job->{result});
        $self->{stats}{completed}++;
    }
    
    return $job;
}

sub process_all {
    my ($self, $queue) = @_;
    my $count = 0;
    while (my $job = $self->process_one($queue)) {
        $count++;
        last if $count > 10000;  # Safety
    }
    return $count;
}

sub get_job   { $_[0]->{jobs}{$_[1]} }
sub stats     { %{$_[0]->{stats}} }
sub queue_size { grep { $self->{jobs}{$_}{status} eq "pending" } @{$_[0]->{queues}{$_[1]}//[]} }
}

package main;

printf "=== Job Queue ===\n\n";

my $q = JobQueue->new;

# Register handlers
$q->register("send_email", sub {
    my ($payload, $job) = @_;
    printf "  Sending email to %s: %s\n", $payload->{to}, $payload->{subject};
    $job->{result} = { sent => 1, message_id => "msg_" . rand(999) };
});

$q->register("resize_image", sub {
    my ($payload, $job) = @_;
    printf "  Resizing %s to %dx%d\n", $payload->{file}, $payload->{width}, $payload->{height};
    $job->{result} = { output => $payload->{file} . ".resized.jpg" };
});

$q->register("broken_job", sub {
    die "This job always fails!\n";
});

# Enqueue jobs
my @ids = (
    $q->enqueue("send_email",    { to=>"alice\@test.com", subject=>"Welcome!" }),
    $q->enqueue("send_email",    { to=>"bob\@test.com",   subject=>"Newsletter"}, priority=>5),
    $q->enqueue("resize_image",  { file=>"photo.jpg", width=>800, height=>600 }),
    $q->enqueue("resize_image",  { file=>"avatar.png",width=>100, height=>100 }, queue=>"images"),
    $q->enqueue("broken_job",    { data=>"test" }, max_attempts=>2),
    $q->enqueue("unknown_type",  { data=>"x" }),
);

printf "Enqueued %d jobs\n\n", scalar @ids;

# Process
printf "Processing default queue:\n";
$q->process_all("default");

printf "\nProcessing images queue:\n";
$q->process_all("images");

# Report
printf "\n--- Job Report ---\n";
for my $id (@ids) {
    my $j = $q->get_job($id);
    printf "  [%s] type=%-14s status=%-8s attempts=%d %s\n",
        substr($j->{id},4,8), $j->{type}, $j->{status}, $j->{attempts},
        $j->{error} ? "error=".substr($j->{error},0,30) : "";
}

my %stats = $q->stats;
printf "\nStats: enqueued=%d completed=%d failed=%d retried=%d\n",
    @stats{qw(enqueued completed failed retried)};
```

---

## Step 412: Cron-style Scheduler

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Scheduler;

sub new {
    my ($class) = @_;
    return bless {
        tasks     => [],
        _task_id  => 0,
        running   => 0,
        log       => [],
    }, $class;
}

# Schedule with cron expression (simplified: min hour day month weekday)
sub schedule {
    my ($self, $cron, $name, $handler) = @_;
    my $id = ++$self->{_task_id};
    push @{$self->{tasks}}, {
        id      => $id,
        name    => $name,
        cron    => $cron,
        handler => $handler,
        last_run => undef,
        runs     => 0,
        enabled  => 1,
    };
    return $id;
}

# every() convenience methods
sub every_minute  { $_[0]->schedule("* * * * *",    $_[1], $_[2]) }
sub every_hour    { $_[0]->schedule("0 * * * *",    $_[1], $_[2]) }
sub every_day     { $_[0]->schedule("0 0 * * *",    $_[1], $_[2]) }
sub every_week    { $_[0]->schedule("0 0 * * 0",    $_[1], $_[2]) }

sub should_run {
    my ($self, $task, $now) = @_;
    return 0 unless $task->{enabled};
    
    my ($sec, $min, $hour, $mday, $mon, $year, $wday) = localtime($now);
    $mon++; # 1-12
    
    my ($c_min, $c_hour, $c_mday, $c_mon, $c_wday) = split /\s+/, $task->{cron};
    
    return 0 unless _matches($c_min,  $min);
    return 0 unless _matches($c_hour, $hour);
    return 0 unless _matches($c_mday, $mday);
    return 0 unless _matches($c_mon,  $mon);
    return 0 unless _matches($c_wday, $wday);
    
    # Don't run twice in same minute
    return 0 if $task->{last_run} && int($task->{last_run}/60) == int($now/60);
    
    return 1;
}

sub tick {
    my ($self, $time) = @_;
    $time //= time();
    my @ran;
    
    for my $task (@{$self->{tasks}}) {
        if ($self->should_run($task, $time)) {
            eval { $task->{handler}->($task) };
            my $err = $@;
            $task->{last_run} = $time;
            $task->{runs}++;
            push @ran, { id => $task->{id}, name => $task->{name}, error => $err };
            push @{$self->{log}}, { ts=>$time, task=>$task->{name}, err=>$err//undef };
        }
    }
    return @ran;
}

sub simulate {
    my ($self, $from, $to, $step) = @_;
    $step //= 60;
    my @all_ran;
    for (my $t = $from; $t <= $to; $t += $step) {
        my @ran = $self->tick($t);
        push @all_ran, @ran;
    }
    return @all_ran;
}

sub disable { my($s,$id)=@_; $_->{id}==$id && ($_->{enabled}=0) for @{$s->{tasks}} }
sub enable  { my($s,$id)=@_; $_->{id}==$id && ($_->{enabled}=1) for @{$s->{tasks}} }
sub task    { my($s,$id)=@_; (grep{$_->{id}==$id} @{$s->{tasks}})[0] }

sub _matches {
    my ($expr, $val) = @_;
    return 1 if $expr eq "*";
    return $expr == $val if $expr =~ /^\d+$/;
    if ($expr =~ m{^\*/(\d+)$}) { return $val % $1 == 0 }
    if ($expr =~ /^(\d+)-(\d+)$/) { return $val >= $1 && $val <= $2 }
    if ($expr =~ /,/) { return grep { $_ == $val } split /,/, $expr }
    return 0;
}
}

package main;

printf "=== Cron Scheduler ===\n\n";

my $sched = Scheduler->new;
my @output;

# Schedule tasks
$sched->schedule("* * * * *",   "heartbeat",       sub { push @output, "HEARTBEAT at ".localtime(time()) });
$sched->schedule("0 * * * *",   "hourly_cleanup",  sub { push @output, "CLEANUP at ".localtime(time()) });
$sched->schedule("*/5 * * * *", "stats_collect",   sub { push @output, "STATS at ".localtime(time()) });
$sched->schedule("0 9 * * 1-5", "morning_report",  sub { push @output, "MORNING REPORT" });
$sched->schedule("30 23 * * *", "nightly_backup",  sub { push @output, "BACKUP at ".localtime(time()) });

# Fixed reference time: simulate Monday 2024-01-08 08:59:00
# Use epoch that corresponds to a known time
use POSIX qw(mktime);
my $base = mktime(0, 59, 8, 8, 0, 124);  # 2024-01-08 08:59:00

printf "Simulating 90 minutes starting at %s\n\n", scalar localtime($base);

my @ran = $sched->simulate($base, $base + 90*60, 60);

printf "Tasks that ran:\n";
for my $r (@ran) {
    printf "  [%s] %s\n", $r->{error} ? "FAIL" : "OK", $r->{name};
}

printf "\nTask run counts:\n";
for my $task (@{$sched->{tasks}}) {
    printf "  %-20s runs=%d last=%s\n",
        $task->{name}, $task->{runs},
        $task->{last_run} ? scalar(localtime($task->{last_run})) : "never";
}
```

---

## Step 413: Worker Process Management

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Worker::Manager;

sub new {
    my ($class, %opts) = @_;
    return bless {
        min_workers  => $opts{min_workers}  // 2,
        max_workers  => $opts{max_workers}  // 10,
        workers      => {},
        worker_stats => {},
        _next_id     => 1,
        queue_size   => 0,
        jobs_done    => 0,
    }, $class;
}

sub spawn_worker {
    my ($self) = @_;
    return if scalar(keys %{$self->{workers}}) >= $self->{max_workers};
    
    my $id = $self->{_next_id}++;
    $self->{workers}{$id} = {
        id        => $id,
        state     => "idle",
        started   => time(),
        jobs_done => 0,
        current   => undef,
    };
    $self->{worker_stats}{$id} = { total_time => 0, jobs => 0 };
    return $id;
}

sub kill_worker {
    my ($self, $worker_id) = @_;
    return 0 if $self->{workers}{$worker_id}{state} eq "busy";
    delete $self->{workers}{$worker_id};
    return 1;
}

sub assign_job {
    my ($self, $job) = @_;
    my ($idle_worker) = grep { $self->{workers}{$_}{state} eq "idle" } keys %{$self->{workers}};
    return undef unless $idle_worker;
    
    $self->{workers}{$idle_worker}{state}   = "busy";
    $self->{workers}{$idle_worker}{current} = $job;
    return $idle_worker;
}

sub complete_job {
    my ($self, $worker_id, $duration) = @_;
    my $w = $self->{workers}{$worker_id} or return;
    $w->{state}     = "idle";
    $w->{jobs_done}++;
    $w->{current}   = undef;
    $self->{worker_stats}{$worker_id}{total_time} += $duration // 1;
    $self->{worker_stats}{$worker_id}{jobs}++;
    $self->{jobs_done}++;
}

sub scale {
    my ($self) = @_;
    my $total   = scalar keys %{$self->{workers}};
    my $busy    = grep { $self->{workers}{$_}{state} eq "busy" } keys %{$self->{workers}};
    my $load    = $total ? $busy / $total : 0;
    my $backlog = $self->{queue_size};
    
    my @actions;
    
    # Scale up: if load > 80% or backlog > workers
    if (($load > 0.8 || $backlog > $total) && $total < $self->{max_workers}) {
        my $to_add = [int(($self->{max_workers} - $total) / 2), 1]->[0];
        for (1..$to_add) { push @actions, "spawn_" . $self->spawn_worker }
    }
    
    # Scale down: if load < 20% and above minimum
    if ($load < 0.2 && $total > $self->{min_workers}) {
        my $to_remove = $total - $self->{min_workers};
        for my $wid (keys %{$self->{workers}}) {
            last unless $to_remove-- > 0;
            if ($self->kill_worker($wid)) {
                push @actions, "killed_$wid";
            }
        }
    }
    
    return @actions;
}

sub status {
    my $self = shift;
    my $total= scalar keys %{$self->{workers}};
    my $busy = grep { $self->{workers}{$_}{state} eq "busy" } keys %{$self->{workers}};
    return {
        workers  => $total,
        busy     => $busy,
        idle     => $total - $busy,
        load     => $total ? sprintf("%.0f%%", 100*$busy/$total) : "0%",
        jobs_done=> $self->{jobs_done},
        backlog  => $self->{queue_size},
    };
}
}

package main;

printf "=== Worker Process Manager ===\n\n";

my $mgr = Worker::Manager->new(min_workers => 2, max_workers => 8);

# Start minimum workers
printf "Starting minimum workers:\n";
$mgr->spawn_worker for 1..$mgr->{min_workers};
my $s = $mgr->status;
printf "  workers=%d idle=%d busy=%d\n\n", $s->{workers}, $s->{idle}, $s->{busy};

# Simulate load
printf "Simulating load:\n";
my @pending_jobs = map { { id => "job_$_", type => "process", data => $_ } } 1..15;

# Tick simulation
for my $tick (1..10) {
    # Add some jobs to queue
    $mgr->{queue_size} = [15, 12, 8, 4, 1, 0, 0, 3, 6, 0]->[$tick-1] // 0;
    
    # Scale
    my @actions = $mgr->scale;
    
    # Complete some jobs
    my @busy_workers = grep { $mgr->{workers}{$_}{state} eq "busy" } keys %{$mgr->{workers}};
    for my $wid (@busy_workers[0..1]) {
        $mgr->complete_job($wid, 1) if defined $wid;
    }
    
    # Assign queued jobs
    for (1..$mgr->{queue_size}) {
        my $assigned = $mgr->assign_job({ id => "job_t${tick}_$_" });
        # Reduce queue
        $mgr->{queue_size}-- if $assigned;
    }
    
    my $st = $mgr->status;
    printf "  Tick %2d: workers=%d busy=%d idle=%d load=%-4s backlog=%d%s\n",
        $tick, $st->{workers}, $st->{busy}, $st->{idle}, $st->{load}, $st->{backlog},
        @actions ? " [" . join(",", @actions) . "]" : "";
}

my $final = $mgr->status;
printf "\nFinal: workers=%d jobs_done=%d\n", $final->{workers}, $final->{jobs_done};
```

---

## Step 414: Priority Job Queue

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package PriorityQueue;

# Binary heap-based priority queue
sub new {
    my ($class, %opts) = @_;
    return bless {
        heap   => [],
        cmp    => $opts{cmp} // sub { $_[0]{priority} <=> $_[1]{priority} },
    }, $class;
}

sub push {
    my ($self, $item) = @_;
    push @{$self->{heap}}, $item;
    $self->_sift_up($#{$self->{heap}});
    return $self;
}

sub pop {
    my $self = shift;
    return undef unless @{$self->{heap}};
    my $top  = $self->{heap}[0];
    my $last = pop @{$self->{heap}};
    if (@{$self->{heap}}) {
        $self->{heap}[0] = $last;
        $self->_sift_down(0);
    }
    return $top;
}

sub peek { $_[0]->{heap}[0] }
sub size { scalar @{$_[0]->{heap}} }

sub _sift_up {
    my ($self, $i) = @_;
    while ($i > 0) {
        my $parent = int(($i-1)/2);
        if ($self->{cmp}->($self->{heap}[$parent], $self->{heap}[$i]) > 0) {
            @{$self->{heap}}[$parent, $i] = @{$self->{heap}}[$i, $parent];
            $i = $parent;
        } else { last }
    }
}

sub _sift_down {
    my ($self, $i) = @_;
    my $n = $self->size;
    while (1) {
        my ($l, $r, $min) = (2*$i+1, 2*$i+2, $i);
        $min = $l if $l<$n && $self->{cmp}->($self->{heap}[$l],$self->{heap}[$min]) < 0;
        $min = $r if $r<$n && $self->{cmp}->($self->{heap}[$r],$self->{heap}[$min]) < 0;
        last if $min == $i;
        @{$self->{heap}}[$i, $min] = @{$self->{heap}}[$min, $i];
        $i = $min;
    }
}
}

{
package PriorityJobQueue;

sub new {
    my ($class, %opts) = @_;
    return bless {
        queues   => {
            critical => PriorityQueue->new(cmp => sub { $_[0]{created} <=> $_[1]{created} }),
            high     => PriorityQueue->new(cmp => sub { $_[0]{created} <=> $_[1]{created} }),
            normal   => PriorityQueue->new(cmp => sub { $_[0]{created} <=> $_[1]{created} }),
            low      => PriorityQueue->new(cmp => sub { $_[0]{created} <=> $_[1]{created} }),
        },
        handlers => {},
        stats    => { by_priority => {} },
    }, $class;
}

sub enqueue {
    my ($self, $type, $payload, %opts) = @_;
    my $priority = $opts{priority} // "normal";
    my $job = { id => _id(), type => $type, payload => $payload,
                priority => $priority, created => time(), status => "pending" };
    $self->{queues}{$priority}->push($job);
    $self->{stats}{by_priority}{$priority}++;
    return $job->{id};
}

sub dequeue {
    my $self = shift;
    # Process in priority order with starvation prevention
    for my $level (qw(critical high normal low)) {
        my $q = $self->{queues}{$level};
        return $q->pop if $q->size;
    }
    return undef;
}

sub register { $_[0]->{handlers}{$_[1]} = $_[2] }

sub process_one {
    my $self = shift;
    my $job  = $self->dequeue or return undef;
    
    my $handler = $self->{handlers}{$job->{type}};
    unless ($handler) {
        printf "  No handler for '%s'\n", $job->{type};
        return $job;
    }
    
    eval { $job->{result} = $handler->($job->{payload}) };
    $job->{status} = $@ ? "failed" : "done";
    $job->{error}  = $@ if $@;
    return $job;
}

sub process_all { my $self=shift; my @r; push @r, $self->process_one while $self->total_size; return @r }
sub total_size  { my $s=shift; my $t=0; $t+=$_->size for values %{$s->{queues}}; $t }
sub queue_sizes { my $s=shift; map { $_ => $s->{queues}{$_}->size } keys %{$s->{queues}} }

sub _id { sprintf "%08x", int rand 0xFFFFFFFF }
}

package main;

printf "=== Priority Job Queue ===\n\n";

my $pq = PriorityJobQueue->new;

# Register handlers
$pq->register("notify", sub {
    my $p = shift;
    printf "  [%s] Notify %s: %s\n", uc($p->{level}//"?"), $p->{user}, $p->{message};
    return 1;
});

$pq->register("report", sub {
    my $p = shift;
    printf "  [REPORT] Generating %s report\n", $p->{type};
    return { filename => "$p->{type}_report.pdf" };
});

# Enqueue mixed priorities
printf "Enqueueing jobs:\n";
$pq->enqueue("notify", { user=>"admin",level=>"critical",message=>"System down!" }, priority=>"critical");
$pq->enqueue("report", { type=>"daily" },                                           priority=>"low");
$pq->enqueue("notify", { user=>"alice",level=>"warning", message=>"Disk 90% full"}, priority=>"high");
$pq->enqueue("report", { type=>"weekly"},                                           priority=>"low");
$pq->enqueue("notify", { user=>"bob",  level=>"info",    message=>"New user login"},priority=>"normal");
$pq->enqueue("notify", { user=>"ops",  level=>"critical",message=>"DB replication lag"},priority=>"critical");
$pq->enqueue("report", { type=>"monthly"},                                          priority=>"low");
$pq->enqueue("notify", { user=>"carol",level=>"warning", message=>"High memory"},   priority=>"high");

my %sizes = $pq->queue_sizes;
printf "Queue sizes: critical=%d high=%d normal=%d low=%d\n\n",
    $sizes{critical}//$pq->{queues}{critical}->size,
    $sizes{high}//$pq->{queues}{high}->size,
    $sizes{normal}//$pq->{queues}{normal}->size,
    $sizes{low}//$pq->{queues}{low}->size;

printf "Processing (priority order):\n";
my @results = $pq->process_all;
printf "\nProcessed %d jobs total\n", scalar @results;
```

---

## Step 415: Retry & Exponential Backoff

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Retry;

sub new {
    my ($class, %opts) = @_;
    return bless {
        max_attempts  => $opts{max_attempts}  // 3,
        base_delay    => $opts{base_delay}    // 1,
        max_delay     => $opts{max_delay}     // 60,
        multiplier    => $opts{multiplier}    // 2,
        jitter        => $opts{jitter}        // 1,
        retryable     => $opts{retryable}     // sub { 1 },
        on_retry      => $opts{on_retry}      // sub {},
    }, $class;
}

sub execute {
    my ($self, $fn) = @_;
    my $attempts = 0;
    my @errors;
    
    while ($attempts < $self->{max_attempts}) {
        $attempts++;
        my $result = eval { $fn->() };
        
        unless ($@) {
            return { ok => 1, result => $result, attempts => $attempts };
        }
        
        my $err = $@;
        push @errors, { attempt => $attempts, error => $err };
        
        last unless $self->{retryable}->($err);
        last if $attempts >= $self->{max_attempts};
        
        my $delay = $self->delay_for($attempts);
        $self->{on_retry}->($attempts, $err, $delay);
        
        # Simulate delay (skip actual sleep in demo)
        # sleep($delay);
    }
    
    return { ok => 0, attempts => $attempts, errors => \@errors };
}

sub delay_for {
    my ($self, $attempt) = @_;
    my $delay = $self->{base_delay} * ($self->{multiplier} ** ($attempt-1));
    $delay = $self->{max_delay} if $delay > $self->{max_delay};
    
    if ($self->{jitter}) {
        $delay = $delay * (0.5 + rand(0.5));  # 50-100% of computed delay
    }
    
    return $delay;
}

sub with_context {
    my ($self, %ctx) = @_;
    return sub {
        my $fn = shift;
        $self->execute(sub { $fn->(%ctx) });
    };
}
}

{
package RetryPolicy;

# Predefined retry policies
sub transient_errors {
    return Retry->new(
        max_attempts => 5,
        base_delay   => 0.1,
        max_delay    => 30,
        multiplier   => 2,
        jitter       => 1,
        retryable    => sub {
            my $err = shift;
            return 1 if $err =~ /timeout|connection refused|temporarily unavailable/i;
            return 0;  # Don't retry permanent errors
        },
    );
}

sub network_errors {
    return Retry->new(
        max_attempts => 4,
        base_delay   => 0.5,
        max_delay    => 60,
        multiplier   => 3,
        on_retry => sub {
            my ($attempt, $err, $delay) = @_;
            printf "  Retry %d after %.2fs (err: %s)\n",
                $attempt, $delay, substr($err,0,40);
        },
    );
}

sub database_deadlock {
    return Retry->new(
        max_attempts => 3,
        base_delay   => 0.1,
        max_delay    => 1,
        multiplier   => 2,
        retryable => sub { $_[0] =~ /deadlock|lock.*timeout/i },
    );
}
}

package main;

printf "=== Retry & Exponential Backoff ===\n\n";

# Show backoff schedule
printf "Backoff schedule (base=1s, mult=2, max=30s):\n";
my $retry = Retry->new(base_delay=>1, multiplier=>2, max_delay=>30, jitter=>0);
for my $attempt (1..8) {
    printf "  Attempt %d: delay=%.1fs\n", $attempt, $retry->delay_for($attempt);
}
printf "\n";

# Simulate flaky service
my $call_count = 0;
my $fail_until = 3;

printf "Retrying flaky service (fails first %d times):\n", $fail_until;
my $net_retry = RetryPolicy->network_errors;

my $result = $net_retry->execute(sub {
    $call_count++;
    printf "  Attempt %d\n", $call_count;
    die "connection refused\n" if $call_count <= $fail_until;
    return { data => "success_response", call_num => $call_count };
});

if ($result->{ok}) {
    printf "  Succeeded on attempt %d!\n", $result->{attempts};
    printf "  Result: data=%s\n\n", $result->{result}{data};
} else {
    printf "  All %d attempts failed\n\n", $result->{attempts};
}

# Non-retryable error
printf "Non-retryable error (permission denied):\n";
my $perm_retry = RetryPolicy->transient_errors;
my $perm_result = $perm_retry->execute(sub {
    die "permission denied\n";
});
printf "  Stopped at attempt %d (non-retryable)\n\n", $perm_result->{attempts};

# Deadlock retry
printf "Database deadlock retry:\n";
my $deadlock_calls = 0;
my $db_retry = RetryPolicy->database_deadlock;
my $db_result = $db_retry->execute(sub {
    $deadlock_calls++;
    die "deadlock detected\n" if $deadlock_calls < 2;
    return { rows => 5 };
});
printf "  Result: %s after %d attempts\n",
    $db_result->{ok} ? "success ($db_result->{result}{rows} rows)" : "failed",
    $db_result->{attempts};
```

---

## Step 416: Dead Letter Queue

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package DLQ;

sub new {
    my ($class, %opts) = @_;
    return bless {
        queue           => [],
        max_size        => $opts{max_size}       // 1000,
        retention_days  => $opts{retention_days} // 7,
        alert_threshold => $opts{alert_threshold}// 10,
        alerts          => [],
    }, $class;
}

sub push_dead {
    my ($self, $job, $reason) = @_;
    
    if (@{$self->{queue}} >= $self->{max_size}) {
        shift @{$self->{queue}};  # Remove oldest
    }
    
    push @{$self->{queue}}, {
        job       => $job,
        reason    => $reason,
        failed_at => time(),
        attempts  => $job->{attempts} // 0,
    };
    
    # Alert if threshold breached
    if (@{$self->{queue}} >= $self->{alert_threshold}) {
        push @{$self->{alerts}}, {
            message => sprintf("DLQ has %d dead messages!", scalar @{$self->{queue}}),
            ts      => time(),
        };
    }
    
    return $self;
}

sub pop_dead {
    my ($self) = @_;
    return shift @{$self->{queue}};
}

sub replay {
    my ($self, $target_queue, $filter) = @_;
    my @replayed;
    my @remaining;
    
    for my $dead (@{$self->{queue}}) {
        if (!$filter || $filter->($dead)) {
            push @replayed, $dead;
            $target_queue->enqueue(
                $dead->{job}{type},
                $dead->{job}{payload},
                max_attempts => ($dead->{job}{max_attempts}//3) + 1,
            ) if ref $target_queue;
        } else {
            push @remaining, $dead;
        }
    }
    
    $self->{queue} = \@remaining;
    return scalar @replayed;
}

sub cleanup_old {
    my $self = shift;
    my $cutoff = time() - $self->{retention_days} * 86400;
    my $before = scalar @{$self->{queue}};
    @{$self->{queue}} = grep { $_->{failed_at} > $cutoff } @{$self->{queue}};
    return $before - scalar @{$self->{queue}};
}

sub size        { scalar @{$_[0]->{queue}} }
sub errors_by_type {
    my $self = shift;
    my %cnt;
    $cnt{$_->{job}{type}}++ for @{$self->{queue}};
    return %cnt;
}
sub recent_alerts { @{$_[0]->{alerts}} }
}

package main;

printf "=== Dead Letter Queue ===\n\n";

my $dlq = DLQ->new(max_size => 100, alert_threshold => 3);

# Simulate failed jobs
my @failed_jobs = (
    { id=>"j1", type=>"send_email",   payload=>{to=>"bad\@x.invalid"}, attempts=>3 },
    { id=>"j2", type=>"process_file", payload=>{file=>"missing.csv"},   attempts=>3 },
    { id=>"j3", type=>"send_email",   payload=>{to=>"another\@bad"},    attempts=>3 },
    { id=>"j4", type=>"send_webhook", payload=>{url=>"http://down"},     attempts=>5 },
    { id=>"j5", type=>"send_email",   payload=>{to=>"third\@bad"},       attempts=>3 },
);

for my $job (@failed_jobs) {
    $dlq->push_dead($job, "Max retries exceeded");
    printf "  Dead: [%s] %s\n", $job->{type}, $job->{id};
}

printf "\nDLQ size: %d\n", $dlq->size;

my %by_type = $dlq->errors_by_type;
printf "Errors by type:\n";
printf "  %s: %d\n", $_, $by_type{$_} for sort keys %by_type;

my @alerts = $dlq->recent_alerts;
printf "\nAlerts triggered: %d\n", scalar @alerts;
printf "  %s\n", $_->{message} for @alerts;

# Replay email failures
printf "\nReplaying email failures:\n";

{
package MockQueue;
sub new { bless { jobs=>[] }, shift }
sub enqueue { my($s,$t,$p,%o)=@_; push @{$s->{jobs}},{type=>$t,payload=>$p,%o}; "id_".rand(999) }
sub size { scalar @{$_[0]->{jobs}} }
}

my $main_queue = MockQueue->new;
my $replayed = $dlq->replay($main_queue, sub {
    $_[0]->{job}{type} eq "send_email"
});

printf "  Replayed %d email jobs\n", $replayed;
printf "  Remaining in DLQ: %d\n", $dlq->size;
printf "  Jobs added back to main queue: %d\n\n", $main_queue->size;

# Cleanup old messages
$dlq->{queue}[0]{failed_at} = time() - 8*86400 if $dlq->size;
my $cleaned = $dlq->cleanup_old;
printf "Cleaned %d old messages (>7 days)\n", $cleaned;
```

---

## Step 417: Job Batching & Chunking

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package BatchProcessor;

sub new {
    my ($class, %opts) = @_;
    return bless {
        batch_size  => $opts{batch_size}  // 100,
        concurrency => $opts{concurrency} // 4,
        on_batch    => $opts{on_batch},
        on_error    => $opts{on_error}    // sub {},
        stats       => { batches=>0, items=>0, errors=>0, skipped=>0 },
    }, $class;
}

sub process {
    my ($self, @items) = @_;
    my $size    = $self->{batch_size};
    my @batches;
    
    # Split into batches
    while (@items) {
        push @batches, [splice @items, 0, $size];
    }
    
    printf "  Processing %d items in %d batches of %d\n",
        $self->{stats}{items} + scalar(@{$batches[0]}//$batches[-1]//[]) * scalar @batches,
        scalar @batches, $size;
    
    for my $i (0..$#batches) {
        my $batch = $batches[$i];
        $self->{stats}{batches}++;
        $self->{stats}{items} += scalar @$batch;
        
        eval {
            my $result = $self->{on_batch}->($batch, {
                batch_num => $i+1,
                total     => scalar @batches,
                size      => scalar @$batch,
            });
            $self->{stats}{skipped} += $result->{skipped} // 0 if ref $result;
        };
        if ($@) {
            $self->{stats}{errors}++;
            $self->{on_error}->($@, $batch, $i+1);
        }
    }
    
    return %{$self->{stats}};
}
}

{
package Chunker;
# Lazy chunking iterator

sub new {
    my ($class, $items, $size) = @_;
    return bless { items => $items, size => $size, offset => 0 }, $class;
}

sub has_next { $_[0]->{offset} < scalar @{$_[0]->{items}} }

sub next {
    my $self = shift;
    return undef unless $self->has_next;
    my @chunk = @{$self->{items}}[$self->{offset}..[$self->{offset}+$self->{size}-1,
                                  $#{$self->{items}}]->[0]];
    $self->{offset} += $self->{size};
    return \@chunk;
}

sub total_chunks {
    my $self = shift;
    return int((scalar @{$self->{items}} + $self->{size} - 1) / $self->{size});
}
}

package main;

printf "=== Batch Processing & Chunking ===\n\n";

# Create sample data
my @emails = map { { id=>$_, to=>"user${_}\@test.com", subject=>"Newsletter" } } 1..250;

my $processor = BatchProcessor->new(
    batch_size => 50,
    on_batch   => sub {
        my ($batch, $meta) = @_;
        printf "  Batch %d/%d: sending %d emails (ids %d-%d)\n",
            $meta->{batch_num}, $meta->{total}, $meta->{size},
            $batch->[0]{id}, $batch->[-1]{id};
        return { sent => scalar @$batch };
    },
    on_error => sub {
        my ($err, $batch, $num) = @_;
        printf "  BATCH ERROR %d: %s\n", $num, $err;
    },
);

printf "Email batch processing:\n";
my %stats = $processor->process(@emails);
printf "  Total: batches=%d items=%d errors=%d\n\n",
    $stats{batches}, $stats{items}, $stats{errors};

# Chunker
printf "Lazy chunking iterator:\n";
my @records = 1..33;
my $chunker = Chunker->new(\@records, 10);
printf "  Total chunks: %d\n", $chunker->total_chunks;

while ($chunker->has_next) {
    my $chunk = $chunker->next;
    printf "  Chunk: [%s]\n", join(",", @$chunk);
}

# Database batch insert
printf "\nDatabase batch insert simulation:\n";
my @rows = map { { name=>"User$_", score=>int(rand(100)) } } 1..1000;
my $batch_size = 100;
my $batches    = int(scalar(@rows) / $batch_size) + (scalar(@rows) % $batch_size ? 1 : 0);
my $inserted   = 0;

for my $b (0..$batches-1) {
    my @batch = @rows[$b*$batch_size..[$b*$batch_size+$batch_size-1, $#rows]->[0]];
    # Simulate: INSERT INTO users (name,score) VALUES (?,?),... 
    $inserted += scalar @batch;
}
printf "  Inserted %d rows in %d batches of %d\n", $inserted, $batches, $batch_size;
```

---

## Step 418: Rate Limiting for Jobs

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package RateLimiter;

sub new {
    my ($class, %opts) = @_;
    return bless {
        rate      => $opts{rate}      // 10,   # permits per period
        period    => $opts{period}    // 1,    # seconds
        burst     => $opts{burst}     // 0,    # extra burst capacity
        _tokens   => undef,
        _last_ts  => time(),
        stats     => { allowed=>0, throttled=>0 },
    }, $class;
}

sub _refill {
    my $self = shift;
    my $now  = time();
    my $elapsed = $now - $self->{_last_ts};
    $self->{_last_ts} = $now;
    
    my $max = $self->{rate} + $self->{burst};
    $self->{_tokens} //= $max;
    $self->{_tokens} += $elapsed * ($self->{rate} / $self->{period});
    $self->{_tokens} = $max if $self->{_tokens} > $max;
}

sub try_acquire {
    my ($self, $n) = @_;
    $n //= 1;
    $self->_refill;
    
    if ($self->{_tokens} >= $n) {
        $self->{_tokens} -= $n;
        $self->{stats}{allowed}++;
        return 1;
    }
    $self->{stats}{throttled}++;
    return 0;
}

sub wait_time {
    my ($self, $n) = @_;
    $n //= 1;
    $self->_refill;
    return 0 if $self->{_tokens} >= $n;
    return ($n - $self->{_tokens}) / ($self->{rate} / $self->{period});
}

sub stats { %{$_[0]->{stats}} }
}

{
package JobThrottle;

sub new {
    my ($class, %opts) = @_;
    return bless {
        limiters => {},
        defaults => { rate => $opts{rate}//10, period => $opts{period}//1 },
    }, $class;
}

sub add_limit {
    my ($self, $key, %opts) = @_;
    $self->{limiters}{$key} = RateLimiter->new(%opts);
}

sub check {
    my ($self, $key) = @_;
    my $limiter = $self->{limiters}{$key} //= RateLimiter->new(%{$self->{defaults}});
    return $limiter->try_acquire;
}

sub check_global { $_[0]->{global} //= RateLimiter->new(%{$_[0]->{defaults}}); $_[0]->{global}->try_acquire }
}

package main;

printf "=== Rate Limiting for Jobs ===\n\n";

# Simple rate limiter
my $rl = RateLimiter->new(rate => 5, period => 1, burst => 2);

printf "Token bucket (rate=5/s, burst=2):\n";
printf "Initial tokens: %.1f\n", $rl->{_tokens} // ($rl->{rate} + $rl->{burst});
$rl->_refill;

my ($allowed, $throttled) = (0, 0);
for my $i (1..10) {
    if ($rl->try_acquire) {
        $allowed++;
        printf "  Request %2d: ALLOW (tokens=%.1f)\n", $i, $rl->{_tokens};
    } else {
        $throttled++;
        printf "  Request %2d: DENY  (tokens=%.1f, wait=%.2fs)\n", $i, $rl->{_tokens}, $rl->wait_time;
    }
}
printf "  Allowed=%d Throttled=%d\n\n", $allowed, $throttled;

# Per-client throttling
my $throttle = JobThrottle->new(rate => 3, period => 1);
$throttle->add_limit("email_sender",   rate => 2, period => 1);
$throttle->add_limit("report_builder", rate => 1, period => 1);

printf "Per-job-type throttling:\n";
my %results;
for my $job_type (qw(email_sender email_sender email_sender
                     report_builder report_builder
                     file_processor file_processor file_processor file_processor)) {
    my $ok = $throttle->check($job_type);
    $results{$job_type}{$ok ? "ok" : "deny"}++;
    printf "  %-20s: %s\n", $job_type, $ok ? "ALLOWED" : "THROTTLED";
}

printf "\nSummary:\n";
for my $type (sort keys %results) {
    printf "  %-20s ok=%d deny=%d\n", $type,
        $results{$type}{ok}//0, $results{$type}{deny}//0;
}
```

---

## Step 419: Job Monitoring & Metrics

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package JobMonitor;

sub new {
    my ($class) = @_;
    return bless {
        jobs     => {},
        timeline => [],
        alerts   => [],
        thresholds => {
            max_duration  => 300,  # 5 min
            max_retries   => 5,
            error_rate    => 0.1,  # 10%
            queue_depth   => 100,
        },
    }, $class;
}

sub record_start {
    my ($self, $job_id, %meta) = @_;
    $self->{jobs}{$job_id} = {
        id        => $job_id,
        %meta,
        started_at => time(),
        status    => "running",
    };
    push @{$self->{timeline}}, { ts=>time(), event=>"start", job_id=>$job_id, %meta };
}

sub record_end {
    my ($self, $job_id, %result) = @_;
    my $job = $self->{jobs}{$job_id} or return;
    $job->{finished_at} = time();
    $job->{duration}    = $job->{finished_at} - $job->{started_at};
    $job->{status}      = $result{ok} ? "done" : "failed";
    $job->{result}      = $result{result};
    $job->{error}       = $result{error};
    
    push @{$self->{timeline}}, { ts=>time(), event=>"end", job_id=>$job_id, %result };
    
    # Check thresholds
    if ($job->{duration} > $self->{thresholds}{max_duration}) {
        $self->_alert("slow_job", "Job $job_id took $job->{duration}s (max=$self->{thresholds}{max_duration}s)");
    }
}

sub _alert {
    my ($self, $type, $msg) = @_;
    push @{$self->{alerts}}, { type=>$type, message=>$msg, ts=>time() };
    printf "  ALERT [%s]: %s\n", $type, $msg;
}

sub summary {
    my $self = shift;
    my @jobs = values %{$self->{jobs}};
    return {
        total    => scalar @jobs,
        done     => scalar(grep { $_->{status} eq "done"    } @jobs),
        failed   => scalar(grep { $_->{status} eq "failed"  } @jobs),
        running  => scalar(grep { $_->{status} eq "running" } @jobs),
        avg_duration => do {
            my @d = map { $_->{duration}//0 } grep { $_->{status} ne "running" } @jobs;
            @d ? (my $s = 0, $s+=$_ for @d, $s/@d) : 0;
        },
    };
}

sub throughput {
    my ($self, $window) = @_;
    $window //= 60;
    my $cutoff = time() - $window;
    my @recent = grep { $_->{ts} > $cutoff && $_->{event} eq "end" } @{$self->{timeline}};
    return scalar(@recent) / $window;
}

sub error_rate {
    my $self = shift;
    my @finished = grep { $_->{status} ne "running" } values %{$self->{jobs}};
    return 0 unless @finished;
    my $failed = grep { $_->{status} eq "failed" } @finished;
    return $failed / scalar @finished;
}
}

package main;

printf "=== Job Monitoring ===\n\n";

my $monitor = JobMonitor->new;
my $sim_time = time();

# Simulate jobs
my @scenarios = (
    { id=>"j001", type=>"email",    ok=>1, duration=>0.1 },
    { id=>"j002", type=>"report",   ok=>1, duration=>2.5 },
    { id=>"j003", type=>"email",    ok=>0, error=>"SMTP timeout" },
    { id=>"j004", type=>"import",   ok=>1, duration=>45  },
    { id=>"j005", type=>"export",   ok=>1, duration=>3.2 },
    { id=>"j006", type=>"email",    ok=>1, duration=>0.2 },
    { id=>"j007", type=>"cleanup",  ok=>0, error=>"DB connection failed" },
    { id=>"j008", type=>"sync",     ok=>1, duration=>400 },  # Slow — triggers alert
    { id=>"j009", type=>"index",    ok=>1, duration=>12  },
    { id=>"j010", type=>"archive",  ok=>1, duration=>8   },
);

printf "Simulating %d jobs:\n", scalar @scenarios;
for my $s (@scenarios) {
    $monitor->record_start($s->{id}, type => $s->{type});
    
    if ($s->{ok}) {
        # Manually set duration for simulation
        $monitor->{jobs}{$s->{id}}{started_at} = time() - ($s->{duration}//1);
        $monitor->record_end($s->{id}, ok=>1, result=>"success");
    } else {
        $monitor->record_end($s->{id}, ok=>0, error=>$s->{error});
    }
}

my $sum = $monitor->summary;
printf "\nSummary:\n";
printf "  total=%d done=%d failed=%d running=%d\n",
    $sum->{total}, $sum->{done}, $sum->{failed}, $sum->{running};
printf "  avg_duration=%.1fs\n", $sum->{avg_duration};
printf "  error_rate=%.1f%%\n", $monitor->error_rate * 100;
printf "  throughput=%.2f jobs/s (last 60s)\n", $monitor->throughput(60);

printf "\nAlerts: %d\n", scalar @{$monitor->{alerts}};
printf "  [%s] %s\n", $_->{type}, $_->{message} for @{$monitor->{alerts}};
```

---

## Step 420: Capstone — Complete Job Processing System

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

my $JSON = JSON::PP->new->utf8->canonical;

# Full job processing: queue, workers, scheduler, retry, DLQ, monitoring

my @job_store;
my @dlq_store;
my %handlers;
my @schedule;
my @log;

my $job_seq = 0;

sub enqueue_job {
    my (%opts) = @_;
    my $job = {
        id        => sprintf("JOB%04d", ++$job_seq),
        type      => $opts{type}       // "unknown",
        payload   => $opts{payload}    // {},
        priority  => $opts{priority}   // 5,
        queue     => $opts{queue}      // "default",
        status    => "pending",
        attempts  => 0,
        max_attempts => $opts{max_attempts} // 3,
        created   => time(),
        next_run  => time() + ($opts{delay}//0),
    };
    push @job_store, $job;
    push @log, { ts=>time(), event=>"enqueued", job_id=>$job->{id}, type=>$job->{type} };
    return $job->{id};
}

sub register_handler {
    my ($type, $fn) = @_;
    $handlers{$type} = $fn;
}

sub process_jobs {
    my (%opts) = @_;
    my $queue  = $opts{queue} // "default";
    my $limit  = $opts{limit} // 100;
    my $done   = 0;
    
    my @pending = sort { $b->{priority} <=> $a->{priority} }
                  grep { $_->{status} eq "pending"
                         && $_->{queue} eq $queue
                         && $_->{next_run} <= time() } @job_store;
    
    for my $job (@pending) {
        last if $done >= $limit;
        
        $job->{status}    = "running";
        $job->{started}   = time();
        $job->{attempts}++;
        
        my $handler = $handlers{$job->{type}};
        unless ($handler) {
            $job->{status} = "failed";
            $job->{error}  = "No handler for '$job->{type}'";
            push @dlq_store, { %$job, dlq_reason => $job->{error} };
            push @log, { ts=>time(), event=>"dead", job_id=>$job->{id} };
            $done++;
            next;
        }
        
        my $result = eval { $handler->($job->{payload}) };
        if ($@) {
            if ($job->{attempts} < $job->{max_attempts}) {
                $job->{status}   = "pending";
                $job->{next_run} = time() + (2 ** $job->{attempts});
                push @log, { ts=>time(), event=>"retry", job_id=>$job->{id}, attempt=>$job->{attempts} };
            } else {
                $job->{status} = "failed";
                $job->{error}  = $@;
                push @dlq_store, { %$job, dlq_reason => $job->{error} };
                push @log, { ts=>time(), event=>"dead", job_id=>$job->{id} };
            }
        } else {
            $job->{status}   = "done";
            $job->{result}   = $result;
            $job->{finished} = time();
            push @log, { ts=>time(), event=>"done", job_id=>$job->{id} };
        }
        $done++;
    }
    return $done;
}

sub job_stats {
    my %by_status;
    $by_status{$_->{status}}++ for @job_store;
    return %by_status;
}

# --- Register handlers ---
printf "=== Complete Job Processing System ===\n\n";

register_handler("send_email", sub {
    my $p = shift;
    printf "  EMAIL: to=%s subject='%s'\n", $p->{to}, $p->{subject};
    die "invalid email\n" unless $p->{to} =~ /\@/;
    return { sent_at => time() };
});

register_handler("generate_report", sub {
    my $p = shift;
    printf "  REPORT: type=%s period=%s\n", $p->{type}, $p->{period}//"all";
    return { filename => "$p->{type}_report.pdf", size => 1024 };
});

register_handler("process_payment", sub {
    my $p = shift;
    printf "  PAYMENT: amount=\$%.2f user=%s\n", $p->{amount}, $p->{user_id};
    die "card declined\n" if ($p->{card}//"") eq "declined";
    return { transaction_id => sprintf "TXN%08d", rand(99999999) };
});

register_handler("sync_inventory", sub {
    my $p = shift;
    printf "  SYNC: source=%s\n", $p->{source};
    return { synced => 100 + int(rand(50)) };
});

# --- Enqueue jobs ---
printf "Enqueueing jobs:\n";
enqueue_job(type=>"send_email",    payload=>{to=>"alice\@test.com",    subject=>"Welcome"},       priority=>8);
enqueue_job(type=>"send_email",    payload=>{to=>"INVALID_NO_AT_SIGN", subject=>"Should fail"},   priority=>8);
enqueue_job(type=>"generate_report",payload=>{type=>"daily",period=>"today"},                     priority=>5);
enqueue_job(type=>"process_payment",payload=>{amount=>99.99,user_id=>1},                          priority=>9);
enqueue_job(type=>"process_payment",payload=>{amount=>50.00,user_id=>2,card=>"declined"},         priority=>9);
enqueue_job(type=>"sync_inventory", payload=>{source=>"erp"},                                     queue=>"sync");
enqueue_job(type=>"generate_report",payload=>{type=>"monthly"},                                   priority=>3);
enqueue_job(type=>"unknown_type",   payload=>{},                                                  priority=>1);

printf "\nProcessing default queue:\n";
my $count = process_jobs(queue=>"default");
printf "Processed %d jobs\n\n", $count;

printf "Processing sync queue:\n";
process_jobs(queue=>"sync");

# Second pass for retries
printf "\nRetry pass:\n";
process_jobs(queue=>"default");

# Stats
my %stats = job_stats();
printf "\n=== Final Statistics ===\n";
printf "  %-10s: %d\n", $_, $stats{$_}//0 for qw(done failed pending running);
printf "  DLQ: %d dead messages\n", scalar @dlq_store;
printf "\nDLQ contents:\n";
for my $d (@dlq_store) {
    printf "  [%s] type=%-20s reason=%s\n", $d->{id}, $d->{type}, substr($d->{dlq_reason}//"-",0,40);
}

printf "\nEvent log (%d events):\n", scalar @log;
for my $e (@log) {
    printf "  [%s] %s (job=%s)\n", $e->{event}, $e->{type}//"", $e->{job_id};
}
```

---

## สรุป Part 42 — Background Jobs & Task Queues

### สิ่งที่เรียนรู้:
- **Job Queue** — Structured job lifecycle, multi-queue, handler registry
- **Cron Scheduler** — Expression parsing, simulation, every_*() helpers
- **Worker Manager** — Spawn/kill, autoscaling based on load
- **Priority Queue** — Binary heap, critical/high/normal/low levels
- **Retry & Backoff** — Exponential backoff, jitter, policy presets
- **Dead Letter Queue** — Retention, replay, tag filtering, alerts
- **Batching & Chunking** — Lazy iterator, DB batch insert
- **Rate Limiting** — Token bucket, per-job-type limits
- **Monitoring** — Duration, throughput, error rate, threshold alerts
- **Capstone** — Complete system with retry, DLQ, worker stats

**ถัดไป: [Part 43 — Email & MIME Processing](part_43.md)**
