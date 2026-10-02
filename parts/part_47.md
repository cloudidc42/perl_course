# Part 47: Process Management & IPC
## Steps 461-470: fork, exec, pipes, signals, shared memory

---

## Step 461: fork & exec

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(:sys_wait_h);

printf "=== fork & exec ===\n\n";

# Basic fork pattern
{
package ProcessManager;

sub spawn {
    my ($class, %opts) = @_;
    my $pid = fork();
    die "fork failed: $!" unless defined $pid;
    
    if ($pid == 0) {
        # Child process
        eval { $opts{child}->() } if $opts{child};
        exit $@ ? 1 : 0;
    }
    
    # Parent
    return $pid;
}

sub wait_for {
    my ($class, @pids) = @_;
    my %results;
    for my $pid (@pids) {
        my $waited = waitpid($pid, 0);
        if ($waited == $pid) {
            my $exit_code = WIFEXITED($?) ? WEXITSTATUS($?) : -1;
            my $signal    = WIFSIGNALED($?) ? WTERMSIG($?) : 0;
            $results{$pid} = { exit => $exit_code, signal => $signal };
        }
    }
    return %results;
}

sub wait_any {
    my ($class) = @_;
    my $pid = wait();
    return undef if $pid <= 0;
    return { pid => $pid, exit => WIFEXITED($?) ? WEXITSTATUS($?) : -1 };
}

sub spawn_workers {
    my ($class, $count, $worker_fn) = @_;
    my @pids;
    for my $i (1..$count) {
        push @pids, $class->spawn(child => sub { $worker_fn->($i) });
    }
    return @pids;
}
}

# Demonstrate: fork N workers, collect results via pipe
printf "Spawning 3 worker processes:\n";

my @pipes;
my @pids;

for my $i (1..3) {
    pipe(my $read_fh, my $write_fh);
    my $pid = fork();
    die "fork: $!" unless defined $pid;
    
    if ($pid == 0) {
        # Child: close read end, compute, write result
        close $read_fh;
        my $result = $i * $i;
        sleep($i * 0);  # No sleep needed in simulation
        print $write_fh "worker $i result: $result\n";
        close $write_fh;
        exit 0;
    }
    
    # Parent: close write end
    close $write_fh;
    push @pipes, $read_fh;
    push @pids, $pid;
}

# Collect results
for my $fh (@pipes) {
    while (<$fh>) {
        chomp;
        printf "  Received: %s\n", $_;
    }
    close $fh;
}

# Wait for all children
for my $pid (@pids) {
    waitpid($pid, 0);
    my $exit = WIFEXITED($?) ? WEXITSTATUS($?) : -1;
    printf "  PID %d exited with code %d\n", $pid, $exit;
}

# exec example (simulation)
printf "\nexec simulation (would replace process image):\n";
printf "  exec('ls', '-la', '/tmp') would run: ls -la /tmp\n";
printf "  (not executed to keep output clean)\n";
```

---

## Step 462: Pipes & IPC

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Pipe;

sub new {
    my ($class) = @_;
    pipe(my $r, my $w) or die "pipe: $!";
    return bless { reader => $r, writer => $w }, $class;
}

sub read_end  { $_[0]->{reader} }
sub write_end { $_[0]->{writer} }
sub close_read  { close $_[0]->{reader} }
sub close_write { close $_[0]->{writer} }

sub send_data {
    my ($self, $data) = @_;
    my $len = length($data);
    my $hdr = pack("N", $len);
    print { $self->{writer} } $hdr . $data;
}

sub recv_data {
    my $self = shift;
    my $hdr;
    read($self->{reader}, $hdr, 4) or return undef;
    my $len = unpack("N", $hdr);
    my $data;
    read($self->{reader}, $data, $len) or return undef;
    return $data;
}
}

{
package ChildProcess;

sub run_command {
    my ($class, $cmd, @args) = @_;
    
    my ($child_in_r, $child_in_w);
    my ($child_out_r, $child_out_w);
    my ($child_err_r, $child_err_w);
    
    pipe($child_in_r,  $child_in_w)  or die "pipe stdin:  $!";
    pipe($child_out_r, $child_out_w) or die "pipe stdout: $!";
    pipe($child_err_r, $child_err_w) or die "pipe stderr: $!";
    
    my $pid = fork();
    die "fork: $!" unless defined $pid;
    
    if ($pid == 0) {
        close $child_in_w; close $child_out_r; close $child_err_r;
        open(STDIN,  "<&", $child_in_r)  or die;
        open(STDOUT, ">&", $child_out_w) or die;
        open(STDERR, ">&", $child_err_w) or die;
        exec($cmd, @args) or die "exec: $!";
    }
    
    close $child_in_r; close $child_out_w; close $child_err_w;
    
    return {
        pid    => $pid,
        stdin  => $child_in_w,
        stdout => $child_out_r,
        stderr => $child_err_r,
    };
}

sub capture {
    my ($class, @cmd) = @_;
    
    # Use open3-style with backtick simulation
    my $output = "";
    eval {
        local $/;
        open(my $fh, "-|", @cmd) or die "Cannot run: $!";
        $output = <$fh>;
        close $fh;
    };
    return $output // "";
}
}

package main;

printf "=== Pipes & IPC ===\n\n";

# Basic pipe
printf "Basic pipe communication:\n";
my $pipe = Pipe->new;

my $pid = fork();
die "fork: $!" unless defined $pid;

if ($pid == 0) {
    $pipe->close_read;
    $pipe->send_data("Hello from child!");
    $pipe->send_data("Second message");
    $pipe->close_write;
    exit 0;
}

$pipe->close_write;
while (defined(my $data = $pipe->recv_data)) {
    printf "  Parent received: %s\n", $data;
}
$pipe->close_read;
waitpid($pid, 0);

# Two-way pipe
printf "\nTwo-way pipe (parent/child dialog):\n";
pipe(my $p2c_r, my $p2c_w);  # parent to child
pipe(my $c2p_r, my $c2p_w);  # child to parent

$pid = fork();
die "fork: $!" unless defined $pid;

if ($pid == 0) {
    close $p2c_w; close $c2p_r;
    while (my $line = <$p2c_r>) {
        chomp $line;
        last if $line eq "quit";
        printf $c2p_w "echo: %s\n", uc($line);
    }
    close $p2c_r; close $c2p_w;
    exit 0;
}

close $p2c_r; close $c2p_w;

my @messages = ("hello world", "perl programming", "quit");
for my $msg (@messages) {
    printf "  Parent sends: %s\n", $msg;
    print $p2c_w "$msg\n";
    last if $msg eq "quit";
    my $reply = <$c2p_r>;
    chomp($reply //= "");
    printf "  Child replies: %s\n", $reply;
}
close $p2c_w; close $c2p_r;
waitpid($pid, 0);

# Command capture
printf "\nCommand output capture:\n";
my @ls_output = split /\n/, `ls /tmp 2>/dev/null | head -5`;
printf "  /tmp files (first 5): %s\n", join(", ", @ls_output[0..2]);
```

---

## Step 463: Signal Handling

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(SIGTERM SIGINT SIGHUP SIGUSR1 SIGUSR2);

{
package SignalManager;

my %_handlers = ();
my $_blocked  = 0;
my @_deferred = ();

sub register {
    my ($class, $sig, $handler) = @_;
    $_handlers{$sig} = $handler;
    $SIG{$sig} = sub {
        if ($_blocked) {
            push @_deferred, $sig;
        } else {
            $handler->($sig);
        }
    };
    return $class;
}

sub ignore { my($class,$sig)=@_; $SIG{$sig}="IGNORE"; $class }
sub default{ my($class,$sig)=@_; $SIG{$sig}="DEFAULT"; $class }

sub block_signals {
    my ($class, $code) = @_;
    local $_blocked = 1;
    local @_deferred = ();
    $code->();
    $class->_process_deferred;
}

sub _process_deferred {
    my $class = shift;
    for my $sig (@_deferred) {
        $_handlers{$sig}->($sig) if $_handlers{$sig};
    }
}

sub send_self {
    my ($class, $sig) = @_;
    kill $sig, $$;
}

sub send_pid {
    my ($class, $pid, $sig) = @_;
    kill $sig, $pid;
}
}

package main;

printf "=== Signal Handling ===\n\n";

my @signal_log;

SignalManager->register("USR1", sub {
    push @signal_log, "Received SIGUSR1 - reloading config";
});

SignalManager->register("USR2", sub {
    push @signal_log, "Received SIGUSR2 - dumping stats";
});

SignalManager->register("HUP", sub {
    push @signal_log, "Received SIGHUP - restarting";
});

SignalManager->register("TERM", sub {
    push @signal_log, "Received SIGTERM - shutting down gracefully";
});

printf "Sending signals to self:\n";
SignalManager->send_self("USR1");
SignalManager->send_self("USR2");
SignalManager->send_self("HUP");
SignalManager->send_self("TERM");

printf "  %s\n", $_ for @signal_log;

# Graceful shutdown pattern
printf "\nGraceful shutdown simulation:\n";

my $shutdown_requested = 0;
my $reload_requested   = 0;

$SIG{TERM} = sub { $shutdown_requested = 1 };
$SIG{USR1} = sub { $reload_requested   = 1 };

# Simulate server loop
my @tasks = ("Process request 1", "Process request 2", "Process request 3");
my $iter  = 0;

for my $task (@tasks) {
    printf "  Working: %s\n", $task;
    
    # Simulate receiving TERM signal mid-work
    if ($iter == 1) {
        $SIG{TERM}->();  # Simulate signal
        printf "  [Signal received during work]\n";
    }
    
    if ($shutdown_requested) {
        printf "  Shutdown requested — finishing current task then stopping\n";
        # Complete current task but don't start new ones
        last if $iter > 1;
    }
    $iter++;
}

printf "  Server stopped cleanly\n";
```

---

## Step 464: Shared Memory & Semaphores

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Simulate shared state between processes using temp files
# (since IPC::SysV requires system calls not always available)

{
package SharedState;

sub new {
    my ($class, $path) = @_;
    require Storable;
    return bless { path => $path, data => {} }, $class;
}

sub set {
    my ($self, $key, $value) = @_;
    $self->_lock_and_modify(sub { $_[0]->{$key} = $value });
}

sub get {
    my ($self, $key) = @_;
    my $data = $self->_read;
    return $data->{$key};
}

sub incr {
    my ($self, $key, $amount) = @_;
    $amount //= 1;
    my $new_val;
    $self->_lock_and_modify(sub { $new_val = ($_[0]->{$key}//0) + $amount; $_[0]->{$key} = $new_val });
    return $new_val;
}

sub _lock_and_modify {
    my ($self, $fn) = @_;
    use Fcntl qw(:flock);
    
    open my $fh, "+>>", $self->{path} or die "Cannot open: $!";
    flock $fh, LOCK_EX;
    
    seek $fh, 0, 0;
    local $/; my $raw = <$fh>;
    my $data = ($raw && length $raw) ? eval { Storable::thaw($raw) } // {} : {};
    
    $fn->($data);
    
    truncate $fh, 0;
    seek $fh, 0, 0;
    print $fh Storable::nfreeze($data);
    
    flock $fh, LOCK_UN;
    close $fh;
}

sub _read {
    my $self = shift;
    return {} unless -f $self->{path};
    open my $fh, "<", $self->{path} or return {};
    flock $fh, LOCK_SH;
    local $/; my $raw = <$fh>;
    flock $fh, LOCK_UN;
    close $fh;
    return ($raw && length $raw) ? eval { Storable::thaw($raw) } // {} : {};
}

sub all { $_[0]->_read }
}

{
package Semaphore;
# File-based semaphore

sub new {
    my ($class, $path, $value) = @_;
    $value //= 1;
    my $self = bless { path => $path, value => $value }, $class;
    $self->_init($value);
    return $self;
}

sub _init {
    my ($self, $value) = @_;
    return if -f $self->{path};
    open my $fh, ">", $self->{path} or die "Cannot create semaphore: $!";
    print $fh $value;
    close $fh;
}

sub acquire {
    my ($self, $timeout) = @_;
    $timeout //= 10;
    my $waited = 0;
    while ($waited < $timeout) {
        if ($self->_try_acquire) { return 1 }
        select(undef,undef,undef,0.1);
        $waited += 0.1;
    }
    return 0;
}

sub _try_acquire {
    my $self = shift;
    use Fcntl qw(:flock);
    open my $fh, "+<", $self->{path} or return 0;
    flock $fh, LOCK_EX;
    local $/; my $val = <$fh> // 0;
    chomp $val;
    if ($val > 0) {
        truncate $fh, 0; seek $fh, 0, 0;
        print $fh ($val - 1);
        flock $fh, LOCK_UN; close $fh;
        return 1;
    }
    flock $fh, LOCK_UN; close $fh;
    return 0;
}

sub release {
    my $self = shift;
    use Fcntl qw(:flock);
    open my $fh, "+<", $self->{path} or return;
    flock $fh, LOCK_EX;
    local $/; my $val = <$fh> // 0;
    chomp $val;
    truncate $fh, 0; seek $fh, 0, 0;
    print $fh ($val + 1);
    flock $fh, LOCK_UN; close $fh;
}
}

package main;

printf "=== Shared State & Semaphores ===\n\n";

use File::Temp qw(tmpnam);
my $state_file = tmpnam();
my $sem_file   = tmpnam();

# Shared counter
my $shared = SharedState->new($state_file);
$shared->set("counter", 0);
$shared->set("status", "running");

printf "Initial state:\n";
printf "  counter=%s status=%s\n", $shared->get("counter"), $shared->get("status");

# Spawn workers that increment counter
printf "\nSpawning 3 workers to increment counter:\n";
my @pids;
for my $i (1..3) {
    my $pid = fork();
    die "fork: $!" unless defined $pid;
    if ($pid == 0) {
        # Child: increment 3 times each
        for (1..3) {
            $shared->incr("counter");
        }
        exit 0;
    }
    push @pids, $pid;
}

waitpid($_, 0) for @pids;
printf "  Final counter = %d (expected 9)\n\n", $shared->get("counter");

# Semaphore demonstration
printf "Semaphore (permits=2):\n";
my $sem = Semaphore->new($sem_file, 2);

for my $i (1..4) {
    if ($sem->acquire(0.5)) {
        printf "  Worker %d: acquired semaphore\n", $i;
        $sem->release;
        printf "  Worker %d: released semaphore\n", $i;
    } else {
        printf "  Worker %d: timeout acquiring semaphore\n", $i;
    }
}

unlink $state_file, $sem_file;
```

---

## Step 465-470: Process Pool, Job Delegation, Capstone

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(:sys_wait_h);

{
package ProcessPool;

sub new {
    my ($class, %opts) = @_;
    my $size = $opts{size} // 4;
    my $self = bless {
        size     => $size,
        workers  => {},
        queue    => [],
        results  => {},
        _next_id => 1,
    }, $class;
    $self->_start_workers;
    return $self;
}

sub _start_workers {
    my $self = shift;
    for my $i (1..$self->{size}) {
        pipe(my $task_r, my $task_w);
        pipe(my $res_r,  my $res_w);
        
        my $pid = fork();
        die "fork: $!" unless defined $pid;
        
        if ($pid == 0) {
            close $task_w; close $res_r;
            $self->_worker_loop($task_r, $res_w);
            exit 0;
        }
        
        close $task_r; close $res_w;
        $self->{workers}{$pid} = {
            pid     => $pid,
            task_w  => $task_w,
            res_r   => $res_r,
            busy    => 0,
            worker_id => $i,
        };
    }
}

sub _worker_loop {
    my ($self, $task_r, $res_w) = @_;
    while (1) {
        local $/;
        my $task_data = <$task_r>;
        last unless defined $task_data;
        
        my $task = eval { require Storable; Storable::thaw($task_data) };
        last unless $task;
        last if $task->{type} eq "stop";
        
        my $result;
        eval { $result = $task->{fn}->(@{$task->{args}//[]}) };
        my $err = $@;
        
        my $response = Storable::nfreeze({
            id     => $task->{id},
            result => $result,
            error  => $err || undef,
        });
        print $res_w pack("N", length $response) . $response;
    }
}

sub _send_task {
    my ($self, $worker, $task) = @_;
    require Storable;
    my $data = Storable::nfreeze($task);
    print { $worker->{task_w} } pack("N", length $data) . $data;
}

sub _read_result {
    my ($self, $res_r) = @_;
    my $hdr; read($res_r, $hdr, 4) or return undef;
    my $len = unpack("N", $hdr);
    my $data; read($res_r, $data, $len) or return undef;
    return eval { require Storable; Storable::thaw($data) };
}

sub submit {
    my ($self, $fn, @args) = @_;
    my $id = $self->{_next_id}++;
    
    # Find idle worker
    for my $pid (keys %{$self->{workers}}) {
        my $w = $self->{workers}{$pid};
        unless ($w->{busy}) {
            $w->{busy} = 1;
            $self->_send_task($w, { id => $id, type => "task", fn => $fn, args => \@args });
            my $result = $self->_read_result($w->{res_r});
            $w->{busy} = 0;
            return $result;
        }
    }
    
    # All busy — use first worker
    my ($w) = values %{$self->{workers}};
    $self->_send_task($w, { id => $id, type => "task", fn => $fn, args => \@args });
    return $self->_read_result($w->{res_r});
}

sub shutdown {
    my $self = shift;
    for my $w (values %{$self->{workers}}) {
        require Storable;
        my $stop = Storable::nfreeze({type => "stop"});
        print { $w->{task_w} } pack("N", length $stop) . $stop;
        close $w->{task_w};
    }
    while (waitpid(-1, WNOHANG) > 0) {}
}
}

package main;

printf "=== Process Pool ===\n\n";

my $pool = ProcessPool->new(size => 3);
printf "Pool started with %d workers\n\n", $pool->{size};

# Submit tasks
printf "Submitting computation tasks:\n";
my @tasks = (
    [sub { my $n=shift; $n*$n }, 7],
    [sub { my $n=shift; my $s=0; $s+=$_ for 1..$n; $s }, 100],
    [sub { my @a=@_; my $m=shift @a; $m=$_>$m?$_:$m for @a; $m }, 3,1,4,1,5,9,2,6],
    [sub { join("-", reverse @_) }, qw(a b c d e)],
);

for my $task (@tasks) {
    my ($fn, @args) = @$task;
    my $result = $pool->submit($fn, @args);
    printf "  task(%s) = %s\n",
        join(", ", map { length($_)>10 ? substr($_,0,10)."..." : $_ } @args),
        $result->{result} // ("ERROR: " . ($result->{error}//"?"));
}

$pool->shutdown;
printf "\nPool shutdown complete\n\n";

# Job delegation pattern
printf "=== Job Delegation ===\n\n";

{
package JobDelegator;

sub new {
    my ($class) = @_;
    return bless {
        handlers    => {},
        middlewares => [],
        results     => [],
    }, $class;
}

sub register { my($s,$t,$h)=@_; $s->{handlers}{$t}=$h; $s }

sub use_middleware {
    my ($self, $mw) = @_;
    push @{$self->{middlewares}}, $mw;
    return $self;
}

sub dispatch {
    my ($self, $job) = @_;
    
    # Run through middleware chain
    my @mws = @{$self->{middlewares}};
    my $execute = sub {
        my $handler = $self->{handlers}{$job->{type}};
        return { error => "No handler for $job->{type}" } unless $handler;
        return eval { $handler->($job) } // { error => $@ };
    };
    
    my $chain = $execute;
    for my $mw (reverse @mws) {
        my $next = $chain;
        $chain = sub { $mw->($job, $next) };
    }
    
    my $result = $chain->();
    push @{$self->{results}}, { job => $job, result => $result };
    return $result;
}
}

my $delegator = JobDelegator->new;

# Logging middleware
$delegator->use_middleware(sub {
    my ($job, $next) = @_;
    printf "  [LOG] Executing job: %s\n", $job->{type};
    my $t0 = time();
    my $result = $next->();
    printf "  [LOG] Completed in %dms\n", (time()-$t0)*1000;
    return $result;
});

# Error handling middleware
$delegator->use_middleware(sub {
    my ($job, $next) = @_;
    my $result = eval { $next->() };
    if ($@) { return { error => $@, job_id => $job->{id} } }
    return $result;
});

$delegator->register("compute", sub {
    my $job = shift;
    return { value => $job->{a} * $job->{b}, job_id => $job->{id} };
});

$delegator->register("transform", sub {
    my $job = shift;
    return { result => uc($job->{text}), job_id => $job->{id} };
});

for my $job (
    { id => 1, type => "compute",   a => 6, b => 7 },
    { id => 2, type => "transform", text => "hello perl" },
    { id => 3, type => "unknown" },
) {
    printf "\nDispatching job %d (type=%s):\n", $job->{id}, $job->{type};
    my $r = $delegator->dispatch($job);
    printf "  Result: %s\n", join(", ", map { "$_=$r->{$_}" } grep { defined $r->{$_} } keys %$r);
}
```

---

## สรุป Part 47 — Process Management & IPC

### สิ่งที่เรียนรู้:
- **fork & exec** — Spawning child processes, collecting results via pipes
- **Pipes** — Unidirectional and bidirectional pipe communication
- **Signal Handling** — SIGUSR1/2, SIGTERM, SIGHUP, deferred signals
- **Shared State** — File-locked shared memory simulation with Storable
- **Semaphore** — File-based counting semaphore for resource limiting
- **Process Pool** — N-worker pool with task dispatch and result collection
- **Job Delegation** — Middleware chain dispatch pattern for background work

**ถัดไป: [Part 48 — Advanced CPAN Module Development](part_48.md)**
