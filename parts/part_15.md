# Part 15: Error Handling และ Debugging
## Steps 141-150: จัดการข้อผิดพลาดอย่างมืออาชีพ

---

## Step 141: die, warn, eval

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# die — throw exception
# =====================

sub read_positive {
    my $n = shift;
    die "Must be positive, got: $n\n" if $n <= 0;
    return $n;
}

# =====================
# eval — catch exceptions
# =====================

my $result = eval { read_positive(5) };
print "Result: $result\n";

eval { read_positive(-1) };
if ($@) {
    print "Caught: $@";   # includes trailing \n
}

# $@ is cleared before eval block, set on exception
eval { 1 };  # success
print "After success: '$@'\n";   # empty string

# =====================
# Nested eval
# =====================

sub risky_operation {
    my $level = shift;
    die "Error at level $level\n" if $level > 2;
    return "OK at level $level";
}

for my $i (1..4) {
    my $r = eval { risky_operation($i) };
    if ($@) {
        printf "Level %d failed: %s", $i, $@;
    } else {
        printf "Level %d: %s\n", $i, $r;
    }
}

# =====================
# warn — non-fatal
# =====================

sub parse_int {
    my $str = shift;
    unless ($str =~ /^-?\d+$/) {
        warn "Not an integer: '$str' — returning 0\n";
        return 0;
    }
    return int $str;
}

my $n = parse_int("abc");
print "Got: $n\n";

# Catch warnings
local $SIG{__WARN__} = sub { print "CAUGHT WARN: $_[0]" };
warn "test warning\n";

# =====================
# die with object
# =====================

package MyException;

sub new {
    my ($class, %opts) = @_;
    return bless {
        message => $opts{message} // "Unknown error",
        code    => $opts{code}    // 0,
        trace   => caller_info(),
    }, $class;
}

sub caller_info {
    my @info;
    my $i = 1;
    while (my @c = caller($i++)) {
        push @info, "$c[3] at $c[1] line $c[2]";
        last if $i > 5;
    }
    return \@info;
}

sub message  { $_[0]->{message} }
sub code     { $_[0]->{code} }
sub trace    { join "\n", @{$_[0]->{trace}} }
sub throw    { die shift->new(@_) }

package main;

eval {
    MyException->throw(message => "Connection failed", code => 503);
};

if (my $e = $@) {
    if (ref $e && $e->isa('MyException')) {
        printf "Exception %d: %s\n", $e->code, $e->message;
    } else {
        print "Unknown error: $e\n";
    }
}
```

---

## Step 142: Exception Classes

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Exception hierarchy
# =====================

package Exception;

sub new {
    my ($class, %opts) = @_;
    return bless {
        message  => $opts{message}  // "An error occurred",
        code     => $opts{code}     // 0,
        context  => $opts{context}  // {},
        _stack   => _stack_trace(),
    }, $class;
}

sub _stack_trace {
    my @trace;
    my $i = 2;
    while (my @frame = caller($i++)) {
        push @trace, { 
            package => $frame[0], 
            file => $frame[1], 
            line => $frame[2],
            sub  => $frame[3]
        };
        last if $i > 10;
    }
    return \@trace;
}

sub message  { $_[0]->{message} }
sub code     { $_[0]->{code} }
sub context  { $_[0]->{context} }
sub stack    { $_[0]->{_stack} }

sub throw {
    my $class = shift;
    die $class->new(@_);
}

sub stringify {
    my $self = shift;
    return sprintf "[%s] %s (code: %d)", ref $self, $self->message, $self->code;
}

use overload '""' => \&stringify;

# =====================
# Derived exception classes
# =====================

package IOException;
our @ISA = ('Exception');

sub new {
    my ($class, %opts) = @_;
    my $self = Exception::new($class, %opts);
    $self->{filename} = $opts{filename} // "";
    $self->{errno}    = $opts{errno}    // $!;
    return $self;
}

sub filename { $_[0]->{filename} }
sub errno    { $_[0]->{errno} }

package NetworkException;
our @ISA = ('Exception');

sub new {
    my ($class, %opts) = @_;
    my $self = Exception::new($class, %opts);
    $self->{host}    = $opts{host}    // "";
    $self->{timeout} = $opts{timeout} // 0;
    return $self;
}

package ValidationException;
our @ISA = ('Exception');

sub new {
    my ($class, %opts) = @_;
    my $self = Exception::new($class, %opts);
    $self->{field}  = $opts{field}  // "";
    $self->{value}  = $opts{value}  // "";
    return $self;
}

sub field { $_[0]->{field} }
sub value { $_[0]->{value} }

# =====================
# Usage
# =====================

package main;

sub validate_age {
    my ($age) = @_;
    ValidationException->throw(
        message => "Age must be positive",
        field   => "age",
        value   => $age,
        code    => 422,
    ) if $age < 0;
    
    ValidationException->throw(
        message => "Age must be realistic",
        field   => "age",
        value   => $age,
        code    => 422,
    ) if $age > 150;
    
    return $age;
}

# Try multiple operations
my @test_ages = (25, -5, 200);

for my $age (@test_ages) {
    eval { validate_age($age) };
    if (my $e = $@) {
        if (ref $e && $e->isa('ValidationException')) {
            printf "Validation error on field '%s': %s (value=%s)\n",
                $e->field, $e->message, $e->value;
        } elsif (ref $e && $e->isa('Exception')) {
            printf "General error: %s\n", $e->message;
        } else {
            print "Unknown: $e\n";
        }
    } else {
        print "Age $age is valid\n";
    }
}
```

---

## Step 143: Carp — Better Error Messages

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Carp qw(carp croak confess cluck);

# =====================
# Carp functions
# =====================
#
#  warn   → carp    (reports at caller)
#  die    → croak   (reports at caller)
#  warn + stack → cluck
#  die + stack  → confess

package Database;

sub new {
    my ($class, %opts) = @_;
    return bless \%opts, $class;
}

sub connect {
    my ($self) = @_;
    unless ($self->{dsn}) {
        croak "DSN is required for Database::connect";  # blame caller
    }
    print "Connected to $self->{dsn}\n";
}

sub query {
    my ($self, $sql) = @_;
    unless ($sql) {
        carp "Empty SQL query — ignoring";   # warn at caller
        return;
    }
    print "Executing: $sql\n";
}

package App;

sub run {
    my $db = Database->new();
    eval { $db->connect };
    if ($@) {
        print "App::run caught: $@\n";
    }
}

package main;

# croak shows caller location (not Database::connect)
App::run();

# confess shows full stack
package BadModule;

sub deep_error {
    confess "Error with stack trace";
}

sub middle {
    deep_error();
}

package main;

eval { BadModule::middle() };
print "confess message: $@\n";

# =====================
# Carp::Always — always show stacks
# =====================

# use Carp::Always;   # uncomment to show stacks everywhere
# Now all die/warn show call stacks
```

---

## Step 144: $SIG — Signal Handlers

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(SIGTERM SIGINT);

# =====================
# Signal handlers
# =====================

# Catch Ctrl+C
$SIG{INT} = sub {
    print "\nCaught SIGINT — cleaning up...\n";
    # Cleanup here
    exit 0;
};

# Catch SIGTERM (kill PID)
$SIG{TERM} = sub {
    print "Caught SIGTERM\n";
    exit 0;
};

# Catch errors (die)
$SIG{__DIE__} = sub {
    my $err = shift;
    # Don't intercept eval{}
    return if $^S;
    print "DIE HANDLER: $err";
    exit 1;
};

# Catch warnings
my @warnings;
$SIG{__WARN__} = sub {
    my $msg = shift;
    push @warnings, $msg;
    print STDERR "WARNING: $msg";
};

# Generate some warnings
warn "Test warning 1\n";
warn "Test warning 2\n";

printf "Captured %d warnings\n", scalar @warnings;

# =====================
# ALRM — timeout
# =====================

sub with_timeout {
    my ($timeout, $code) = @_;
    
    local $SIG{ALRM} = sub { die "Timeout after ${timeout}s\n" };
    alarm $timeout;
    
    my $result = eval { $code->() };
    alarm 0;   # cancel alarm
    
    die $@ if $@ && $@ !~ /^Timeout/;
    return ($@, $result);
}

my ($err, $result) = with_timeout(2, sub {
    # Simulate fast operation
    return "done quickly";
});

print $err ? "Timed out: $err" : "Result: $result\n";

# =====================
# HUP — reload config
# =====================

my %config = (debug => 0);

$SIG{HUP} = sub {
    print "SIGHUP: reloading config\n";
    %config = (debug => 1, reloaded => 1);
};

# Send HUP to self
kill 'HUP', $$;

printf "Config after HUP: debug=%d\n", $config{debug};
```

---

## Step 145: Debugging with Data::Dumper

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Data::Dumper;

# =====================
# Dumper options
# =====================

$Data::Dumper::Sortkeys = 1;      # sort hash keys
$Data::Dumper::Terse    = 1;      # no $VAR1 =
$Data::Dumper::Indent   = 1;      # readable indentation
$Data::Dumper::Maxdepth = 3;      # limit depth

my %complex = (
    name    => "Alice",
    scores  => [95, 87, 92, 78],
    address => {
        street => "123 Main St",
        city   => "Bangkok",
        zip    => "10100",
    },
    tags    => [qw(admin user developer)],
    meta    => { created => time(), active => 1 },
);

print "Data structure:\n";
print Dumper(\%complex);

# =====================
# Debugging helper
# =====================

sub debug_dump {
    my ($label, $data) = @_;
    local $Data::Dumper::Terse = 1;
    local $Data::Dumper::Indent = 1;
    printf "[DEBUG] %s =\n%s\n", $label, Dumper($data);
}

my @arr = map { { id => $_, val => $_ * $_ } } 1..5;
debug_dump("arr", \@arr);

# =====================
# Devel::Peek — low-level
# =====================

# use Devel::Peek;
# Dump($complex{name});   # shows SV internals

# =====================
# Stack trace
# =====================

sub show_stack {
    my $i = 0;
    print "Call stack:\n";
    while (my @frame = caller($i++)) {
        printf "  %d: %s() called at %s line %d\n",
            $i, $frame[3], $frame[1], $frame[2];
        last if $i > 10;
    }
}

sub c { show_stack() }
sub b { c() }
sub a { b() }

a();

# =====================
# Variable tracing
# =====================

# Tie to trace variable access
package TraceScalar;

sub TIESCALAR {
    my ($class, $name, $val) = @_;
    return bless { name => $name, val => $val }, $class;
}

sub FETCH {
    my $self = shift;
    print "[TRACE] Reading $self->{name} = $self->{val}\n";
    return $self->{val};
}

sub STORE {
    my ($self, $val) = @_;
    print "[TRACE] Writing $self->{name}: $self->{val} -> $val\n";
    $self->{val} = $val;
}

package main;

my $traced;
tie $traced, 'TraceScalar', 'x', 10;

my $read = $traced;     # triggers FETCH
$traced = 20;           # triggers STORE
$traced += 5;           # triggers both
```

---

## Step 146: Logging ขั้นสูง

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(strftime);
use Fcntl qw(:flock);
use Scalar::Util qw(blessed);

# =====================
# Log levels
# =====================

use constant {
    LOG_TRACE => 0,
    LOG_DEBUG => 1,
    LOG_INFO  => 2,
    LOG_WARN  => 3,
    LOG_ERROR => 4,
    LOG_FATAL => 5,
};

my %LEVEL_NAMES = (
    0 => 'TRACE',
    1 => 'DEBUG',
    2 => 'INFO ',
    3 => 'WARN ',
    4 => 'ERROR',
    5 => 'FATAL',
);

# =====================
# Logger class
# =====================

package Logger;

sub new {
    my ($class, %opts) = @_;
    return bless {
        name     => $opts{name}     // 'root',
        level    => $opts{level}    // ::LOG_INFO,
        handlers => $opts{handlers} // [Logger::Handler::Console->new],
        context  => $opts{context}  // {},
    }, $class;
}

sub with_context {
    my ($self, %ctx) = @_;
    my $child = bless { %$self }, ref $self;
    $child->{context} = { %{$self->{context}}, %ctx };
    return $child;
}

sub _log {
    my ($self, $level, $msg, %extra) = @_;
    return if $level < $self->{level};
    
    my $entry = {
        ts      => strftime("%Y-%m-%d %H:%M:%S", localtime),
        level   => $level,
        name    => $self->{name},
        message => $msg,
        context => { %{$self->{context}}, %extra },
    };
    
    $_->write($entry) for @{$self->{handlers}};
}

sub trace { $_[0]->_log(::LOG_TRACE, @_[1..$#_]) }
sub debug { $_[0]->_log(::LOG_DEBUG, @_[1..$#_]) }
sub info  { $_[0]->_log(::LOG_INFO,  @_[1..$#_]) }
sub warn  { $_[0]->_log(::LOG_WARN,  @_[1..$#_]) }
sub error { $_[0]->_log(::LOG_ERROR, @_[1..$#_]) }
sub fatal { $_[0]->_log(::LOG_FATAL, @_[1..$#_]) }

# =====================
# Console handler
# =====================

package Logger::Handler::Console;

sub new { bless {}, shift }

sub write {
    my ($self, $entry) = @_;
    my $ctx = "";
    if (%{$entry->{context}}) {
        $ctx = " {" . join(", ", map { "$_=$entry->{context}{$_}" } 
                               sort keys %{$entry->{context}}) . "}";
    }
    printf "[%s] [%s] [%s] %s%s\n",
        $entry->{ts},
        $::LEVEL_NAMES{$entry->{level}},
        $entry->{name},
        $entry->{message},
        $ctx;
}

# =====================
# File handler
# =====================

package Logger::Handler::File;

sub new {
    my ($class, %opts) = @_;
    return bless {
        file  => $opts{file} // '/tmp/app.log',
        level => $opts{level} // ::LOG_INFO,
    }, $class;
}

sub write {
    my ($self, $entry) = @_;
    return if $entry->{level} < $self->{level};
    
    open(my $fh, '>>', $self->{file}) or return;
    flock $fh, Fcntl::LOCK_EX;
    seek $fh, 0, 2;
    printf $fh "[%s] [%s] %s\n",
        $entry->{ts}, $::LEVEL_NAMES{$entry->{level}}, $entry->{message};
    flock $fh, Fcntl::LOCK_UN;
    close $fh;
}

# =====================
# Test
# =====================

package main;

my $log = Logger->new(
    name     => 'myapp',
    level    => LOG_DEBUG,
    handlers => [
        Logger::Handler::Console->new,
        Logger::Handler::File->new(file => '/tmp/app.log', level => LOG_WARN),
    ],
);

$log->debug("Starting application");
$log->info("Server listening on port 8080");

my $request_log = $log->with_context(
    request_id => "REQ-001",
    user_id    => 42,
);

$request_log->info("Processing request");
$request_log->warn("Slow response: 2500ms");
$request_log->error("Database timeout");
```

---

## Step 147: Perl Debugger

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Usando o debugger
# =====================
#
# Para iniciar o debugger:
#   perl -d script.pl
#
# Comandos principais:
#   h       — help
#   n       — next line (step over)
#   s       — step into
#   c       — continue
#   c LINE  — continue to line
#   b LINE  — set breakpoint
#   b sub   — breakpoint at subroutine
#   B *     — remove all breakpoints
#   p EXPR  — print expression
#   x EXPR  — print with Data::Dumper
#   l       — list current area
#   l LINE  — list around line
#   q       — quit
#   r       — return from subroutine
#   w       — watch expression
#   V PKG   — list variables in package

# =====================
# Inline debugger (โค้ดสำหรับ debug)
# =====================

sub DEBUG { $ENV{DEBUG} || 0 }

sub dprint {
    return unless DEBUG;
    my ($level, $fmt, @args) = @_;
    return unless $level <= DEBUG;
    printf "[DEBUG $level] $fmt\n", @args;
}

# Usage:
# DEBUG=2 perl script.pl

dprint(1, "Simple debug message");
dprint(2, "Detailed: value=%d", 42);

# =====================
# breakpoint simulation
# =====================

package Debugger;

my $enabled = $ENV{TRACE} // 0;

sub trace {
    return unless $enabled;
    my ($msg) = @_;
    my @caller = caller(1);
    printf "[TRACE] %s::%s line %d: %s\n",
        $caller[0], $caller[3], $caller[2], $msg;
}

package main;

sub calculate {
    my ($a, $b, $op) = @_;
    Debugger::trace("calculate($a, $b, $op)");
    
    my $result = 
        $op eq '+' ? $a + $b :
        $op eq '-' ? $a - $b :
        $op eq '*' ? $a * $b :
        $op eq '/' ? ($b != 0 ? $a / $b : die "Division by zero") :
        die "Unknown op: $op";
    
    Debugger::trace("result = $result");
    return $result;
}

my $r = calculate(10, 3, '*');
print "10 * 3 = $r\n";

# =====================
# Assert
# =====================

sub assert {
    my ($condition, $message) = @_;
    die "Assertion failed: $message\n" unless $condition;
}

assert(2 + 2 == 4, "Basic math should work");
assert("hello" ne "world", "Different strings should not be equal");

eval { assert(1 == 2, "This will fail") };
print "Assertion failed: $@" if $@;
```

---

## Step 148: Profiling

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Time::HiRes qw(gettimeofday tv_interval);
use Scalar::Util qw(looks_like_number);

# =====================
# Simple profiler
# =====================

my %profile_data;

sub profile {
    my ($name, $code) = @_;
    
    my $t0 = [gettimeofday];
    my @result = eval { $code->() };
    my $elapsed = tv_interval($t0);
    
    die $@ if $@;
    
    $profile_data{$name}{calls}++;
    $profile_data{$name}{total} += $elapsed;
    $profile_data{$name}{min} //= $elapsed;
    $profile_data{$name}{min} = $elapsed if $elapsed < $profile_data{$name}{min};
    $profile_data{$name}{max} //= $elapsed;
    $profile_data{$name}{max} = $elapsed if $elapsed > $profile_data{$name}{max};
    
    return wantarray ? @result : $result[0];
}

sub print_profile {
    printf "\n%-25s %6s %12s %12s %12s\n",
        "Operation", "Calls", "Total(ms)", "Min(ms)", "Max(ms)";
    print "-" x 75 . "\n";
    
    for my $name (sort { 
        $profile_data{$b}{total} <=> $profile_data{$a}{total}
    } keys %profile_data) {
        my $d = $profile_data{$name};
        printf "%-25s %6d %12.3f %12.3f %12.3f\n",
            $name, $d->{calls},
            $d->{total} * 1000,
            $d->{min} * 1000,
            $d->{max} * 1000;
    }
}

# =====================
# Functions to profile
# =====================

sub bubble_sort {
    my @arr = @_;
    for my $i (0..$#arr) {
        for my $j (0..$#arr-$i-1) {
            @arr[$j, $j+1] = @arr[$j+1, $j] if $arr[$j] > $arr[$j+1];
        }
    }
    return @arr;
}

sub perl_sort { sort { $a <=> $b } @_ }

# Run benchmarks
my @data = map { int(rand 1000) } 1..1000;

for (1..5) {
    profile("bubble_sort", sub { bubble_sort(@data) });
    profile("perl_sort",   sub { perl_sort(@data) });
    profile("string_ops",  sub { my $s = join(",", @data); split /,/, $s });
}

print_profile();

# =====================
# Memory usage
# =====================

sub check_memory {
    open(my $fh, '<', '/proc/self/status') or return {};
    my %mem;
    while (my $line = <$fh>) {
        if ($line =~ /^(VmRSS|VmSize|VmPeak):\s+(\d+)\s+kB/) {
            $mem{$1} = $2;
        }
    }
    close $fh;
    return %mem;
}

my %before = check_memory();

my @large = map { "x" x 1000 } 1..10000;

my %after = check_memory();

if (%before && %after) {
    printf "\nMemory before: %dK\n", $before{VmRSS} // 0;
    printf "Memory after:  %dK\n", $after{VmRSS} // 0;
    printf "Increase:      %dK\n", ($after{VmRSS}//0) - ($before{VmRSS}//0);
}
```

---

## Step 149: Try::Tiny และ Exception Handling Patterns

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Try::Tiny (CPAN module)
# =====================

use Try::Tiny;

# Basic try/catch/finally
try {
    die "Something went wrong\n";
} catch {
    print "Caught: $_";
} finally {
    print "Always runs\n";
};

# With exception objects
package AppError;
sub new {
    my ($class, %opts) = @_;
    return bless \%opts, $class;
}
sub message { $_[0]->{message} }
sub code    { $_[0]->{code} }

package main;

try {
    die AppError->new(message => "DB Connection failed", code => 503);
} catch {
    if (ref $_ && $_->isa('AppError')) {
        printf "AppError [%d]: %s\n", $_->code, $_->message;
    } else {
        print "Unknown error: $_\n";
    }
};

# =====================
# Pattern: Result type
# =====================

package Result;

sub ok  { bless { ok => 1, value => $_[1] }, $_[0] }
sub err { bless { ok => 0, error => $_[1] }, $_[0] }

sub is_ok  { $_[0]->{ok} }
sub value  { $_[0]->{value} }
sub error  { $_[0]->{error} }

sub map_ok {
    my ($self, $fn) = @_;
    return $self unless $self->is_ok;
    return Result->ok($fn->($self->value));
}

sub unwrap {
    my $self = shift;
    die "Unwrap failed: " . ($self->error // "unknown") unless $self->is_ok;
    return $self->value;
}

package main;

sub safe_divide {
    my ($a, $b) = @_;
    return Result->err("Division by zero") if $b == 0;
    return Result->ok($a / $b);
}

sub safe_sqrt {
    my $n = shift;
    return Result->err("Cannot sqrt negative") if $n < 0;
    return Result->ok(sqrt($n));
}

# Chain operations
my @tests = ([10, 2], [10, 0], [25, 5], [-4, 2]);

for my $t (@tests) {
    my $r = safe_divide($t->[0], $t->[1])
              ->map_ok(sub { safe_sqrt(shift)->unwrap });
    
    if ($r->is_ok) {
        printf "sqrt(%d/%d) = %.4f\n", @$t, $r->value;
    } else {
        printf "Error(%d,%d): %s\n", @$t, $r->error;
    }
}

# =====================
# Pattern: Guard clause
# =====================

sub process_user {
    my (%user) = @_;
    
    return { error => "Missing name" }    unless $user{name};
    return { error => "Missing email" }   unless $user{email};
    return { error => "Invalid email" }   unless $user{email} =~ /\@/;
    return { error => "Invalid age" }     unless $user{age} && $user{age} > 0;
    return { error => "Too young" }       if $user{age} < 18;
    
    # All checks passed
    return { ok => 1, user => \%user };
}

my @users = (
    { name => "Alice", email => "alice\@example.com", age => 25 },
    { name => "Bob",   email => "bob\@example.com",   age => 15 },
    { name => "",      email => "x\@y.com",            age => 30 },
    { name => "Dave",  email => "invalid",             age => 22 },
);

for my $user (@users) {
    my $r = process_user(%$user);
    if ($r->{ok}) {
        printf "OK: %s (%s)\n", $user->{name}, $user->{email};
    } else {
        printf "FAIL %s: %s\n", $user->{name} // "?", $r->{error};
    }
}
```

---

## Step 150: โปรแกรมสรุป — Error Handling Framework

```perl
#!/usr/bin/perl
#
# error_demo.pl — ตัวอย่างการจัดการ error แบบครบวงจร
#

use strict;
use warnings;
use POSIX qw(strftime);

# =====================
# Exception hierarchy
# =====================

package X;  # base exception

sub new {
    my ($class, %opts) = @_;
    return bless {
        message => $opts{message} // "Error",
        code    => $opts{code}    // 500,
        detail  => $opts{detail}  // {},
        ts      => time(),
    }, $class;
}

sub message { $_[0]->{message} }
sub code    { $_[0]->{code} }
sub detail  { $_[0]->{detail} }
sub ts      { strftime "%Y-%m-%d %H:%M:%S", localtime $_[0]->{ts} }
sub throw   { die $_[0]->new(@_[1..$#_]) }
sub is      { ref($_[0]) eq $_[1] || $_[0]->isa($_[1]) }

use overload '""' => sub { "[" . ref(shift) . "] " . shift->message };

package X::IO;        our @ISA = ('X');
package X::Network;   our @ISA = ('X');
package X::Database;  our @ISA = ('X');
package X::Auth;      our @ISA = ('X');
package X::Validate;  our @ISA = ('X');

# =====================
# Application services
# =====================

package Service::User;

sub new { bless {}, shift }

sub create {
    my ($self, %data) = @_;
    
    # Validate
    X::Validate->throw(message => "name required", code => 422)
        unless $data{name};
    
    X::Validate->throw(message => "email required", code => 422)
        unless $data{email};
    
    X::Validate->throw(message => "invalid email format", code => 422,
                       detail => { field => "email", value => $data{email} })
        unless $data{email} =~ /^[^\@]+\@[^\@]+\.[^\@]+$/;
    
    # Simulate DB operation
    die X::Database->new(message => "Connection refused", code => 503)
        if $ENV{FAIL_DB};
    
    return { id => int(rand 1000), %data };
}

package Service::Auth;

sub new { bless { tokens => {} }, shift }

sub login {
    my ($self, $username, $password) = @_;
    
    # Simulate credential check
    unless ($username eq "admin" && $password eq "secret") {
        X::Auth->throw(message => "Invalid credentials", code => 401);
    }
    
    my $token = join "", map { ('a'..'z','0'..'9')[rand 36] } 1..32;
    $self->{tokens}{$token} = { user => $username, expires => time + 3600 };
    
    return $token;
}

sub verify {
    my ($self, $token) = @_;
    my $session = $self->{tokens}{$token}
        or X::Auth->throw(message => "Invalid token", code => 401);
    
    X::Auth->throw(message => "Token expired", code => 401)
        if $session->{expires} < time;
    
    return $session->{user};
}

# =====================
# Error handler
# =====================

package main;

my %error_responses = (
    422 => "Validation Failed",
    401 => "Unauthorized",
    403 => "Forbidden",
    404 => "Not Found",
    500 => "Internal Server Error",
    503 => "Service Unavailable",
);

sub handle_error {
    my $err = shift;
    
    if (ref $err && $err->isa('X')) {
        my $status = $err->code;
        my $label  = $error_responses{$status} // "Unknown Error";
        printf "  [%d %s] %s\n", $status, $label, $err->message;
        if (%{$err->detail}) {
            printf "    detail: %s\n", 
                join(", ", map { "$_=$err->detail->{$_}" } keys %{$err->detail});
        }
    } else {
        printf "  [500] Unexpected: $err\n";
    }
}

# =====================
# Demo
# =====================

my $auth = Service::Auth->new;
my $users = Service::User->new;

print "=== Auth Demo ===\n";

# Test login
for my $cred (["admin", "secret"], ["user", "wrong"]) {
    print "Login $cred->[0]: ";
    my $token = eval { $auth->login($cred->[0], $cred->[1]) };
    if ($@) {
        handle_error($@);
    } else {
        printf "OK (token: %.8s...)\n", $token;
        
        # Verify token
        my $user = eval { $auth->verify($token) };
        printf "  Verified as: %s\n", $user if $user;
    }
}

print "\n=== User Creation Demo ===\n";

my @new_users = (
    { name => "Alice",  email => "alice\@example.com" },
    { name => "",       email => "bob\@example.com" },
    { name => "Carol",  email => "not-an-email" },
    { name => "Dave",   email => "dave\@test.org" },
);

for my $user (@new_users) {
    printf "Create user '%s': ", $user->{name} || "(empty)";
    my $created = eval { $users->create(%$user) };
    if ($@) {
        handle_error($@);
    } else {
        printf "OK (id=%d)\n", $created->{id};
    }
}
```

---

## สรุป Part 15

ใน Part นี้คุณได้เรียนรู้:
- ✅ die, warn, eval
- ✅ Exception class hierarchy
- ✅ Carp (croak, confess, carp, cluck)
- ✅ Signal handlers ($SIG)
- ✅ Debugging ด้วย Data::Dumper
- ✅ Perl debugger (-d)
- ✅ Profiling
- ✅ Try::Tiny
- ✅ Result type pattern
- ✅ Guard clauses
- ✅ Error handling framework

**ถัดไป: [Part 16 — Advanced Data Structures](part_16.md)**
