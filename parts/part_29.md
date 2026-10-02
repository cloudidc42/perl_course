# Part 29: CPAN Module Development
## Steps 281-290: การสร้าง Perl Modules

---

## Step 281: Module Structure

```perl
#!/usr/bin/perl
# =====================
# Standard Perl module structure
# =====================

# lib/My/Module.pm
package My::Module;

use strict;
use warnings;
our $VERSION = '1.00';

use Exporter 'import';
our @EXPORT      = ();           # nothing exported by default
our @EXPORT_OK   = qw(func1 func2);
our %EXPORT_TAGS = (
    all     => \@EXPORT_OK,
    basic   => [qw(func1)],
);

# Constructor
sub new {
    my ($class, %args) = @_;
    return bless {
        name    => $args{name} // "default",
        debug   => $args{debug} // 0,
        _data   => {},
    }, $class;
}

# Accessors
sub name  { $_[0]->{name} }
sub debug { $_[0]->{debug} }

# Methods
sub func1 { "func1: $_[1]" }
sub func2 { "func2: $_[1]" }

sub set { $_[0]->{_data}{$_[1]} = $_[2]; $_[0] }
sub get { $_[0]->{_data}{$_[1]} }

# Cleanup
sub DESTROY {
    my $self = shift;
    # cleanup resources
}

1;    # Required: module must return true

__END__

=head1 NAME

My::Module - A sample module

=head1 SYNOPSIS

    use My::Module;
    my $obj = My::Module->new(name => "test");
    $obj->set("key", "value");
    print $obj->get("key");

=head1 DESCRIPTION

This module demonstrates standard Perl module structure.

=head1 METHODS

=head2 new(%args)

Constructor. Accepts: name, debug.

=head2 set($key, $value)

Store a key-value pair.

=head2 get($key)

Retrieve a value by key.

=head1 AUTHOR

Your Name

=head1 LICENSE

This library is free software; you can redistribute it and/or modify
it under the same terms as Perl itself.

=cut
```

```perl
#!/usr/bin/perl
# test_my_module.pl
use strict;
use warnings;

# Inline test of module structure
eval <<'END_MODULE';
package My::Module;
use strict;
use warnings;
our $VERSION = '1.00';

sub new {
    my ($class, %args) = @_;
    return bless { name => $args{name}//"default", _data => {} }, $class;
}
sub name { $_[0]->{name} }
sub set  { $_[0]->{_data}{$_[1]} = $_[2]; $_[0] }
sub get  { $_[0]->{_data}{$_[1]} }
1;
END_MODULE

my $m = My::Module->new(name => "test");
printf "Name: %s\n", $m->name;
$m->set("key", "value");
printf "Get:  %s\n", $m->get("key");
printf "Version: %s\n", $My::Module::VERSION;
```

---

## Step 282: Exporter

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Module with Exporter
# =====================

{
package Math::Extra;
use strict;
use warnings;

use Exporter 'import';
our @EXPORT      = qw(add);                   # auto-exported
our @EXPORT_OK   = qw(subtract multiply divide factorial fibonacci);
our %EXPORT_TAGS = (
    arithmetic => [qw(add subtract multiply divide)],
    series     => [qw(factorial fibonacci)],
    all        => [@EXPORT_OK],
);

sub add      { my $s=0; $s+=$_ for @_; $s }
sub subtract { my $r = shift; $r -= $_ for @_; $r }
sub multiply { my $p=1; $p*=$_ for @_; $p }
sub divide   { die "div by zero\n" unless $_[1]; $_[0]/$_[1] }

sub factorial {
    my $n = shift;
    return 1 if $n <= 1;
    return $n * factorial($n-1);
}

sub fibonacci {
    my $n = shift;
    return $n if $n <= 1;
    my ($a,$b) = (0,1);
    ($a,$b) = ($b, $a+$b) for 2..$n;
    return $b;
}

sub pi    { 3.14159265358979 }
sub euler { 2.71828182845905 }
}

package main;

# Default import (add only)
Math::Extra->import();
printf "add(1..5) = %d\n", add(1..5);

# Named imports
Math::Extra->import(qw(multiply factorial));
printf "multiply(2,3,4) = %d\n", multiply(2,3,4);
printf "factorial(10) = %d\n", factorial(10);

# Tag import
Math::Extra->import(':series');
printf "fibonacci(10) = %d\n", fibonacci(10);

# Full qualified (no import needed)
printf "pi = %.5f\n", Math::Extra::pi();
printf "e  = %.5f\n", Math::Extra::euler();

# Test all arithmetic
Math::Extra->import(':arithmetic');
printf "\nArithmetic:\n";
printf "5 + 3 = %d\n",  add(5,3);
printf "10 - 3 = %d\n", subtract(10,3);
printf "4 * 5 = %d\n",  multiply(4,5);
printf "15 / 3 = %d\n", divide(15,3);
```

---

## Step 283: Object-Oriented Module

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Professional OO module
# =====================

{
package Cache::Simple;

use strict;
use warnings;
our $VERSION = '1.00';

use constant {
    DEFAULT_TTL  => 300,
    DEFAULT_SIZE => 1000,
};

sub new {
    my ($class, %args) = @_;
    return bless {
        max_size => $args{max_size} // DEFAULT_SIZE,
        ttl      => $args{ttl}      // DEFAULT_TTL,
        _store   => {},
        _order   => [],      # LRU order
        _stats   => { hits => 0, misses => 0, evictions => 0 },
    }, $class;
}

sub set {
    my ($self, $key, $value, $ttl) = @_;
    $ttl //= $self->{ttl};
    
    # Remove if already exists (for LRU reorder)
    if (exists $self->{_store}{$key}) {
        $self->{_order} = [grep { $_ ne $key } @{$self->{_order}}];
    }
    
    # Evict if full
    if (@{$self->{_order}} >= $self->{max_size}) {
        my $evict = shift @{$self->{_order}};
        delete $self->{_store}{$evict};
        $self->{_stats}{evictions}++;
    }
    
    $self->{_store}{$key} = {
        value   => $value,
        expires => $ttl > 0 ? time() + $ttl : 0,
        created => time(),
    };
    push @{$self->{_order}}, $key;
    
    return $self;
}

sub get {
    my ($self, $key) = @_;
    
    my $entry = $self->{_store}{$key};
    
    unless ($entry) {
        $self->{_stats}{misses}++;
        return undef;
    }
    
    # Check TTL
    if ($entry->{expires} && $entry->{expires} < time()) {
        $self->delete($key);
        $self->{_stats}{misses}++;
        return undef;
    }
    
    # LRU: move to end
    $self->{_order} = [grep { $_ ne $key } @{$self->{_order}}];
    push @{$self->{_order}}, $key;
    
    $self->{_stats}{hits}++;
    return $entry->{value};
}

sub delete {
    my ($self, $key) = @_;
    return unless exists $self->{_store}{$key};
    $self->{_order} = [grep { $_ ne $key } @{$self->{_order}}];
    delete $self->{_store}{$key};
    return $self;
}

sub has    { my ($self,$k)=@_; exists $self->{_store}{$k} && (!$self->{_store}{$k}{expires} || $self->{_store}{$k}{expires} >= time()) }
sub size   { scalar @{$_[0]->{_order}} }
sub clear  { $_[0]->{_store} = {}; $_[0]->{_order} = []; $_[0] }
sub keys   { @{$_[0]->{_order}} }
sub stats  { %{$_[0]->{_stats}} }

sub get_or_set {
    my ($self, $key, $builder, $ttl) = @_;
    my $val = $self->get($key);
    unless (defined $val) {
        $val = $builder->();
        $self->set($key, $val, $ttl);
    }
    return $val;
}

sub info {
    my $self = shift;
    my %s    = $self->stats;
    my $total = $s{hits} + $s{misses};
    my $rate  = $total ? sprintf("%.1f%%", $s{hits}/$total*100) : "N/A";
    return {
        size      => $self->size,
        max_size  => $self->{max_size},
        hit_rate  => $rate,
        %s,
    };
}
}

package main;

my $cache = Cache::Simple->new(max_size => 5, ttl => 60);

# Basic operations
$cache->set("user:1", { name => "Alice", age => 28 });
$cache->set("user:2", { name => "Bob",   age => 35 });
$cache->set("config", { debug => 1, version => "1.0" });

printf "User 1: %s\n", $cache->get("user:1")->{name};
printf "User 2: %s\n", $cache->get("user:2")->{name};
printf "Size: %d\n",   $cache->size;
printf "Has user:1: %s\n", $cache->has("user:1") ? "yes" : "no";
printf "Has user:9: %s\n", $cache->has("user:9") ? "yes" : "no";

# get_or_set
my $expensive = $cache->get_or_set("computed", sub {
    printf "(computing expensive value)\n";
    return sqrt(2) * 100;
}, 300);
printf "Computed: %.2f\n", $expensive;

# Second call: cached
my $cached = $cache->get_or_set("computed", sub { die "should not compute" }, 300);
printf "Cached:   %.2f\n", $cached;

# Fill cache to test eviction
$cache->set("item:$_", "value_$_") for 3..6;
printf "Size after fill: %d (max: 5)\n", $cache->size;

# Stats
my $info = $cache->info;
printf "\nCache info:\n";
printf "  size:     %d/%d\n", $info->{size}, $info->{max_size};
printf "  hits:     %d\n", $info->{hits};
printf "  misses:   %d\n", $info->{misses};
printf "  evictions: %d\n", $info->{evictions};
printf "  hit_rate: %s\n", $info->{hit_rate};
```

---

## Step 284: Module with Config

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Module with configuration
# =====================

{
package App::Config;
use strict;
use warnings;
our $VERSION = '1.00';

my %_defaults = (
    debug      => 0,
    log_level  => 'info',
    max_conn   => 100,
    timeout    => 30,
    encoding   => 'UTF-8',
);

my %_valid_levels = map { $_ => 1 } qw(debug info warn error fatal);

sub new {
    my ($class, %args) = @_;
    my %data = (%_defaults, %args);
    return bless { _data => \%data, _locked => 0 }, $class;
}

sub get {
    my ($self, $key) = @_;
    return $self->{_data}{$key};
}

sub set {
    my ($self, $key, $value) = @_;
    die "Config is locked\n" if $self->{_locked};
    die "Unknown key: $key\n" unless exists $_defaults{$key} || $key =~ /^[a-z_]\w*$/;
    
    # Validate specific keys
    if ($key eq 'log_level') {
        die "Invalid log level: $value\n" unless $_valid_levels{$value};
    }
    if ($key =~ /^(max_conn|timeout)$/) {
        die "$key must be a positive integer\n" unless $value =~ /^\d+$/ && $value > 0;
    }
    
    $self->{_data}{$key} = $value;
    return $self;
}

sub lock    { $_[0]->{_locked} = 1; $_[0] }
sub is_locked { $_[0]->{_locked} }

sub to_hash { %{$_[0]->{_data}} }

sub load_file {
    my ($self, $file) = @_;
    open my $fh, '<', $file or die "Cannot open $file: $!\n";
    while (<$fh>) {
        chomp; s/#.*//; s/^\s+|\s+$//g;
        next unless length;
        my ($k, $v) = split /\s*=\s*/, $_, 2;
        eval { $self->set($k, $v) };
        warn "Config: $@" if $@;
    }
    close $fh;
    return $self;
}

sub dump {
    my $self = shift;
    my %d = $self->to_hash;
    return join("\n", map { "$_ = $d{$_}" } sort keys %d);
}
}

package main;

my $cfg = App::Config->new(debug => 1, log_level => "debug");
printf "debug: %s\n",     $cfg->get("debug");
printf "log_level: %s\n", $cfg->get("log_level");

$cfg->set("max_conn", 200);
$cfg->set("timeout",  60);
printf "max_conn: %s\n", $cfg->get("max_conn");

# Validate
eval { $cfg->set("log_level", "verbose") };
printf "Invalid level error: %s", $@ if $@;

eval { $cfg->set("timeout", -5) };
printf "Invalid timeout: %s", $@ if $@;

# Lock
$cfg->lock;
eval { $cfg->set("debug", 0) };
printf "Locked error: %s", $@ if $@;

printf "\nConfig dump:\n%s\n", $cfg->dump;
```

---

## Step 285: Exception Hierarchy

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Scalar::Util qw(blessed);

# =====================
# Exception class hierarchy
# =====================

{
package Exception;
use overload '""' => \&to_string, fallback => 1;

sub new {
    my ($class, %args) = @_;
    return bless {
        message   => $args{message} // "Unknown error",
        code      => $args{code}    // 0,
        stack     => $args{stack}   // _capture_stack(),
        cause     => $args{cause},
    }, $class;
}

sub _capture_stack {
    my @stack;
    my $i = 1;
    while (my @frame = caller($i++)) {
        push @stack, sprintf "%s line %d", $frame[1], $frame[2];
        last if @stack >= 5;
    }
    return \@stack;
}

sub message { $_[0]->{message} }
sub code    { $_[0]->{code}    }
sub cause   { $_[0]->{cause}   }
sub stack   { @{$_[0]->{stack}} }

sub throw {
    my ($class, %args) = @_;
    die $class->new(%args);
}

sub to_string {
    my $self = shift;
    return sprintf "[%s] %s (code: %d)", ref($self), $self->{message}, $self->{code};
}

sub rethrow { die shift }
}

{
package IOException;
our @ISA = ('Exception');
sub new {
    my ($class, %args) = @_;
    $args{code} //= 500;
    return $class->SUPER::new(%args);
}
}

{
package NetworkException;
our @ISA = ('IOException');
sub new {
    my ($class, %args) = @_;
    my $self = $class->SUPER::new(%args);
    $self->{url}    = $args{url};
    $self->{status} = $args{status};
    return $self;
}
sub url    { $_[0]->{url} }
sub status { $_[0]->{status} }
}

{
package ValidationException;
our @ISA = ('Exception');
sub new {
    my ($class, %args) = @_;
    my $self = $class->SUPER::new(%args);
    $self->{field}  = $args{field};
    $self->{errors} = $args{errors} // [];
    return $self;
}
sub field  { $_[0]->{field} }
sub errors { @{$_[0]->{errors}} }
}

{
package AuthException;
our @ISA = ('Exception');
sub new {
    my ($class, %args) = @_;
    $args{code} //= 401;
    return $class->SUPER::new(%args);
}
}

{
package NotFoundException;
our @ISA = ('Exception');
sub new {
    my ($class, %args) = @_;
    $args{code} //= 404;
    return $class->SUPER::new(%args);
}
}

# =====================
# Try/catch utility
# =====================

sub try (&) { my $code = shift; $code }

sub catch {
    my ($try, %handlers) = @_;
    eval { $try->() };
    if (my $e = $@) {
        for my $type (keys %handlers) {
            if (blessed($e) && $e->isa($type)) {
                return $handlers{$type}->($e);
            }
        }
        # Default handler
        if ($handlers{default}) {
            return $handlers{default}->($e);
        }
        die $e;  # re-throw
    }
}

# =====================
# Demo
# =====================

package main;

# Function that throws various exceptions
sub fetch_resource {
    my ($type, $id) = @_;
    
    die AuthException->new(message => "Not authenticated")
        if $type eq "private";
    
    die NotFoundException->new(message => "Resource $id not found")
        if $id > 100;
    
    die NetworkException->new(
        message => "Connection failed",
        url     => "https://api.example.com/$id",
        status  => 503,
    ) if $type eq "network";
    
    die ValidationException->new(
        message => "Invalid input",
        field   => "id",
        errors  => ["Must be positive", "Must be integer"],
    ) if $id < 0;
    
    return { id => $id, type => $type, data => "..." };
}

my @tests = (
    ["public", 42],
    ["private", 1],
    ["public", 999],
    ["network", 50],
    ["public", -1],
);

for my $test (@tests) {
    my ($type, $id) = @$test;
    
    catch try { fetch_resource($type, $id) },
        AuthException       => sub { printf "Auth: %s\n",   $_[0]->message },
        NotFoundException   => sub { printf "NotFound: %s (code=%d)\n", $_[0]->message, $_[0]->code },
        NetworkException    => sub { printf "Network: %s status=%d\n", $_[0]->message, $_[0]->status },
        ValidationException => sub { printf "Validation: %s field=%s\n", $_[0]->message, $_[0]->field },
        default             => sub { printf "Resource: %s\n", ref($_[0]) || "OK" };
}

# Successful
my $result = eval { fetch_resource("public", 42) };
printf "Success: id=%d\n", $result->{id} unless $@;
```

---

## Step 286: Module Testing

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Test::More;

# =====================
# Test a complete module
# =====================

{
package Stack::Simple;

sub new   { bless { items => [], max_size => $_[1]//1000 }, $_[0] }
sub push  { my($s,$v)=@_; die "Stack full\n" if $s->size>=$s->{max_size}; push @{$s->{items}},$v; $s }
sub pop   { @{$_[0]->{items}} ? CORE::pop @{$_[0]->{items}} : undef }
sub peek  { $_[0]->{items}[-1] }
sub size  { scalar @{$_[0]->{items}} }
sub empty { !@{$_[0]->{items}} }
sub clear { $_[0]->{items}=[]; $_[0] }
sub to_array { @{$_[0]->{items}} }
}

# =====================
# Comprehensive tests
# =====================

subtest 'constructor' => sub {
    my $s = Stack::Simple->new;
    ok(defined $s, "can create stack");
    isa_ok($s, 'Stack::Simple', "correct class");
    ok($s->empty, "new stack is empty");
    is($s->size, 0, "new stack size is 0");
    ok(!defined $s->peek, "peek on empty is undef");
    ok(!defined $s->pop,  "pop on empty is undef");
};

subtest 'push and pop' => sub {
    my $s = Stack::Simple->new;
    $s->push(1);
    is($s->size, 1, "size after push");
    ok(!$s->empty, "not empty");
    is($s->peek, 1, "peek returns top");
    
    $s->push(2)->push(3);
    is($s->size, 3, "size after 3 pushes");
    is($s->peek, 3, "peek returns latest");
    
    is($s->pop, 3, "pop returns top");
    is($s->pop, 2, "pop returns next");
    is($s->pop, 1, "pop returns first");
    ok($s->empty, "empty after all pops");
};

subtest 'max_size' => sub {
    my $s = Stack::Simple->new(3);
    $s->push($_) for 1..3;
    is($s->size, 3, "size at max");
    
    eval { $s->push(4) };
    like($@, qr/full/i, "push on full dies");
    is($s->size, 3, "size unchanged");
};

subtest 'clear' => sub {
    my $s = Stack::Simple->new;
    $s->push($_) for 1..5;
    $s->clear;
    ok($s->empty, "empty after clear");
    is($s->size, 0, "size 0 after clear");
};

subtest 'to_array' => sub {
    my $s = Stack::Simple->new;
    $s->push($_) for 1..5;
    my @arr = $s->to_array;
    is_deeply(\@arr, [1,2,3,4,5], "to_array returns elements");
    is($s->size, 5, "to_array doesn't remove");
};

subtest 'LIFO order' => sub {
    my $s = Stack::Simple->new;
    $s->push($_) for 1..5;
    my @popped;
    push @popped, $s->pop while !$s->empty;
    is_deeply(\@popped, [5,4,3,2,1], "LIFO order correct");
};

subtest 'method chaining' => sub {
    my $s = Stack::Simple->new;
    my $ret = $s->push(1)->push(2)->push(3);
    isa_ok($ret, 'Stack::Simple', "push returns self");
    is($s->size, 3, "all items pushed");
    
    my $ret2 = $s->clear;
    isa_ok($ret2, 'Stack::Simple', "clear returns self");
};

done_testing;
```

---

## Step 287: Distribution Structure

```perl
#!/usr/bin/perl
# =====================
# CPAN distribution structure demo
# Shows what a typical CPAN module distribution looks like
# =====================

use strict;
use warnings;

my $structure = <<'TREE';
My-Module-1.00/
├── lib/
│   └── My/
│       ├── Module.pm          # Main module
│       └── Module/
│           ├── Config.pm      # Submodule
│           └── Utils.pm       # Utilities
├── t/                         # Tests
│   ├── 00-load.t              # Test module loads
│   ├── 01-basic.t             # Basic functionality
│   ├── 02-config.t            # Config tests
│   └── 99-pod.t               # POD tests
├── Makefile.PL                # Build system
├── MANIFEST                   # File list
├── META.yml                   # Module metadata
├── README                     # Documentation
├── Changes                    # Changelog
└── COPYING                    # License
TREE

print "CPAN Distribution Structure:\n";
print $structure;

# =====================
# Makefile.PL
# =====================

my $makefile_pl = <<'MAKEFILE';
use ExtUtils::MakeMaker;

WriteMakefile(
    NAME          => 'My::Module',
    VERSION_FROM  => 'lib/My/Module.pm',
    ABSTRACT_FROM => 'lib/My/Module.pm',
    AUTHOR        => 'Your Name <you@example.com>',
    LICENSE       => 'perl',
    MIN_PERL_VERSION => '5.010',
    PREREQ_PM     => {
        'strict'         => 0,
        'warnings'       => 0,
        'Scalar::Util'   => 0,
        'List::Util'     => 0,
    },
    TEST_REQUIRES => {
        'Test::More'     => '0.98',
    },
    META_MERGE    => {
        resources => {
            repository => 'https://github.com/yourname/My-Module',
            bugtracker => 'https://rt.cpan.org/Public/Dist/Display.html?Name=My-Module',
        },
    },
);
MAKEFILE

print "\nMakefile.PL:\n$makefile_pl";

# =====================
# META.yml
# =====================

my $meta_yml = <<'META';
---
name: My-Module
version: 1.00
author:
  - 'Your Name <you@example.com>'
abstract: A sample CPAN module
license: perl
requires:
  perl: 5.010
  Scalar::Util: 0
  List::Util: 0
build_requires:
  Test::More: 0.98
resources:
  repository: https://github.com/yourname/My-Module
META

print "META.yml:\n$meta_yml";

# =====================
# Changes file
# =====================

my $changes = <<'CHANGES';
Revision history for My-Module

1.00  2024-01-15
    - Initial release
    - Basic module structure
    - Core functionality implemented
    - Tests added

0.99  2024-01-10
    - Beta release
    - Internal testing only
CHANGES

print "Changes:\n$changes";

# =====================
# Build workflow
# =====================

print "Build workflow:\n";
print "  perl Makefile.PL\n";
print "  make\n";
print "  make test\n";
print "  make install\n";
print "\nor with Module::Build:\n";
print "  perl Build.PL\n";
print "  ./Build\n";
print "  ./Build test\n";
print "  ./Build install\n";
```

---

## Step 288: POD Documentation

```perl
#!/usr/bin/perl
# =====================
# POD — Plain Old Documentation
# =====================

use strict;
use warnings;
use Pod::Text;

my $pod = <<'END_POD';
=pod

=encoding UTF-8

=head1 NAME

My::Logger - A simple logging module

=head1 VERSION

Version 1.00

=head1 SYNOPSIS

    use My::Logger;

    my $log = My::Logger->new(level => 'debug');
    $log->debug("Starting application");
    $log->info("Server started on port 8080");
    $log->warn("High memory usage: 85%");
    $log->error("Database connection failed");

=head1 DESCRIPTION

My::Logger provides a simple, flexible logging interface for Perl applications.
It supports multiple log levels, output destinations, and formatting options.

=head1 METHODS

=head2 new(%args)

Creates a new logger instance.

    my $log = My::Logger->new(
        level  => 'info',      # Minimum log level
        output => 'STDERR',    # Output destination
        format => '[%level] %message',
    );

Arguments:

=over 4

=item level

Minimum log level to output. One of: debug, info, warn, error, fatal.
Default: 'info'.

=item output

Where to write log output. Can be 'STDERR', 'STDOUT', or a filename.
Default: 'STDERR'.

=item format

Log line format string. Default: '[%Y-%m-%d %H:%M:%S] [%level] %message'.

=back

=head2 debug($message)

Log a debug message.

=head2 info($message)

Log an informational message.

=head2 warn($message)

Log a warning message.

=head2 error($message)

Log an error message.

=head2 fatal($message)

Log a fatal error and die.

=head1 CONFIGURATION

=head2 Log Levels

The following log levels are supported (in increasing severity):

    debug < info < warn < error < fatal

=head1 EXAMPLES

=head2 Basic usage

    my $log = My::Logger->new;
    $log->info("Application started");

=head2 Debug mode

    my $log = My::Logger->new(level => 'debug');
    $log->debug("Entering function foo");

=head1 BUGS AND LIMITATIONS

Please report bugs at L<https://github.com/yourname/My-Logger/issues>.

=head1 SEE ALSO

L<Log::Log4perl>, L<Log::Any>

=head1 AUTHOR

Your Name E<lt>you@example.comE<gt>

=head1 LICENSE

This library is free software; you can redistribute it and/or modify
it under the same terms as Perl itself.

=cut
END_POD

# Convert POD to text
my $text_output = "";
open my $in,  '<', \$pod;
open my $out, '>', \$text_output;

my $parser = Pod::Text->new(width => 70, indent => 2);
$parser->parse_from_filehandle($in, $out);

close $in;
close $out;

printf "=== POD Documentation (text format) ===\n";
print $text_output;
```

---

## Step 289: Module Versioning

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Version numbers in Perl
# =====================

# Declaring versions
package My::App;
our $VERSION = '2.1.3';

package My::App::Database;
our $VERSION = '2.1.0';

package My::App::Cache;
our $VERSION = '1.5.2';

package main;

# Version comparison
use version;

my @versions = ('1.0', '1.10', '2.0', '1.9.1', '2.1.3');

my @sorted = sort { version->parse($a) <=> version->parse($b) } @versions;
printf "Sorted versions: %s\n", join(", ", @sorted);

# Check minimum version
sub requires_version {
    my ($module_version, $required) = @_;
    return version->parse($module_version) >= version->parse($required);
}

for my $req (qw(1.0 2.0 2.1.3 2.2.0)) {
    my $ok = requires_version($My::App::VERSION, $req);
    printf "My::App %s >= %s: %s\n", $My::App::VERSION, $req, $ok ? "yes" : "no";
}

# =====================
# Module compatibility checking
# =====================

{
package CompatCheck;

sub check_deps {
    my %required = @_;
    my @missing;
    my @version_mismatch;
    
    for my $module (sort keys %required) {
        my $min_ver = $required{$module};
        
        eval "require $module";
        if ($@) {
            push @missing, $module;
            next;
        }
        
        my $installed = eval { $module->VERSION } // 0;
        if ($min_ver && version->parse($installed) < version->parse($min_ver)) {
            push @version_mismatch, "$module (have $installed, need $min_ver)";
        }
    }
    
    return (missing => \@missing, mismatch => \@version_mismatch);
}
}

my %deps = (
    'Scalar::Util'  => '1.27',
    'List::Util'    => '1.33',
    'POSIX'         => '0',
    'JSON::PP'      => '2.0',
    'DoesNotExist'  => '1.0',
);

my %check = CompatCheck::check_deps(%deps);

printf "\nDependency check:\n";
printf "  Missing: %s\n",  join(", ", @{$check{missing}})  || "(none)";
printf "  Mismatch: %s\n", join(", ", @{$check{mismatch}}) || "(none)";

# Modules that loaded OK
my @ok = grep {
    !grep { $_ eq $_ } @{$check{missing}}, @{$check{mismatch}}
} keys %deps;
printf "  OK: %s\n", join(", ", sort @ok);
```

---

## Step 290: โปรแกรมสรุป — Complete Module Package

```perl
#!/usr/bin/perl
# complete_module.pl — Production-ready Perl module
use strict;
use warnings;

# =====================
# My::DataStore — a production-ready module
# =====================

{
package My::DataStore;

use strict;
use warnings;
our $VERSION = '1.00';

use Storable qw(dclone);
use Scalar::Util qw(blessed looks_like_number);
use Carp qw(croak confess);

use constant {
    MAX_KEY_LENGTH => 255,
    MAX_SIZE       => 10_000,
};

sub new {
    my ($class, %args) = @_;
    
    croak "max_size must be positive" 
        if $args{max_size} && ($args{max_size} !~ /^\d+$/ || $args{max_size} <= 0);
    
    return bless {
        _store    => {},
        _meta     => {},
        max_size  => $args{max_size}  // MAX_SIZE,
        on_change => $args{on_change} // sub {},
        _version  => 0,
    }, $class;
}

sub _validate_key {
    my ($self, $key) = @_;
    croak "Key cannot be undef"                if !defined $key;
    croak "Key cannot be empty"                if $key eq "";
    croak "Key too long (max " . MAX_KEY_LENGTH . ")" if length($key) > MAX_KEY_LENGTH;
    croak "Key must be a string"               if ref $key;
}

sub set {
    my ($self, $key, $value, %opts) = @_;
    $self->_validate_key($key);
    
    croak "Store is full (max: $self->{max_size})"
        if !exists $self->{_store}{$key} && $self->size >= $self->{max_size};
    
    my $old    = $self->{_store}{$key};
    my $is_new = !exists $self->{_store}{$key};
    
    $self->{_store}{$key} = dclone(ref $value ? $value : \$value);
    $self->{_store}{$key} = $${ $self->{_store}{$key} } unless ref $value;
    
    $self->{_meta}{$key} = {
        created  => $is_new ? time() : ($self->{_meta}{$key}{created} // time()),
        updated  => time(),
        version  => ($self->{_meta}{$key}{version}//0) + 1,
        tags     => $opts{tags} // $self->{_meta}{$key}{tags} // [],
    };
    
    $self->{_version}++;
    $self->{on_change}->($is_new ? "set" : "update", $key, $value);
    
    return $self;
}

sub get {
    my ($self, $key, $default) = @_;
    $self->_validate_key($key);
    return $default unless exists $self->{_store}{$key};
    my $v = $self->{_store}{$key};
    return ref $v ? dclone($v) : $v;
}

sub delete {
    my ($self, $key) = @_;
    $self->_validate_key($key);
    return unless exists $self->{_store}{$key};
    delete $self->{_store}{$key};
    delete $self->{_meta}{$key};
    $self->{_version}++;
    $self->{on_change}->("delete", $key, undef);
    return $self;
}

sub has   { my($s,$k)=@_; $s->_validate_key($k); exists $s->{_store}{$k} }
sub size  { scalar keys %{$_[0]->{_store}} }
sub keys  { CORE::keys %{$_[0]->{_store}} }
sub clear { my$s=shift; $s->{_store}={}; $s->{_meta}={}; $s->{_version}++; $s }

sub find_by_tag {
    my ($self, $tag) = @_;
    return grep {
        my $tags = $self->{_meta}{$_}{tags} // [];
        grep { $_ eq $tag } @$tags;
    } $self->keys;
}

sub meta { $_[0]->{_meta}{$_[1]} }

sub update {
    my ($self, $key, $updater) = @_;
    croak "updater must be a coderef" unless ref $updater eq 'CODE';
    my $current = $self->get($key);
    my $new = $updater->($current);
    $self->set($key, $new);
    return $self;
}

sub each {
    my ($self, $cb) = @_;
    $cb->($_, $self->get($_)) for $self->keys;
    return $self;
}

sub grep_keys {
    my ($self, $pred) = @_;
    return grep { $pred->($_, $self->get($_)) } $self->keys;
}

sub map_values {
    my ($self, $fn) = @_;
    my %result;
    $result{$_} = $fn->($self->get($_)) for $self->keys;
    return %result;
}

sub dump {
    my $self = shift;
    my @lines = ("DataStore v=$self->{_version} size=${\$self->size}");
    for my $k (sort $self->keys) {
        my $v = $self->get($k);
        my $meta = $self->meta($k);
        push @lines, sprintf "  %-20s = %s  [v%d]",
            $k, (ref $v ? "(ref)" : $v), $meta->{version};
    }
    return join "\n", @lines;
}
}

package main;

# Create with change listener
my @change_log;
my $store = My::DataStore->new(
    max_size  => 100,
    on_change => sub { push @change_log, "@_[0..1]" },
);

# Basic CRUD
$store->set("user:1", { name => "Alice", age => 28 }, tags => ["user", "active"]);
$store->set("user:2", { name => "Bob",   age => 35 }, tags => ["user"]);
$store->set("config", { debug => 1, version => "1.0" }, tags => ["config"]);

printf "Size: %d\n", $store->size;
printf "Has user:1: %s\n", $store->has("user:1") ? "yes" : "no";

my $alice = $store->get("user:1");
printf "Alice: %s, age %d\n", $alice->{name}, $alice->{age};

# Update
$store->update("user:1", sub {
    my $u = shift;
    $u->{age}++;
    return $u;
});
printf "Alice new age: %d\n", $store->get("user:1")->{age};

# Find by tag
my @users = $store->find_by_tag("user");
printf "Users: %s\n", join(", ", sort @users);

# Iterate
printf "\nAll entries:\n";
$store->each(sub {
    my ($k, $v) = @_;
    my $display = ref $v ? "{name=>" . ($v->{name}//"?") . "}" : $v;
    printf "  %s => %s\n", $k, $display;
});

# Filter
my @version_keys = $store->grep_keys(sub {
    my ($k, $v) = @_;
    ref $v && exists $v->{version};
});
printf "Keys with 'version': %s\n", join(", ", @version_keys);

# Change log
printf "\nChange log:\n";
printf "  %s\n", $_ for @change_log;

# Meta
my $meta = $store->meta("user:1");
printf "\nuser:1 meta: version=%d tags=%s\n",
    $meta->{version}, join(",", @{$meta->{tags}});

# Dump
printf "\n%s\n", $store->dump;

print "\nComplete module demo done!\n";
```

---

## สรุป Part 29

ใน Part นี้คุณได้เรียนรู้:
- ✅ Module structure: package, $VERSION, 1;
- ✅ Exporter: @EXPORT, @EXPORT_OK, %EXPORT_TAGS
- ✅ OOP module with full interface
- ✅ Module configuration
- ✅ Exception hierarchy
- ✅ Module testing
- ✅ CPAN distribution structure (Makefile.PL, META.yml)
- ✅ POD documentation
- ✅ Version management
- ✅ Production-ready DataStore module

**ถัดไป: [Part 30 — Intermediate Capstone Project](part_30.md)**
