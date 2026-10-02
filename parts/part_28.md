# Part 28: Concurrency and Forking
## Steps 271-280: การเขียนโปรแกรมแบบขนาน

---

## Step 271: fork() พื้นฐาน

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(WNOHANG);

# =====================
# fork() basics
# =====================

printf "Parent PID: %d\n", $$;

my $pid = fork();
defined $pid or die "fork failed: $!";

if ($pid == 0) {
    # Child process
    printf "Child  PID: %d  Parent: %d\n", $$, getppid();
    sleep 1;
    printf "Child done\n";
    exit 0;
} else {
    # Parent process
    printf "Parent forked child: %d\n", $pid;
    my $waited = waitpid($pid, 0);
    printf "Child %d exited with: %d\n", $waited, $? >> 8;
}

# =====================
# Multiple children
# =====================

printf "\n--- Multiple children ---\n";

my @pids;
for my $i (1..3) {
    my $pid = fork() // die "fork: $!";
    if ($pid == 0) {
        # Child
        my $delay = 0.1 * $i;
        select undef, undef, undef, $delay;
        printf "  Child %d finished (delay=%.1fs)\n", $i, $delay;
        exit $i;  # exit code = child number
    } else {
        push @pids, $pid;
    }
}

# Wait for all children
for my $child_pid (@pids) {
    my $ret = waitpid($child_pid, 0);
    printf "Child %d exited: %d\n", $ret, $? >> 8;
}

# =====================
# Non-blocking wait
# =====================

printf "\n--- Non-blocking wait ---\n";

my @bg_pids;
for my $i (1..3) {
    my $pid = fork() // die "fork: $!";
    if ($pid == 0) {
        select undef, undef, undef, 0.2 * $i;
        exit 0;
    }
    push @bg_pids, $pid;
}

# Poll until all done
while (@bg_pids) {
    for my $i (reverse 0..$#bg_pids) {
        my $ret = waitpid($bg_pids[$i], WNOHANG);
        if ($ret > 0) {
            printf "  Reaped %d\n", $ret;
            splice @bg_pids, $i, 1;
        }
    }
    select undef, undef, undef, 0.1 unless !@bg_pids;
}

printf "All children done\n";
```

---

## Step 272: Pipes and IPC

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Pipe between parent and child
# =====================

# Create pipe: parent reads, child writes
pipe(my $reader, my $writer) or die "pipe: $!";

my $pid = fork() // die "fork: $!";

if ($pid == 0) {
    # Child: write to pipe
    close $reader;
    
    for my $i (1..5) {
        printf $writer "Message %d from child\n", $i;
        sleep 0 + 0.1;
    }
    print $writer "DONE\n";
    close $writer;
    exit 0;
} else {
    # Parent: read from pipe
    close $writer;
    
    while (my $line = <$reader>) {
        chomp $line;
        last if $line eq "DONE";
        printf "Parent got: %s\n", $line;
    }
    close $reader;
    waitpid($pid, 0);
}

# =====================
# Bidirectional pipe
# =====================

printf "\n--- Bidirectional ---\n";

pipe(my $p2c_r, my $p2c_w) or die "pipe p2c: $!";
pipe(my $c2p_r, my $c2p_w) or die "pipe c2p: $!";

my $pid2 = fork() // die "fork: $!";

if ($pid2 == 0) {
    # Child: echo server
    close $p2c_w; close $c2p_r;
    
    while (my $line = <$p2c_r>) {
        chomp $line;
        printf $c2p_w "ECHO: %s\n", uc($line);
    }
    close $p2c_r; close $c2p_w;
    exit 0;
} else {
    # Parent: send and receive
    close $p2c_r; close $c2p_w;
    
    for my $msg (qw(hello world perl)) {
        printf $p2c_w "%s\n", $msg;
        my $reply = <$c2p_r>;
        chomp $reply;
        printf "Sent: %-10s Got: %s\n", $msg, $reply;
    }
    close $p2c_w;
    while (my $r = <$c2p_r>) { chomp $r; printf "Got: %s\n", $r; }
    close $c2p_r;
    waitpid($pid2, 0);
}

# =====================
# open with pipe
# =====================

printf "\n--- Pipe to command ---\n";

# Write to command
open my $cmd, "|-", "sort", "-r" or die "pipe: $!";
print $cmd "$_\n" for qw(banana apple cherry date elderberry);
close $cmd;

# Read from command
open my $out, "-|", "ls", "/tmp" or die "pipe: $!";
my @files = <$out>;
close $out;
printf "Files in /tmp: %d\n", scalar @files;
```

---

## Step 273: Shared Memory

```perl
#!/usr/bin/perl
use strict;
use warnings;
use IPC::Open2;
use Storable qw(freeze thaw);
use File::Temp qw(tempfile);

# =====================
# Shared data via temp file
# =====================

sub shared_counter {
    my ($file) = @_;
    return sub {
        open my $fh, '+<', $file or do {
            open my $fh2, '>', $file; print $fh2 "0\n"; close $fh2;
            open $fh, '+<', $file;
        };
        use Fcntl qw(LOCK_EX);
        flock($fh, LOCK_EX);
        seek($fh, 0, 0);
        my $val = <$fh>; chomp $val; $val //= 0;
        $val++;
        seek($fh, 0, 0);
        print $fh "$val\n";
        truncate $fh, tell $fh;
        close $fh;
        return $val;
    };
}

my ($tmp_fh, $tmp_path) = tempfile(UNLINK => 1);
close $tmp_fh;
open $tmp_fh, '>', $tmp_path; print $tmp_fh "0\n"; close $tmp_fh;

my $counter = shared_counter($tmp_path);

printf "Shared counter demo:\n";
my @child_pids;

for my $i (1..5) {
    my $pid = fork() // die;
    if ($pid == 0) {
        my $val = $counter->();
        printf "  Child %d: counter = %d\n", $i, $val;
        exit 0;
    }
    push @child_pids, $pid;
}

waitpid($_, 0) for @child_pids;

open my $r, '<', $tmp_path;
my $final = <$r>; chomp $final;
close $r;
printf "Final counter: %d\n", $final;

# =====================
# IPC::Open2 — bidirectional process communication
# =====================

printf "\n--- IPC::Open2 ---\n";

my ($child_out, $child_in);
my $bc_pid = open2($child_out, $child_in, "bc", "-q");

# Use bc as a calculator
for my $expr ("2^10", "sqrt(144)", "3.14159 * 2^2") {
    printf $child_in "%s\n", $expr;
    my $result = <$child_out>;
    chomp $result;
    printf "%-20s = %s\n", $expr, $result;
}

close $child_in;
waitpid($bc_pid, 0);
```

---

## Step 274: Signal Handling

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(:signal_h WNOHANG);

# =====================
# Signal handlers
# =====================

my $running = 1;
my $requests = 0;

# SIGINT (Ctrl-C)
$SIG{INT} = sub {
    printf "\nCaught SIGINT — shutting down gracefully\n";
    $running = 0;
};

# SIGTERM (kill)
$SIG{TERM} = sub {
    printf "Caught SIGTERM — terminating\n";
    $running = 0;
};

# SIGHUP (reload)
$SIG{HUP} = sub {
    printf "Caught SIGHUP — reloading config\n";
    # In real app: reload config file
};

# SIGCHLD (child exit) — reap zombies
$SIG{CHLD} = sub {
    while ((my $pid = waitpid(-1, WNOHANG)) > 0) {
        printf "  Reaped child %d\n", $pid;
    }
};

# =====================
# Self-signal demo
# =====================

printf "Signal demo (PID $$)\n";

# Fork a child that signals parent
my $child_pid = fork() // die;
if ($child_pid == 0) {
    sleep 1;
    kill 'HUP', getppid();   # Tell parent to reload
    sleep 1;
    exit 0;
}

# Parent waits (briefly)
for (1..20) {
    last unless $running;
    select undef, undef, undef, 0.1;
    $requests++;
}

printf "Handled %d iterations\n", $requests;
waitpid($child_pid, 0);

# =====================
# Safe signal handler
# =====================

printf "\n--- Safe signal demo ---\n";

my $shutdown_requested = 0;

$SIG{INT} = sub {
    # Only set flag; don't do complex work in signal handler
    $shutdown_requested = 1;
};

# Simulate work loop
for my $i (1..5) {
    if ($shutdown_requested) {
        printf "Shutdown requested, stopping at iteration %d\n", $i;
        last;
    }
    printf "Working... step %d\n", $i;
    select undef, undef, undef, 0.01;
}

# Restore default
$SIG{INT} = 'DEFAULT';
printf "Signal handling demo done\n";

# =====================
# Alarm timeout
# =====================

printf "\n--- Alarm ---\n";

sub with_timeout {
    my ($seconds, $code) = @_;
    
    eval {
        local $SIG{ALRM} = sub { die "timeout\n" };
        alarm $seconds;
        $code->();
        alarm 0;
    };
    
    alarm 0;  # ensure cleared
    
    if ($@ eq "timeout\n") {
        return (undef, "timed out after ${seconds}s");
    } elsif ($@) {
        return (undef, $@);
    }
    
    return (1, undef);
}

my ($ok, $err) = with_timeout(2, sub {
    select undef, undef, undef, 0.5;  # fast operation
    printf "  Fast operation done\n";
});
printf "Fast op: %s\n", $ok ? "succeeded" : "failed ($err)";

my ($ok2, $err2) = with_timeout(1, sub {
    sleep 3;  # too slow — will timeout
});
printf "Slow op: %s\n", $ok2 ? "succeeded" : "timed out ($err2)";
```

---

## Step 275: Thread::Queue (Worker Pool)

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Worker pool using fork (no threads needed)
# =====================

{
package WorkerPool;

use POSIX qw(WNOHANG);

sub new {
    my ($class, %opts) = @_;
    return bless {
        workers  => $opts{workers}  // 4,
        jobs     => [],
        pids     => [],
        results  => {},
        finished => 0,
    }, $class;
}

sub submit {
    my ($self, $job) = @_;
    push @{$self->{jobs}}, $job;
}

sub run {
    my ($self, $worker_fn) = @_;
    
    my @jobs = @{$self->{jobs}};
    my $w    = $self->{workers};
    my @results;
    
    # Process in batches
    while (@jobs) {
        my @batch;
        push @batch, shift @jobs while @jobs && @batch < $w;
        
        my (@batch_pids, %pid_to_job);
        my ($tmp_fh, $tmp_path);
        
        require File::Temp;
        ($tmp_fh, $tmp_path) = File::Temp::tempfile(UNLINK => 1);
        close $tmp_fh;
        
        for my $i (0..$#batch) {
            my $pid = fork() // die "fork: $!";
            if ($pid == 0) {
                my $result = $worker_fn->($batch[$i]);
                require Storable;
                open my $f, '>>', $tmp_path;
                use Fcntl qw(LOCK_EX);
                flock $f, LOCK_EX;
                print $f Storable::freeze([$i, $result]) . "\n---END---\n";
                close $f;
                exit 0;
            }
            push @batch_pids, $pid;
            $pid_to_job{$pid} = $i;
        }
        
        waitpid($_, 0) for @batch_pids;
        push @results, map { $batch[$_] } 0..$#batch;
    }
    
    return @results;
}
}

# =====================
# Parallel processing with fork
# =====================

sub parallel_map {
    my ($workers, $items, $fn) = @_;
    
    my @results = (undef) x @$items;
    my @pending = map { [$_, $items->[$_]] } 0..$#$items;
    my %pid_to_idx;
    my $active = 0;
    
    my ($r_fh, $w_fh);
    pipe($r_fh, $w_fh) or die "pipe: $!";
    
    while (@pending || $active > 0) {
        # Start workers
        while (@pending && $active < $workers) {
            my ($idx, $item) = @{shift @pending};
            my $pid = fork() // die;
            if ($pid == 0) {
                close $r_fh;
                my $result = $fn->($item);
                require Storable;
                my $data = Storable::freeze([$idx, $result]);
                my $len  = length $data;
                syswrite $w_fh, pack("N", $len) . $data;
                close $w_fh;
                exit 0;
            }
            $pid_to_idx{$pid} = $idx;
            $active++;
        }
        
        # Read result
        my $len_buf;
        if (sysread $r_fh, $len_buf, 4) {
            my $len = unpack("N", $len_buf);
            my $data;
            sysread $r_fh, $data, $len;
            require Storable;
            my ($idx, $result) = @{Storable::thaw($data)};
            $results[$idx] = $result;
        }
        
        # Reap children
        while ((my $pid = waitpid(-1, POSIX::WNOHANG())) > 0) {
            delete $pid_to_idx{$pid};
            $active--;
        }
    }
    
    close $r_fh;
    close $w_fh;
    return @results;
}

# Demo: parallel calculation
printf "Worker pool demo:\n";

my @numbers = (1..10);
my @doubled = parallel_map(4, \@numbers, sub { $_[0] * 2 });

printf "Input:  %s\n", join(", ", @numbers);
printf "Output: %s\n", join(", ", @doubled);

# Sequential vs parallel timing
my @heavy = (1..8);
my $t0 = time();
my @seq_results = map { $_**2 + sqrt($_) } @heavy;
printf "\nSequential: %.2fs\n", time()-$t0;

my $t1 = time();
my @par_results = parallel_map(4, \@heavy, sub { $_[0]**2 + sqrt($_[0]) });
printf "Parallel (fork): %.2fs\n", time()-$t1;
printf "Results match: %s\n", "@seq_results" eq "@par_results" ? "yes" : "no";
```

---

## Step 276: AnyEvent / Async (Simulated)

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Event-driven programming (simulated without AnyEvent)
# =====================

{
package EventLoop;

sub new {
    return bless {
        timers    => [],
        io_ready  => [],
        callbacks => {},
        running   => 0,
        tick      => 0,
    }, shift;
}

sub add_timer {
    my ($self, $delay, $cb) = @_;
    my $fire_at = time() + $delay;
    push @{$self->{timers}}, { at => $fire_at, cb => $cb };
    @{$self->{timers}} = sort { $a->{at} <=> $b->{at} } @{$self->{timers}};
}

sub add_interval {
    my ($self, $interval, $cb) = @_;
    my $add;
    $add = sub {
        $self->add_timer($interval, sub {
            $cb->();
            $add->();
        });
    };
    $add->();
}

sub on {
    my ($self, $event, $cb) = @_;
    push @{$self->{callbacks}{$event}}, $cb;
}

sub emit {
    my ($self, $event, @args) = @_;
    $_->(@args) for @{$self->{callbacks}{$event} // []};
}

sub run_until {
    my ($self, $seconds) = @_;
    my $stop_at = time() + $seconds;
    $self->{running} = 1;
    
    while ($self->{running} && time() < $stop_at) {
        $self->{tick}++;
        
        # Process due timers
        my $now = time();
        while (@{$self->{timers}} && $self->{timers}[0]{at} <= $now) {
            my $timer = shift @{$self->{timers}};
            $timer->{cb}->();
        }
        
        select undef, undef, undef, 0.01;  # 10ms tick
    }
    
    $self->{running} = 0;
}

sub stop { $_[0]->{running} = 0 }
}

# =====================
# Demo: event-driven server simulation
# =====================

my $loop = EventLoop->new;

my $request_count = 0;
my $error_count   = 0;

# Simulate incoming requests every 100ms
$loop->add_interval(0.1, sub {
    $request_count++;
    $loop->emit("request", { id => $request_count, path => "/api/data" });
});

# Simulate errors every 350ms
$loop->add_interval(0.35, sub {
    $error_count++;
    $loop->emit("error", "Simulated error #$error_count");
});

# Handle events
$loop->on("request", sub {
    my $req = shift;
    # Simulate async processing
    $loop->add_timer(0.05, sub {
        # "Response" after 50ms
    });
});

$loop->on("error", sub {
    my $msg = shift;
    printf "  [ERROR] %s\n", $msg;
});

# Stats every 500ms
$loop->add_interval(0.5, sub {
    printf "  [STATS] requests=%d errors=%d tick=%d\n",
        $request_count, $error_count, $loop->{tick};
});

# Stop after 2 seconds
$loop->add_timer(2.0, sub {
    printf "  [STOP] Stopping event loop\n";
    $loop->stop;
});

printf "Starting event loop (runs for 2 seconds)...\n";
$loop->run_until(3);

printf "\nFinal: requests=%d errors=%d ticks=%d\n",
    $request_count, $error_count, $loop->{tick};
```

---

## Step 277: Producer-Consumer

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(WNOHANG);

# =====================
# Producer-Consumer via pipe
# =====================

my ($r, $w);
pipe($r, $w) or die "pipe: $!";

my $WORKERS = 3;
my @items   = map { "task_$_" } 1..12;

# Producer
my $producer_pid = fork() // die;
if ($producer_pid == 0) {
    close $r;
    
    for my $item (@items) {
        printf $w "%s\n", $item;
        select undef, undef, undef, 0.05;  # simulate work
    }
    
    # Send poison pill for each worker
    print $w "STOP\n" for 1..$WORKERS;
    close $w;
    exit 0;
}

close $w;

# Results pipe
my ($rr, $rw);
pipe($rr, $rw) or die "pipe: $!";

# Consumers
my @consumer_pids;
for my $i (1..$WORKERS) {
    my $pid = fork() // die;
    if ($pid == 0) {
        close $rr;
        
        while (my $task = <$r>) {
            chomp $task;
            last if $task eq "STOP";
            
            # Process task
            my $result = uc($task) . "_processed";
            select undef, undef, undef, 0.02;  # simulate work
            
            printf $rw "%s\n", $result;
        }
        
        close $r;
        close $rw;
        exit 0;
    }
    push @consumer_pids, $pid;
}

close $r;
close $rw;

# Collect results
my @results;
my $done = 0;

# Read until all consumers done
while (1) {
    my $line = <$rr>;
    last unless defined $line;
    chomp $line;
    push @results, $line;
}
close $rr;

waitpid($producer_pid, 0);
waitpid($_, 0) for @consumer_pids;

printf "Producer-Consumer results (%d/%d):\n", scalar @results, scalar @items;
printf "  %s\n", $_ for sort @results;
```

---

## Step 278: Rate Limiting and Throttle

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Rate limiter (token bucket)
# =====================

{
package RateLimiter;

sub new {
    my ($class, %opts) = @_;
    return bless {
        capacity    => $opts{capacity}    // 10,
        refill_rate => $opts{refill_rate} // 1,   # tokens per second
        tokens      => $opts{capacity}    // 10,
        last_refill => time(),
        denied      => 0,
        allowed     => 0,
    }, $class;
}

sub _refill {
    my $self = shift;
    my $now  = time();
    my $elapsed = $now - $self->{last_refill};
    
    if ($elapsed > 0) {
        my $new_tokens = $elapsed * $self->{refill_rate};
        $self->{tokens} = List::Util::min($self->{capacity}, $self->{tokens} + $new_tokens);
        $self->{last_refill} = $now;
    }
}

sub allow {
    my ($self, $cost) = @_;
    $cost //= 1;
    $self->_refill;
    
    if ($self->{tokens} >= $cost) {
        $self->{tokens} -= $cost;
        $self->{allowed}++;
        return 1;
    }
    
    $self->{denied}++;
    return 0;
}

sub tokens { int($_[0]->{tokens}) }
sub stats  { ($allowed => $_[0]->{allowed}, denied => $_[0]->{denied}) }
}

use List::Util qw(min);
use POSIX qw(floor);

my $limiter = RateLimiter->new(capacity => 5, refill_rate => 2);

printf "Rate Limiter demo (capacity=5, refill=2/s):\n";
printf "Initial tokens: %d\n\n", $limiter->tokens;

# Burst of requests
for my $i (1..8) {
    my $allowed = $limiter->allow;
    printf "Request %2d: %s (tokens: %d)\n",
        $i, $allowed ? "ALLOWED" : "DENIED", $limiter->tokens;
}

# Wait and retry
select undef, undef, undef, 1.5;  # wait 1.5s to refill 3 tokens
printf "\nAfter 1.5s wait:\n";

for my $i (1..5) {
    my $allowed = $limiter->allow;
    printf "Request %2d: %s (tokens: %d)\n",
        $i, $allowed ? "ALLOWED" : "DENIED", $limiter->tokens;
}

my %stats = $limiter->stats;
printf "\nStats: allowed=%d denied=%d\n", $stats{allowed}, $stats{denied};

# =====================
# Throttle / Debounce
# =====================

{
package Throttle;

sub new {
    my ($class, $min_interval) = @_;
    return bless { interval => $min_interval, last_call => 0, count => 0 }, $class;
}

sub call {
    my ($self, $fn) = @_;
    my $now = time();
    
    if ($now - $self->{last_call} >= $self->{interval}) {
        $self->{last_call} = $now;
        $self->{count}++;
        return $fn->();
    }
    return undef;
}

sub call_count { $_[0]->{count} }
}

my $throttle = Throttle->new(0.5);  # max once per 500ms

printf "\nThrottle demo (max 1 call per 0.5s):\n";
my $actual_calls = 0;

for my $i (1..10) {
    my $result = $throttle->call(sub {
        $actual_calls++;
        return "response_$i";
    });
    
    printf "Attempt %2d: %s\n", $i, $result // "(throttled)";
    select undef, undef, undef, 0.1;  # 100ms between attempts
}

printf "Throttled: %d/%d calls executed\n", $throttle->call_count, 10;
```

---

## Step 279: Process Management

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(:sys_wait_h);

# =====================
# Process pool manager
# =====================

{
package ProcessPool;

sub new {
    my ($class, %opts) = @_;
    return bless {
        max_workers => $opts{max_workers} // 4,
        workers     => {},   # pid => info
        tasks       => [],
        completed   => 0,
        failed      => 0,
    }, $class;
}

sub submit {
    my ($self, $task) = @_;
    push @{$self->{tasks}}, $task;
}

sub _start_worker {
    my ($self, $task) = @_;
    
    my $pid = fork() // die "fork: $!";
    if ($pid == 0) {
        eval { $task->{fn}->($task->{args}) };
        exit $@ ? 1 : 0;
    }
    
    $self->{workers}{$pid} = {
        started => time(),
        task    => $task,
    };
}

sub run {
    my $self = shift;
    
    while (@{$self->{tasks}} || %{$self->{workers}}) {
        # Start new workers if slots available
        while (@{$self->{tasks}} && scalar(keys %{$self->{workers}}) < $self->{max_workers}) {
            $self->_start_worker(shift @{$self->{tasks}});
        }
        
        # Reap finished workers
        while ((my $pid = waitpid(-1, WNOHANG)) > 0) {
            my $info   = delete $self->{workers}{$pid};
            my $status = $? >> 8;
            
            if ($status == 0) {
                $self->{completed}++;
            } else {
                $self->{failed}++;
                printf "Worker %d failed (status=%d)\n", $pid, $status;
            }
        }
        
        select undef, undef, undef, 0.01 if %{$self->{workers}};
    }
}

sub stats {
    return (
        completed => $_[0]->{completed},
        failed    => $_[0]->{failed},
    );
}
}

# Demo
my $pool = ProcessPool->new(max_workers => 3);

for my $i (1..9) {
    $pool->submit({
        fn   => sub { select undef,undef,undef, 0.1 },
        args => $i,
    });
}

printf "Processing 9 tasks with 3 workers...\n";
my $t0 = time();
$pool->run;
my $elapsed = time() - $t0;

my %stats = $pool->stats;
printf "Done: %d completed, %d failed in ~%ds\n",
    $stats{completed}, $stats{failed}, $elapsed;

# =====================
# Daemon process
# =====================

sub daemonize {
    # Fork and let parent exit
    my $pid = fork() // die "fork: $!";
    exit 0 if $pid;  # parent exits
    
    # New session
    POSIX::setsid() or die "setsid: $!";
    
    # Fork again (prevent zombie)
    $pid = fork() // die "fork: $!";
    exit 0 if $pid;
    
    # Redirect stdio
    open STDIN,  '<', '/dev/null';
    open STDOUT, '>', '/dev/null';
    open STDERR, '>', '/dev/null';
    
    chdir('/');
    umask(0);
}

printf "\nDaemon demo: would daemonize here (skipped in demo)\n";
printf "Process ID: %d\n", $$;
```

---

## Step 280: โปรแกรมสรุป — Parallel Web Scraper

```perl
#!/usr/bin/perl
# parallel_scraper.pl — parallel URL fetcher using fork
use strict;
use warnings;
use POSIX qw(WNOHANG);

# =====================
# Parallel URL fetcher
# =====================

{
package ParallelFetcher;

sub new {
    my ($class, %opts) = @_;
    return bless {
        workers  => $opts{workers} // 4,
        timeout  => $opts{timeout} // 10,
        results  => {},
    }, $class;
}

sub fetch_all {
    my ($self, @urls) = @_;
    
    my @pending = @urls;
    my %pid_to_url;
    my %results;
    
    # Pipes for result collection
    my ($r, $w);
    pipe($r, $w) or die "pipe: $!";
    
    while (@pending || %pid_to_url) {
        # Spawn workers
        while (@pending && scalar(keys %pid_to_url) < $self->{workers}) {
            my $url = shift @pending;
            my $pid = fork() // die "fork: $!";
            
            if ($pid == 0) {
                close $r;
                my $result = $self->_fetch_url($url);
                
                require Storable;
                my $data = Storable::freeze({ url => $url, %$result });
                my $len  = length $data;
                
                # Write length-prefixed result
                syswrite $w, pack("N", $len);
                syswrite $w, $data;
                
                close $w;
                exit 0;
            }
            
            $pid_to_url{$pid} = $url;
        }
        
        # Read available results (non-blocking)
        my $bits = "";
        vec($bits, fileno($r), 1) = 1;
        
        if (select($bits, undef, undef, 0.1) > 0) {
            my $len_buf;
            if (sysread $r, $len_buf, 4) {
                my $len = unpack("N", $len_buf);
                my $data;
                sysread $r, $data, $len;
                
                require Storable;
                my $result = Storable::thaw($data);
                $results{$result->{url}} = $result;
            }
        }
        
        # Reap workers
        while ((my $pid = waitpid(-1, WNOHANG)) > 0) {
            delete $pid_to_url{$pid};
        }
    }
    
    close $r;
    close $w;
    
    return %results;
}

sub _fetch_url {
    my ($self, $url) = @_;
    
    # Simulate fetch (in real usage: use LWP)
    select undef, undef, undef, 0.1 + rand(0.2);  # simulate network delay
    
    my $status  = (rand() > 0.1) ? 200 : 404;  # 90% success
    my $size    = int(rand(50000) + 1000);
    my $latency = int(rand(200) + 50);
    
    return {
        status  => $status,
        size    => $size,
        latency => $latency,
        ok      => $status == 200,
    };
}
}

# =====================
# Demo
# =====================

package main;

my @urls = map { "https://example$_.com/page" } 1..12;

printf "Fetching %d URLs with 4 parallel workers...\n\n", scalar @urls;

my $fetcher = ParallelFetcher->new(workers => 4);
my $t0      = time();
my %results = $fetcher->fetch_all(@urls);
my $elapsed = time() - $t0;

# Report
my $success = grep { $_->{ok} } values %results;
my $failed  = grep { !$_->{ok} } values %results;
my @latencies = map { $_->{latency} } values %results;

use List::Util qw(sum min max);
my $avg_lat = @latencies ? sum(@latencies) / @latencies : 0;

printf "Results:\n";
for my $url (sort keys %results) {
    my $r = $results{$url};
    printf "  %-40s %s (%dms, %d bytes)\n",
        $url, $r->{ok} ? "OK" : "FAIL", $r->{latency}, $r->{size};
}

printf "\nSummary:\n";
printf "  Total:    %d\n", scalar @urls;
printf "  Success:  %d\n", $success;
printf "  Failed:   %d\n", $failed;
printf "  Avg lat:  %dms\n", $avg_lat;
printf "  Min lat:  %dms\n", min(@latencies);
printf "  Max lat:  %dms\n", max(@latencies);
printf "  Elapsed:  %ds\n",  $elapsed;
printf "  Speedup:  ~%.1fx vs sequential\n",
    (@latencies ? sum(@latencies)/1000 / ($elapsed||1) : 1);

print "\nParallel scraper complete!\n";
```

---

## สรุป Part 28

ใน Part นี้คุณได้เรียนรู้:
- ✅ fork() พื้นฐาน: parent/child, multiple children
- ✅ Pipes และ IPC: bidirectional communication
- ✅ Shared memory via files with locking
- ✅ IPC::Open2 สำหรับสื่อสารกับ process ภายนอก
- ✅ Signal handling: SIGINT, SIGTERM, SIGHUP, SIGCHLD
- ✅ Alarm timeout
- ✅ Worker pool pattern
- ✅ Producer-Consumer pattern
- ✅ Rate limiting (Token Bucket)
- ✅ Process pool manager
- ✅ Parallel web scraper

**ถัดไป: [Part 29 — CPAN Module Development](part_29.md)**
