# Part 14: Modules และ Packages
## Steps 131-140: การสร้างและใช้ Modules

---

## Step 131: Package Basics

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Package (namespace)
# =====================

package Animal;

# variables ใน package Animal
our $count = 0;
my $private = "secret";   # lexical — ไม่ expose ออกมา

sub new {
    my ($class, %args) = @_;
    $count++;
    return bless {
        name    => $args{name} // "Unknown",
        sound   => $args{sound} // "...",
        species => $args{species} // "unknown",
    }, $class;
}

sub name    { $_[0]->{name} }
sub sound   { $_[0]->{sound} }
sub species { $_[0]->{species} }

sub speak {
    my $self = shift;
    printf "%s says: %s!\n", $self->{name}, $self->{sound};
}

sub describe {
    my $self = shift;
    printf "%s is a %s\n", $self->{name}, $self->{species};
}

# =====================
# Package Dog (inherits Animal)
# =====================

package Dog;
our @ISA = ('Animal');   # inheritance

sub new {
    my ($class, %args) = @_;
    $args{sound}   //= "Woof";
    $args{species} //= "Canis lupus familiaris";
    my $self = Animal::new($class, %args);
    $self->{breed} = $args{breed} // "Mixed";
    return $self;
}

sub fetch { print $_[0]->{name}, " fetches the ball!\n" }
sub breed { $_[0]->{breed} }

# =====================
# Back to main
# =====================

package main;

my $dog = Dog->new(name => "Rex", breed => "Labrador");
$dog->speak;
$dog->describe;
$dog->fetch;
printf "Breed: %s\n", $dog->breed;
printf "Total animals: %d\n", $Animal::count;

my $cat = Animal->new(name => "Whiskers", sound => "Meow", species => "Felis catus");
$cat->speak;
$cat->describe;
printf "Total animals: %d\n", $Animal::count;
```

---

## Step 132: สร้าง .pm Module

```perl
# File: lib/MathUtils.pm
package MathUtils;

use strict;
use warnings;
use POSIX qw(floor ceil);
use List::Util qw(sum min max);
use Carp qw(croak);

our $VERSION = '1.0.0';

# =====================
# Export interface
# =====================

use Exporter 'import';

our @EXPORT    = ();                # exported by default
our @EXPORT_OK = qw(
    add subtract multiply divide
    factorial fibonacci
    is_prime primes_up_to
    gcd lcm
    mean median mode stddev
);

our %EXPORT_TAGS = (
    basic  => [qw(add subtract multiply divide)],
    number => [qw(factorial fibonacci is_prime primes_up_to gcd lcm)],
    stats  => [qw(mean median mode stddev)],
);

# =====================
# Basic arithmetic
# =====================

sub add      { $_[0] + $_[1] }
sub subtract { $_[0] - $_[1] }
sub multiply { $_[0] * $_[1] }

sub divide {
    croak "Division by zero" if $_[1] == 0;
    return $_[0] / $_[1];
}

# =====================
# Number theory
# =====================

sub factorial {
    my $n = shift;
    croak "Factorial requires non-negative integer" if $n < 0 || $n != int($n);
    return 1 if $n <= 1;
    my $result = 1;
    $result *= $_ for 2..$n;
    return $result;
}

sub fibonacci {
    my $n = shift;
    return $n if $n <= 1;
    my ($a, $b) = (0, 1);
    ($a, $b) = ($b, $a + $b) for 2..$n;
    return $b;
}

sub is_prime {
    my $n = shift;
    return 0 if $n < 2;
    return 1 if $n == 2;
    return 0 if $n % 2 == 0;
    for (my $i = 3; $i * $i <= $n; $i += 2) {
        return 0 if $n % $i == 0;
    }
    return 1;
}

sub primes_up_to {
    my $limit = shift;
    return grep { is_prime($_) } 2..$limit;
}

sub gcd {
    my ($a, $b) = @_;
    ($a, $b) = ($b, $a % $b) while $b;
    return abs $a;
}

sub lcm {
    my ($a, $b) = @_;
    return abs($a * $b) / gcd($a, $b);
}

# =====================
# Statistics
# =====================

sub mean {
    return 0 unless @_;
    return sum(@_) / scalar @_;
}

sub median {
    my @sorted = sort { $a <=> $b } @_;
    my $n = scalar @sorted;
    return $sorted[$n/2] if $n % 2;
    return ($sorted[$n/2 - 1] + $sorted[$n/2]) / 2;
}

sub mode {
    my %freq;
    $freq{$_}++ for @_;
    my $max = max(values %freq);
    return grep { $freq{$_} == $max } keys %freq;
}

sub stddev {
    return 0 unless @_ > 1;
    my $mean = mean(@_);
    my $sum_sq = 0;
    $sum_sq += ($_ - $mean) ** 2 for @_;
    return sqrt($sum_sq / (@_ - 1));
}

1;  # must return true
```

---

## Step 133: ใช้ Module และ Exporter

```perl
#!/usr/bin/perl
use strict;
use warnings;
use lib 'lib';   # add lib/ to @INC

# =====================
# Using the module
# =====================

use MathUtils qw(:basic factorial is_prime mean stddev);

# Basic math
printf "3 + 4 = %d\n",     add(3, 4);
printf "10 / 3 = %.3f\n",  divide(10, 3);

# Number theory
printf "10! = %d\n", factorial(10);
printf "Primes up to 20: %s\n", join(", ", MathUtils::primes_up_to(20));

# Stats
my @data = (2, 4, 4, 4, 5, 5, 7, 9);
printf "Mean:   %.2f\n", mean(@data);
printf "StdDev: %.2f\n", stddev(@data);

# =====================
# Check VERSION
# =====================

printf "MathUtils version: %s\n", $MathUtils::VERSION;

# =====================
# Module with OO interface
# =====================

package Stats;

sub new {
    my ($class, @data) = @_;
    return bless { data => [@data] }, $class;
}

sub add_data {
    my ($self, @more) = @_;
    push @{$self->{data}}, @more;
    return $self;  # for chaining
}

sub count  { scalar @{$_[0]->{data}} }
sub sum    { do { my $s = 0; $s += $_ for @{$_[0]->{data}}; $s } }
sub mean   { $_[0]->sum / $_[0]->count }
sub min    { (sort { $a <=> $b } @{$_[0]->{data}})[0] }
sub max    { (sort { $b <=> $a } @{$_[0]->{data}})[0] }

sub report {
    my $self = shift;
    printf "Count: %d, Sum: %.2f, Mean: %.2f, Min: %.2f, Max: %.2f\n",
        $self->count, $self->sum, $self->mean, $self->min, $self->max;
}

package main;

my $s = Stats->new(1..10);
$s->add_data(11..20);
$s->report;
```

---

## Step 134: Object-Oriented Perl (Bless)

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Full OOP example
# =====================

package Shape;

sub new {
    my ($class, %args) = @_;
    my $self = {
        color  => $args{color}  // "white",
        filled => $args{filled} // 0,
    };
    return bless $self, $class;
}

sub color   { 
    my $self = shift;
    $self->{color} = shift if @_;
    return $self->{color};
}

sub filled { $_[0]->{filled} }
sub area   { 0 }   # abstract
sub perimeter { 0 }  # abstract

sub describe {
    my $self = shift;
    printf "%s: color=%s, area=%.2f, perimeter=%.2f\n",
        ref $self, $self->{color}, $self->area, $self->perimeter;
}

# =====================
# Circle
# =====================

package Circle;
our @ISA = ('Shape');
use POSIX qw();

sub new {
    my ($class, %args) = @_;
    my $self = Shape::new($class, %args);
    $self->{radius} = $args{radius} // 1;
    return $self;
}

sub radius    { $_[0]->{radius} }
sub area      { 3.14159265358979 * $_[0]->{radius} ** 2 }
sub perimeter { 2 * 3.14159265358979 * $_[0]->{radius} }

# =====================
# Rectangle
# =====================

package Rectangle;
our @ISA = ('Shape');

sub new {
    my ($class, %args) = @_;
    my $self = Shape::new($class, %args);
    $self->{width}  = $args{width}  // 1;
    $self->{height} = $args{height} // 1;
    return $self;
}

sub width     { $_[0]->{width} }
sub height    { $_[0]->{height} }
sub area      { $_[0]->{width} * $_[0]->{height} }
sub perimeter { 2 * ($_[0]->{width} + $_[0]->{height}) }

# =====================
# Square (inherits Rectangle)
# =====================

package Square;
our @ISA = ('Rectangle');

sub new {
    my ($class, %args) = @_;
    $args{height} = $args{side} // $args{width} // 1;
    $args{width}  = $args{height};
    return Rectangle::new($class, %args);
}

# =====================
# Triangle
# =====================

package Triangle;
our @ISA = ('Shape');

sub new {
    my ($class, %args) = @_;
    my $self = Shape::new($class, %args);
    $self->{a} = $args{a} // 3;
    $self->{b} = $args{b} // 4;
    $self->{c} = $args{c} // 5;
    return $self;
}

sub area {
    my $self = shift;
    my ($a, $b, $c) = @{$self}{qw(a b c)};
    my $s = ($a + $b + $c) / 2;
    return sqrt($s * ($s-$a) * ($s-$b) * ($s-$c));
}

sub perimeter { $_[0]->{a} + $_[0]->{b} + $_[0]->{c} }

# =====================
# Test
# =====================

package main;

my @shapes = (
    Circle->new(radius => 5, color => "red"),
    Rectangle->new(width => 4, height => 6, color => "blue"),
    Square->new(side => 5, color => "green"),
    Triangle->new(a => 3, b => 4, c => 5, color => "yellow"),
);

print "=== Shapes ===\n";
$_->describe for @shapes;

# Polymorphism
my $total_area = 0;
$total_area += $_->area for @shapes;
printf "\nTotal area: %.2f\n", $total_area;

# isa check
for my $shape (@shapes) {
    printf "%-15s is_a Circle: %s, is_a Shape: %s\n",
        ref $shape,
        $shape->isa('Circle')    ? "yes" : "no",
        $shape->isa('Shape')     ? "yes" : "no";
}
```

---

## Step 135: AUTOLOAD และ DESTROY

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# AUTOLOAD — catch undefined methods
# =====================

package AutoClass;

our $AUTOLOAD;

sub new {
    my ($class, %data) = @_;
    return bless \%data, $class;
}

sub AUTOLOAD {
    my $self = shift;
    my $method = $AUTOLOAD;
    $method =~ s/.*:://;   # strip package name
    
    # Skip DESTROY
    return if $method eq 'DESTROY';
    
    # Auto-generate accessor
    if (exists $self->{$method}) {
        if (@_) {
            $self->{$method} = $_[0];
        }
        return $self->{$method};
    }
    
    die "Method $method not found in " . ref($self) . "\n";
}

package main;

my $obj = AutoClass->new(name => "Alice", age => 28, city => "Bangkok");

print $obj->name, "\n";    # Alice
print $obj->age, "\n";     # 28

$obj->city("Chiang Mai");   # setter
print $obj->city, "\n";     # Chiang Mai

# =====================
# DESTROY — destructor
# =====================

package Resource;

my $instance_count = 0;

sub new {
    my ($class, $name) = @_;
    $instance_count++;
    printf "Created %s (total: %d)\n", $name, $instance_count;
    return bless { name => $name }, $class;
}

sub DESTROY {
    my $self = shift;
    $instance_count--;
    printf "Destroyed %s (total: %d)\n", $self->{name}, $instance_count;
}

sub count { $instance_count }

package main;

print "Instances: ", Resource::count(), "\n";  # 0

{
    my $r1 = Resource->new("R1");
    my $r2 = Resource->new("R2");
    print "Instances: ", Resource::count(), "\n";  # 2
    # r1, r2 go out of scope here
}

print "After block: ", Resource::count(), "\n";  # 0

# =====================
# Overloading operators
# =====================

package Vector;

use overload
    '+'   => \&add,
    '-'   => \&subtract,
    '*'   => \&multiply,
    '""'  => \&stringify,
    '=='  => \&equal,
    'abs' => \&magnitude;

sub new {
    my ($class, $x, $y) = @_;
    return bless { x => $x // 0, y => $y // 0 }, $class;
}

sub x { $_[0]->{x} }
sub y { $_[0]->{y} }

sub add {
    my ($a, $b) = @_;
    return Vector->new($a->{x} + $b->{x}, $a->{y} + $b->{y});
}

sub subtract {
    my ($a, $b) = @_;
    return Vector->new($a->{x} - $b->{x}, $a->{y} - $b->{y});
}

sub multiply {
    my ($v, $scalar) = @_;
    return Vector->new($v->{x} * $scalar, $v->{y} * $scalar);
}

sub magnitude {
    my $v = shift;
    return sqrt($v->{x}**2 + $v->{y}**2);
}

sub stringify { "(${\$_[0]->{x}}, ${\$_[0]->{y}})" }

sub equal {
    my ($a, $b) = @_;
    return $a->{x} == $b->{x} && $a->{y} == $b->{y};
}

package main;

my $v1 = Vector->new(3, 4);
my $v2 = Vector->new(1, 2);

print "v1 = $v1\n";           # (3, 4)
print "v2 = $v2\n";           # (1, 2)
print "v1 + v2 = ", $v1 + $v2, "\n";   # (4, 6)
print "v1 - v2 = ", $v1 - $v2, "\n";   # (2, 2)
print "v1 * 2 = ", $v1 * 2, "\n";      # (6, 8)
printf "|v1| = %.2f\n", abs $v1;        # 5.00
```

---

## Step 136: Module::Starter และ CPAN Module Structure

```perl
# File structure สำหรับ CPAN module:
#
# MyModule/
# ├── lib/
# │   └── MyModule.pm
# ├── t/
# │   ├── 00-load.t
# │   └── 01-basic.t
# ├── Changes
# ├── MANIFEST
# ├── Makefile.PL (or Build.PL)
# └── README

# File: lib/Perl/Course/Config.pm
package Perl::Course::Config;

use strict;
use warnings;
use Carp qw(croak carp);

our $VERSION = '0.01';

# =====================
# Constructor
# =====================

sub new {
    my ($class, %opts) = @_;
    
    my $self = bless {
        _data    => {},
        _defaults => $opts{defaults} // {},
        _strict   => $opts{strict}   // 0,
        _schema   => $opts{schema}   // {},
    }, $class;
    
    if (my $file = $opts{file}) {
        $self->load_file($file);
    }
    
    return $self;
}

# =====================
# Load from file
# =====================

sub load_file {
    my ($self, $file) = @_;
    
    -e $file or croak "Config file not found: $file";
    
    open(my $fh, '<:utf8', $file) or croak "Cannot open $file: $!";
    
    while (my $line = <$fh>) {
        chomp $line;
        $line =~ s/#.*$//;            # strip comments
        $line =~ s/^\s+|\s+$//g;     # trim
        next unless length $line;
        
        if ($line =~ /^(\w[\w.]*)\s*=\s*(.*)$/) {
            $self->set($1, $2);
        } else {
            carp "Invalid config line: $line" if $self->{_strict};
        }
    }
    
    close $fh;
    return $self;
}

# =====================
# Get/Set
# =====================

sub set {
    my ($self, $key, $value) = @_;
    
    # Validate against schema
    if (my $schema = $self->{_schema}{$key}) {
        if (my $type = $schema->{type}) {
            if ($type eq 'int' && $value !~ /^\d+$/) {
                croak "Config key '$key' must be integer, got: $value";
            }
            if ($type eq 'bool' && $value !~ /^(?:0|1|yes|no|true|false)$/i) {
                croak "Config key '$key' must be boolean, got: $value";
            }
        }
        if (my $re = $schema->{pattern}) {
            croak "Config key '$key' doesn't match pattern" unless $value =~ $re;
        }
    }
    
    $self->{_data}{$key} = $value;
    return $self;
}

sub get {
    my ($self, $key, $default) = @_;
    return $self->{_data}{$key}
        // $self->{_defaults}{$key}
        // $default;
}

sub has { exists $_[0]->{_data}{$_[1]} }
sub delete_key { delete $_[0]->{_data}{$_[1]} }

sub keys_list { sort keys %{$_[0]->{_data}} }

sub as_hash {
    my $self = shift;
    return (%{$self->{_defaults}}, %{$self->{_data}});
}

sub dump {
    my $self = shift;
    print "=== Config ===\n";
    printf "  %s = %s\n", $_, $self->get($_)
        for $self->keys_list;
}

1;
```

---

## Step 137: Test::More — Unit Testing

```perl
#!/usr/bin/perl
# File: t/01-basic.t

use strict;
use warnings;
use Test::More tests => 20;
use lib 'lib';

# =====================
# Basic tests
# =====================

ok(1, "1 is true");
ok("hello", "non-empty string is true");
ok(!0, "0 is false");

is(2 + 2, 4, "2+2=4");
is("hello", "hello", "strings match");
isnt("hello", "world", "different strings");

# =====================
# Numeric tests
# =====================

use MathUtils qw(:basic :number :stats);

is(add(3, 4), 7, "3+4=7");
is(subtract(10, 3), 7, "10-3=7");
is(multiply(3, 4), 12, "3*4=12");
is(divide(10, 5), 2, "10/5=2");

ok(is_prime(7),  "7 is prime");
ok(!is_prime(9), "9 is not prime");

is(factorial(5), 120, "5! = 120");
is(fibonacci(7), 13, "fib(7) = 13");
is(gcd(12, 8), 4, "gcd(12,8)=4");

# =====================
# Stats tests
# =====================

my @data = (2, 4, 4, 4, 5, 5, 7, 9);

is(mean(@data), 5, "mean = 5");
is(median(@data), 4.5, "median = 4.5");

# approximate comparison
my $stddev = stddev(@data);
ok(abs($stddev - 2.0) < 0.1, "stddev ≈ 2.0") or 
    diag "got stddev = $stddev";

# =====================
# Error handling
# =====================

eval { divide(10, 0) };
like($@, qr/zero/i, "divide by zero throws error");

# Test passes

done_testing() unless defined &main::tests_planned;
```

---

## Step 138: Moose — Modern OO

```perl
#!/usr/bin/perl
use strict;
use warnings;

# หมายเหตุ: ต้องติดตั้ง Moose ก่อน
# cpan Moose
# หรือ apt install libmoose-perl

use Moose;

# =====================
# Moose class
# =====================

package Person;
use Moose;

has 'name' => (
    is       => 'rw',        # read-write
    isa      => 'Str',        # type: String
    required => 1,
);

has 'age' => (
    is       => 'rw',
    isa      => 'Int',
    default  => 0,
);

has 'email' => (
    is        => 'rw',
    isa       => 'Str',
    predicate => 'has_email',   # adds has_email method
    clearer   => 'clear_email', # adds clear_email method
);

has 'hobbies' => (
    is      => 'rw',
    isa     => 'ArrayRef[Str]',
    default => sub { [] },
    traits  => ['Array'],
    handles => {
        add_hobby  => 'push',
        all_hobbies => 'elements',
        hobby_count => 'count',
    },
);

sub greet {
    my $self = shift;
    printf "Hello, I'm %s, age %d\n", $self->name, $self->age;
}

# Method modifier
before 'greet' => sub {
    print "[before greet]\n";
};

after 'greet' => sub {
    print "[after greet]\n";
};

no Moose;
__PACKAGE__->meta->make_immutable;  # performance optimization

# =====================
# Employee extends Person
# =====================

package Employee;
use Moose;
extends 'Person';

has 'company' => (is => 'rw', isa => 'Str', required => 1);
has 'salary'  => (is => 'rw', isa => 'Num', default  => 0);
has 'title'   => (is => 'rw', isa => 'Str', default  => 'Staff');

override 'greet' => sub {
    my $self = super;
    printf "  I work at %s as %s\n", $self->company, $self->title;
};

no Moose;
__PACKAGE__->meta->make_immutable;

# =====================
# Role (Mixin)
# =====================

package Printable;
use Moose::Role;

requires 'to_string';

sub print_self {
    my $self = shift;
    print $self->to_string, "\n";
}

package Product;
use Moose;
with 'Printable';

has 'name'  => (is => 'rw', isa => 'Str', required => 1);
has 'price' => (is => 'rw', isa => 'Num', required => 1);

sub to_string {
    my $self = shift;
    return sprintf "%s: \$%.2f", $self->name, $self->price;
}

no Moose;
__PACKAGE__->meta->make_immutable;

# =====================
# Test
# =====================

package main;

my $person = Person->new(name => "Alice", age => 28, email => "alice\@example.com");
$person->greet;
print "Has email: ", ($person->has_email ? "yes" : "no"), "\n";

$person->add_hobby("reading");
$person->add_hobby("coding");
$person->add_hobby("hiking");
printf "Hobbies (%d): %s\n", $person->hobby_count, 
    join(", ", $person->all_hobbies);

my $emp = Employee->new(
    name    => "Bob",
    age     => 35,
    company => "TechCorp",
    title   => "Senior Engineer",
    salary  => 80000,
);
$emp->greet;

my $prod = Product->new(name => "Perl Book", price => 49.99);
$prod->print_self;
```

---

## Step 139: Common CPAN Modules

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# List::Util — functional tools
# =====================

use List::Util qw(
    sum sum0 product
    min max minstr maxstr
    first any all none
    reduce uniq uniq_by
    pairs unpairs
    shuffle
    mesh zip
    max_by min_by
);

my @nums = (3, 1, 4, 1, 5, 9, 2, 6);

printf "sum=%d, product=%d, min=%d, max=%d\n",
    sum(@nums), product(@nums), min(@nums), max(@nums);

my $first_gt5 = first { $_ > 5 } @nums;
printf "first > 5: %d\n", $first_gt5;

print "any > 10: ",  (any  { $_ > 10 } @nums) ? "yes" : "no", "\n";
print "all > 0: ",   (all  { $_ > 0  } @nums) ? "yes" : "no", "\n";
print "none < 0: ",  (none { $_ < 0  } @nums) ? "yes" : "no", "\n";

my @unique = uniq @nums;
printf "unique: @unique\n";

# =====================
# Scalar::Util — type checking
# =====================

use Scalar::Util qw(
    blessed reftype looks_like_number
    weaken isweak
    readonly tainted
    openhandle
);

my $obj    = bless {}, "MyClass";
my $ref    = \42;
my $array  = [1,2,3];
my $num    = 42;
my $str    = "hello";

printf "blessed:  %s\n", blessed($obj) // "undef";
printf "reftype:  %s\n", reftype($ref) // "not a ref";
printf "reftype:  %s\n", reftype($array) // "not a ref";
printf "is num:   %s\n", looks_like_number($num) ? "yes" : "no";
printf "is num:   %s\n", looks_like_number($str) ? "yes" : "no";
printf "is num:   %s\n", looks_like_number("3.14") ? "yes" : "no";

# =====================
# POSIX — system functions
# =====================

use POSIX qw(
    floor ceil fmod pow
    strftime mktime
    INT_MAX INT_MIN
    DBL_MAX
    HUGE_VAL
);

printf "floor(3.7) = %d\n", floor(3.7);
printf "ceil(3.2)  = %d\n", ceil(3.2);
printf "fmod(10,3) = %.1f\n", fmod(10, 3);

printf "strftime: %s\n", strftime("%Y-%m-%d %H:%M:%S", localtime);
printf "INT_MAX: %d\n", INT_MAX;

# =====================
# Data::Dumper — debugging
# =====================

use Data::Dumper;

my %complex = (
    name   => "test",
    nested => { a => 1, b => [2, 3, 4] },
    array  => [1..5],
    sub    => sub { 42 },
);

$Data::Dumper::Sortkeys = 1;
$Data::Dumper::Indent   = 1;
print Dumper(\%complex);

# =====================
# Storable — deep copy
# =====================

use Storable qw(dclone);

my @original = ([1,2,3], [4,5,6]);
my @copy = @{dclone(\@original)};

$copy[0][0] = 99;
print "original: $original[0][0]\n";  # 1 (unchanged)
print "copy:     $copy[0][0]\n";      # 99
```

---

## Step 140: โปรแกรมสรุป — Plugin System

```perl
#!/usr/bin/perl
#
# plugin_system.pl — ระบบ plugin แบบง่าย
#

use strict;
use warnings;
use File::Find;

# =====================
# Plugin base class
# =====================

package Plugin::Base;

sub new {
    my ($class, %opts) = @_;
    return bless {
        name        => $opts{name}    // ref $class,
        version     => $opts{version} // "1.0",
        description => $opts{description} // "",
        enabled     => 1,
    }, $class;
}

sub name        { $_[0]->{name} }
sub version     { $_[0]->{version} }
sub description { $_[0]->{description} }
sub enabled     { $_[0]->{enabled} }

sub enable  { $_[0]->{enabled} = 1 }
sub disable { $_[0]->{enabled} = 0 }

sub execute {
    my ($self, $context) = @_;
    die ref($self) . " must implement execute()\n";
}

sub info {
    my $self = shift;
    return sprintf "[%s] v%s: %s", 
        $self->name, $self->version, $self->description;
}

# =====================
# Concrete plugins
# =====================

package Plugin::Logger;
our @ISA = ('Plugin::Base');

sub new {
    my $class = shift;
    return $class->SUPER::new(
        name        => "Logger",
        version     => "1.0",
        description => "Logs all operations",
        @_
    );
}

sub execute {
    my ($self, $ctx) = @_;
    return unless $self->enabled;
    
    my $ts = localtime;
    printf "[%s] Plugin::Logger: event=%s data=%s\n",
        $ts, $ctx->{event} // "unknown",
        $ctx->{data} // "";
}

package Plugin::Validator;
our @ISA = ('Plugin::Base');

sub new {
    my $class = shift;
    return $class->SUPER::new(
        name        => "Validator",
        version     => "2.1",
        description => "Validates input data",
        @_
    );
}

sub execute {
    my ($self, $ctx) = @_;
    return (1, "ok") unless $self->enabled;
    
    my $data = $ctx->{data} // "";
    
    if (length $data == 0) {
        return (0, "Empty data not allowed");
    }
    if (length $data > 1000) {
        return (0, "Data too long (max 1000 chars)");
    }
    if ($data =~ /<script/i) {
        return (0, "XSS attempt detected");
    }
    
    return (1, "valid");
}

package Plugin::Transform;
our @ISA = ('Plugin::Base');

sub new {
    my ($class, %opts) = @_;
    my $self = $class->SUPER::new(
        name        => "Transform",
        version     => "1.5",
        description => "Transforms data",
        %opts,
    );
    $self->{transforms} = $opts{transforms} // [];
    return $self;
}

sub add_transform {
    my ($self, $name, $fn) = @_;
    push @{$self->{transforms}}, { name => $name, fn => $fn };
}

sub execute {
    my ($self, $ctx) = @_;
    return $ctx->{data} unless $self->enabled;
    
    my $data = $ctx->{data} // "";
    for my $t (@{$self->{transforms}}) {
        $data = $t->{fn}->($data);
        printf "  [Transform] Applied: %s\n", $t->{name};
    }
    return $data;
}

# =====================
# Plugin Manager
# =====================

package PluginManager;

sub new {
    my $class = shift;
    return bless { plugins => {} }, $class;
}

sub register {
    my ($self, $plugin) = @_;
    my $name = $plugin->name;
    $self->{plugins}{$name} = $plugin;
    printf "Registered plugin: %s\n", $plugin->info;
}

sub unregister {
    my ($self, $name) = @_;
    delete $self->{plugins}{$name};
}

sub get    { $_[0]->{plugins}{$_[1]} }
sub all    { values %{$_[0]->{plugins}} }
sub names  { sort keys %{$_[0]->{plugins}} }

sub run_all {
    my ($self, $ctx) = @_;
    my @results;
    for my $name ($self->names) {
        my $plugin = $self->{plugins}{$name};
        next unless $plugin->enabled;
        my @result = $plugin->execute($ctx);
        push @results, { plugin => $name, result => \@result };
    }
    return @results;
}

sub list {
    my $self = shift;
    print "\n=== Registered Plugins ===\n";
    for my $name ($self->names) {
        my $p = $self->{plugins}{$name};
        printf "  %-15s v%-5s %s  [%s]\n",
            $p->name, $p->version, $p->description,
            $p->enabled ? "ENABLED" : "DISABLED";
    }
}

# =====================
# Main demo
# =====================

package main;

my $pm = PluginManager->new;

# Create and register plugins
my $logger = Plugin::Logger->new;
my $validator = Plugin::Validator->new;
my $transformer = Plugin::Transform->new;

$transformer->add_transform("trim",       sub { my $s = shift; $s =~ s/^\s+|\s+$//g; $s });
$transformer->add_transform("lowercase",  sub { lc $_[0] });
$transformer->add_transform("normalize",  sub { my $s = shift; $s =~ s/\s+/ /g; $s });

$pm->register($logger);
$pm->register($validator);
$pm->register($transformer);

$pm->list;

# Run with valid data
print "\n--- Test 1: Valid data ---\n";
my $ctx = { event => "user_input", data => "  Hello   WORLD  " };
my @results = $pm->run_all($ctx);

# Run with invalid data
print "\n--- Test 2: XSS attempt ---\n";
$ctx = { event => "form_submit", data => "<script>alert(1)</script>" };
@results = $pm->run_all($ctx);

for my $r (@results) {
    my @res = @{$r->{result}};
    if (@res == 2) {
        printf "  %s: [%s] %s\n", $r->{plugin}, $res[0] ? "OK" : "FAIL", $res[1];
    }
}

# Disable a plugin
print "\n--- Disabling Logger ---\n";
$pm->get("Logger")->disable;
$pm->list;
```

---

## สรุป Part 14

ใน Part นี้คุณได้เรียนรู้:
- ✅ Package และ namespace
- ✅ สร้าง .pm module file
- ✅ Exporter สำหรับ export functions
- ✅ OOP ด้วย bless
- ✅ Inheritance ด้วย @ISA
- ✅ AUTOLOAD และ DESTROY
- ✅ Operator overloading
- ✅ Moose (modern OO)
- ✅ CPAN modules สำคัญ
- ✅ Plugin pattern

**ถัดไป: [Part 15 — Error Handling และ Debugging](part_15.md)**
