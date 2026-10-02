# Part 22: Advanced OOP
## Steps 211-220: Object-Oriented Perl ขั้นสูง

---

## Step 211: OOP พื้นฐานทบทวน

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Class definition recap
# =====================

package Animal;

sub new {
    my ($class, %args) = @_;
    my $self = {
        name    => $args{name}   // "Unknown",
        species => $args{species} // "Unknown",
        age     => $args{age}    // 0,
        _health => 100,
    };
    return bless $self, $class;
}

# Accessors
sub name    { my ($self,$v)=@_; $self->{name}=$v if @_>1; return $self->{name} }
sub species { my ($self,$v)=@_; $self->{species}=$v if @_>1; return $self->{species} }
sub age     { my ($self,$v)=@_; $self->{age}=$v if @_>1; return $self->{age} }
sub health  { $_[0]->{_health} }

sub speak   { "..." }

sub describe {
    my $self = shift;
    return sprintf("%s the %s, age %d (health: %d%%)",
        $self->name, $self->species, $self->age, $self->health);
}

sub eat {
    my ($self, $food) = @_;
    $self->{_health} = min(100, $self->{_health} + 5);
    printf "%s eats %s\n", $self->name, $food;
}

sub min { $_[0] < $_[1] ? $_[0] : $_[1] }

sub DESTROY {
    my $self = shift;
    # printf "DEBUG: %s destroyed\n", $self->{name} // "unnamed";
}

# =====================
# Inheritance
# =====================

package Dog;
our @ISA = ('Animal');

sub new {
    my ($class, %args) = @_;
    my $self = Animal::new($class, species => "Dog", %args);
    $self->{breed} = $args{breed} // "Mixed";
    $self->{tricks} = [];
    return $self;
}

sub breed  { my ($self,$v)=@_; $self->{breed}=$v if @_>1; return $self->{breed} }
sub speak  { "Woof!" }

sub learn_trick {
    my ($self, $trick) = @_;
    push @{$self->{tricks}}, $trick;
    printf "%s learned: %s\n", $self->name, $trick;
}

sub show_tricks {
    my $self = shift;
    if (@{$self->{tricks}}) {
        printf "%s can: %s\n", $self->name, join(", ", @{$self->{tricks}});
    } else {
        printf "%s has no tricks\n", $self->name;
    }
}

sub describe {
    my $self = shift;
    return $self->SUPER::describe() . " (breed: " . $self->breed . ")";
}

package Cat;
our @ISA = ('Animal');

sub new {
    my ($class, %args) = @_;
    my $self = Animal::new($class, species => "Cat", %args);
    $self->{indoor} = $args{indoor} // 1;
    return $self;
}

sub speak     { "Meow~" }
sub is_indoor { $_[0]->{indoor} }

package main;

my $dog = Dog->new(name => "Rex", breed => "German Shepherd", age => 3);
my $cat = Cat->new(name => "Whiskers", age => 5);

printf "Dog: %s\n", $dog->describe;
printf "Cat: %s\n", $cat->describe;

$dog->speak;   # Actually returns value
printf "%s says: %s\n", $dog->name, $dog->speak;
printf "%s says: %s\n", $cat->name, $cat->speak;

$dog->learn_trick("sit");
$dog->learn_trick("shake");
$dog->learn_trick("roll over");
$dog->show_tricks;

# Polymorphism
my @animals = ($dog, $cat, Dog->new(name => "Buddy", breed => "Lab", age => 2));
printf "\nAll animals say:\n";
printf "  %s: %s\n", $_->name, $_->speak for @animals;

# Type checking
for my $a (@animals) {
    printf "%s isa Animal: %s\n", $a->name, $a->isa('Animal') ? 'yes' : 'no';
    printf "%s isa Dog:    %s\n", $a->name, $a->isa('Dog')    ? 'yes' : 'no';
    printf "%s ref:        %s\n", $a->name, ref($a);
}
```

---

## Step 212: Accessor Generator

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Auto-generate accessors
# =====================

package AccessorBase;

# Generate read-write accessors
sub mk_accessors {
    my ($class, @fields) = @_;
    no strict 'refs';
    for my $field (@fields) {
        *{"${class}::${field}"} = sub {
            my ($self, $val) = @_;
            $self->{$field} = $val if @_ > 1;
            return $self->{$field};
        };
    }
}

# Generate read-only accessors
sub mk_ro_accessors {
    my ($class, @fields) = @_;
    no strict 'refs';
    for my $field (@fields) {
        *{"${class}::${field}"} = sub {
            my $self = shift;
            die "Cannot set read-only field '$field'\n" if @_;
            return $self->{$field};
        };
    }
}

# Generate lazy accessors (computed once)
sub mk_lazy_accessor {
    my ($class, $field, $builder) = @_;
    no strict 'refs';
    *{"${class}::${field}"} = sub {
        my $self = shift;
        unless (exists $self->{"_${field}"}) {
            $self->{"_${field}"} = $builder->($self);
        }
        return $self->{"_${field}"};
    };
}

# new with validation
sub new {
    my ($class, %args) = @_;
    my $self = bless {}, $class;
    
    for my $field (keys %args) {
        $self->$field($args{$field}) if $self->can($field);
    }
    
    $self->_validate if $self->can('_validate');
    return $self;
}

# =====================
# User class
# =====================

package User;
our @ISA = ('AccessorBase');

User->mk_accessors(qw(username email age role));
User->mk_ro_accessors(qw(id created_at));

sub new {
    my ($class, %args) = @_;
    
    die "username required\n" unless $args{username};
    die "email required\n"    unless $args{email};
    
    my $self = bless {
        id         => int(rand 99999),
        created_at => time(),
    }, $class;
    
    $self->$_($args{$_}) for qw(username email age role);
    $self->{role} //= 'user';
    
    return $self;
}

sub _validate {
    my $self = shift;
    die "Invalid email\n" unless $self->email =~ /\@/;
    die "Age must be positive\n" if defined $self->age && $self->age < 0;
}

User->mk_lazy_accessor('display_name', sub {
    my $self = shift;
    return sprintf("%s <%s>", $self->username, $self->email);
});

package Product;
our @ISA = ('AccessorBase');

Product->mk_accessors(qw(name price stock category description));
Product->mk_ro_accessors(qw(sku created_at));

sub new {
    my ($class, %args) = @_;
    
    die "name required\n"  unless $args{name};
    die "price required\n" unless defined $args{price};
    
    my $self = bless {
        sku        => "SKU" . sprintf("%06d", int(rand 999999)),
        created_at => time(),
    }, $class;
    
    $self->$_($args{$_}) for qw(name price stock category description);
    $self->{stock} //= 0;
    
    return $self;
}

sub in_stock  { $_[0]->stock > 0 }
sub available { my ($self, $qty) = @_; $self->stock >= ($qty//1) }

package main;

my $user = User->new(
    username => "alice",
    email    => "alice\@example.com",
    age      => 28,
    role     => "admin",
);

printf "User: %s (role: %s)\n", $user->username, $user->role;
printf "ID: %d (readonly)\n", $user->id;
printf "Display: %s\n", $user->display_name;   # lazy

$user->age(29);
printf "Updated age: %d\n", $user->age;

eval { $user->id(999) };    # Should fail
print "Read-only error: $@" if $@;

my $product = Product->new(name => "Widget", price => 9.99, stock => 50);
printf "\nProduct: %s (%s) \$%.2f\n", $product->name, $product->sku, $product->price;
printf "In stock: %s\n", $product->in_stock ? "yes" : "no";
printf "Can fulfill 30: %s\n", $product->available(30) ? "yes" : "no";

eval { User->new(username => "bob", email => "not-an-email") };
print "Validation error: $@" if $@;
```

---

## Step 213: Role/Mixin Pattern

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Role / Mixin system
# =====================

package Role;

sub import_into {
    my ($role, $target_class) = @_;
    no strict 'refs';
    
    for my $method (keys %{"${role}::"}) {
        next if $method =~ /^(import_into|requires|BEGIN|END|ISA)$/;
        next unless defined &{"${role}::${method}"};
        
        *{"${target_class}::${method}"} = \&{"${role}::${method}"};
    }
    
    # Check requirements
    if ($role->can('requires')) {
        for my $req ($role->requires) {
            die "Class $target_class must implement '$req' (required by role $role)\n"
                unless $target_class->can($req);
        }
    }
}

# =====================
# Roles
# =====================

package Role::Printable;

sub to_string { die ref(shift) . " must implement to_string\n" }

sub print_info {
    my $self = shift;
    printf "[%s] %s\n", ref($self), $self->to_string;
}

sub print_verbose {
    my $self = shift;
    my %data = $self->to_hash if $self->can('to_hash');
    printf "[%s]\n", ref($self);
    printf "  %s: %s\n", $_, $data{$_} for sort keys %data;
}

package Role::Serializable;

sub to_json {
    my $self = shift;
    require JSON::PP;
    return JSON::PP->new->utf8->canonical->encode($self->to_hash);
}

sub from_json {
    my ($class, $json) = @_;
    require JSON::PP;
    my $data = JSON::PP->new->utf8->decode($json);
    return $class->new(%$data);
}

package Role::Comparable;

sub equals {
    my ($self, $other) = @_;
    return 0 unless ref($other) eq ref($self);
    my %a = $self->to_hash;
    my %b = $other->to_hash;
    for my $k (keys %a) {
        return 0 unless defined $b{$k} && $a{$k} eq $b{$k};
    }
    return 1;
}

package Role::Timestamped;

sub created_at { $_[0]->{created_at} //= time() }
sub updated_at { $_[0]->{updated_at} }

sub touch {
    my $self = shift;
    $self->{updated_at} = time();
}

sub age_seconds {
    my $self = shift;
    return time() - ($self->{created_at} // time());
}

# =====================
# Apply roles to classes
# =====================

package Person;

sub new {
    my ($class, %args) = @_;
    return bless {
        name       => $args{name},
        email      => $args{email},
        age        => $args{age} // 0,
        created_at => time(),
    }, $class;
}

sub name  { $_[0]->{name}  }
sub email { $_[0]->{email} }
sub age   { $_[0]->{age}   }

sub to_string { sprintf("%s <%s> age %d", $_[0]{name}, $_[0]{email}, $_[0]{age}) }

sub to_hash {
    my $self = shift;
    return (name => $self->{name}, email => $self->{email}, age => $self->{age});
}

# Apply roles
Role::Printable->import_into('Person');
Role::Serializable->import_into('Person');
Role::Comparable->import_into('Person');
Role::Timestamped->import_into('Person');

package main;

my $alice = Person->new(name => "Alice", email => "alice\@example.com", age => 28);
my $bob   = Person->new(name => "Bob",   email => "bob\@example.com",   age => 35);
my $alice2 = Person->new(name => "Alice", email => "alice\@example.com", age => 28);

$alice->print_info;
$bob->print_info;

my $json = $alice->to_json;
printf "JSON: %s\n", $json;

my $alice3 = Person->from_json($json);
printf "From JSON: %s\n", $alice3->to_string;

printf "alice == alice2: %s\n", $alice->equals($alice2) ? "yes" : "no";
printf "alice == bob:    %s\n", $alice->equals($bob)    ? "yes" : "no";

$alice->touch;
printf "alice age (seconds): %d\n", $alice->age_seconds;
```

---

## Step 214: Overloading

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Scalar::Util qw(blessed);

# =====================
# Operator Overloading
# =====================

package Vec2D;
use overload
    '+'  => \&add,
    '-'  => \&subtract,
    '*'  => \&multiply,
    '/'  => \&divide,
    '==' => \&equal,
    '!=' => \&not_equal,
    '""' => \&to_string,
    'abs' => \&length,
    'neg' => \&negate,
    'fallback' => 1;

sub new {
    my ($class, $x, $y) = @_;
    return bless { x => $x // 0, y => $y // 0 }, $class;
}

sub x { $_[0]->{x} }
sub y { $_[0]->{y} }

sub add {
    my ($a, $b) = @_;
    if (blessed $b) {
        return Vec2D->new($a->x + $b->x, $a->y + $b->y);
    }
    return Vec2D->new($a->x + $b, $a->y + $b);
}

sub subtract {
    my ($a, $b, $swap) = @_;
    if ($swap) { return Vec2D->new($b - $a->x, $b - $a->y) }
    if (blessed $b) { return Vec2D->new($a->x - $b->x, $a->y - $b->y) }
    return Vec2D->new($a->x - $b, $a->y - $b);
}

sub multiply {
    my ($a, $b) = @_;
    if (blessed $b) {
        return $a->x * $b->x + $a->y * $b->y;   # dot product
    }
    return Vec2D->new($a->x * $b, $a->y * $b);
}

sub divide {
    my ($a, $b, $swap) = @_;
    die "Cannot divide by zero\n" if $b == 0;
    return Vec2D->new($a->x / $b, $a->y / $b);
}

sub equal    { my ($a,$b)=@_; blessed $b && $a->x==$b->x && $a->y==$b->y }
sub not_equal { !equal(@_) }
sub negate   { Vec2D->new(-$_[0]->x, -$_[0]->y) }
sub length   { sqrt($_[0]->x**2 + $_[0]->y**2) }

sub normalize {
    my $self = shift;
    my $len = abs($self);
    return Vec2D->new(0,0) if $len == 0;
    return $self / $len;
}

sub dot      { my ($a,$b)=@_; $a->x*$b->x + $a->y*$b->y }
sub cross    { my ($a,$b)=@_; $a->x*$b->y - $a->y*$b->x }
sub distance { abs($_[0] - $_[1]) }
sub angle    { atan2($_[0]->y, $_[0]->x) }

sub to_string {
    my $self = shift;
    return sprintf("(%.3f, %.3f)", $self->x, $self->y);
}

# =====================
# Matrix2x2 with overloading
# =====================

package Matrix2x2;
use overload
    '*'  => \&multiply,
    '""' => \&to_string,
    'fallback' => 1;

sub new {
    my ($class, $a, $b, $c, $d) = @_;
    return bless { a=>$a, b=>$b, c=>$c, d=>$d }, $class;
}

sub identity { Matrix2x2->new(1,0,0,1) }
sub rotation {
    my ($class, $angle) = @_;
    my ($cos, $sin) = (cos($angle), sin($angle));
    return Matrix2x2->new($cos, -$sin, $sin, $cos);
}

sub multiply {
    my ($m, $v) = @_;
    if (blessed $v && $v->isa('Vec2D')) {
        return Vec2D->new(
            $m->{a}*$v->x + $m->{b}*$v->y,
            $m->{c}*$v->x + $m->{d}*$v->y,
        );
    }
    if (blessed $v && $v->isa('Matrix2x2')) {
        return Matrix2x2->new(
            $m->{a}*$v->{a}+$m->{b}*$v->{c}, $m->{a}*$v->{b}+$m->{b}*$v->{d},
            $m->{c}*$v->{a}+$m->{d}*$v->{c}, $m->{c}*$v->{b}+$m->{d}*$v->{d},
        );
    }
    return Matrix2x2->new($m->{a}*$v, $m->{b}*$v, $m->{c}*$v, $m->{d}*$v);
}

sub det { $_[0]->{a}*$_[0]->{d} - $_[0]->{b}*$_[0]->{c} }

sub to_string {
    my $m = shift;
    return sprintf("[%.2f %.2f; %.2f %.2f]", $m->{a},$m->{b},$m->{c},$m->{d});
}

package main;

my $v1 = Vec2D->new(3, 4);
my $v2 = Vec2D->new(1, 2);

printf "v1 = %s\n",    $v1;
printf "v2 = %s\n",    $v2;
printf "v1+v2 = %s\n", $v1 + $v2;
printf "v1-v2 = %s\n", $v1 - $v2;
printf "v1*3 = %s\n",  $v1 * 3;
printf "|v1| = %.3f\n", abs($v1);
printf "-v1 = %s\n",   -$v1;
printf "v1==v1: %s\n", ($v1 == $v1) ? "yes" : "no";
printf "v1==v2: %s\n", ($v1 == $v2) ? "yes" : "no";
printf "norm(v1) = %s\n", $v1->normalize;
printf "dot(v1,v2) = %.1f\n", $v1->dot($v2);

my $pi = 3.14159265;
my $rot = Matrix2x2->rotation($pi / 2);   # 90°
printf "\nRotation matrix: %s\n", $rot;
printf "v1 rotated 90°: %s\n", $rot * $v1;
```

---

## Step 215: Metaprogramming

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Metaprogramming — code that generates code
# =====================

# AUTOLOAD — catch undefined method calls
package AutoAccessor;

sub new {
    my ($class, %data) = @_;
    return bless \%data, $class;
}

our $AUTOLOAD;
sub AUTOLOAD {
    my ($self, $val) = @_;
    my $name = $AUTOLOAD;
    $name =~ s/.*:://;
    
    return if $name eq 'DESTROY';
    
    # Generate the accessor and install it
    no strict 'refs';
    *{$AUTOLOAD} = sub {
        my ($self, $v) = @_;
        $self->{$name} = $v if @_ > 1;
        return $self->{$name};
    };
    
    # Call it
    $self->{$name} = $val if @_ > 1;
    return $self->{$name};
}

# =====================
# Code generation via strings (eval)
# =====================

package ClassBuilder;

sub build_class {
    my ($class_name, %spec) = @_;
    
    my @code;
    push @code, "package $class_name;";
    push @code, "our \@ISA = ('" . join("','", @{$spec{extends}}) . "');" if $spec{extends};
    
    # Constructor
    push @code, <<CODE;
sub new {
    my (\$class, \%args) = \@_;
    return bless \\\%args, \$class;
}
CODE
    
    # Accessors
    for my $field (@{$spec{fields} // []}) {
        push @code, <<CODE;
sub $field {
    my (\$self, \$v) = \@_;
    \$self->{$field} = \$v if \@_ > 1;
    return \$self->{$field};
}
CODE
    }
    
    # Methods
    for my $name (keys %{$spec{methods} // {}}) {
        my $code = $spec{methods}{$name};
        push @code, "sub $name { $code }";
    }
    
    my $full_code = join "\n", @code;
    eval $full_code;
    die "ClassBuilder error: $@\nCode: $full_code\n" if $@;
    
    return $class_name;
}

# =====================
# Method decorators
# =====================

package Decorator;

sub logged {
    my ($class, $method_name, $method) = @_;
    return sub {
        my $self = shift;
        printf "[LOG] Calling %s::%s\n", ref($self)||$self, $method_name;
        my @result = $method->($self, @_);
        printf "[LOG] %s::%s returned: %s\n", ref($self)||$self, $method_name,
            join(", ", @result);
        return wantarray ? @result : $result[0];
    };
}

sub timed {
    my ($class, $method_name, $method) = @_;
    return sub {
        my $self = shift;
        my $start = time();
        my @result = $method->($self, @_);
        printf "[TIMER] %s took %.4fs\n", $method_name, time() - $start;
        return wantarray ? @result : $result[0];
    };
}

sub cached {
    my ($class, $method_name, $method) = @_;
    my %cache;
    return sub {
        my $self = shift;
        my $key  = join("\0", @_);
        unless (exists $cache{$key}) {
            $cache{$key} = [$method->($self, @_)];
        }
        return wantarray ? @{$cache{$key}} : $cache{$key}[0];
    };
}

# Install decorator into class
sub decorate {
    my ($class, $target_class, $method_name, @decorators) = @_;
    no strict 'refs';
    my $method = \&{"${target_class}::${method_name}"};
    for my $dec (@decorators) {
        $method = $dec->($class, $method_name, $method);
    }
    *{"${target_class}::${method_name}"} = $method;
}

# =====================
# Demo
# =====================

package main;

# AutoAccessor demo
my $obj = AutoAccessor->new(name => "Alice", age => 28);
printf "Name: %s\n", $obj->name;
printf "Age:  %d\n", $obj->age;
$obj->email("alice\@example.com");   # dynamic field
printf "Email: %s\n", $obj->email;

# ClassBuilder demo
ClassBuilder::build_class('Animal',
    fields  => [qw(name age species)],
    methods => {
        speak    => '"Generic animal sound"',
        describe => 'sprintf("I am %s the %s, age %d", $_[0]->name, $_[0]->species, $_[0]->age)',
    }
);

ClassBuilder::build_class('Dog',
    extends => ['Animal'],
    fields  => [qw(breed)],
    methods => {
        speak    => '"Woof!"',
        describe => '$_[0]->SUPER::describe() . " (breed: " . $_[0]->breed . ")"',
    }
);

my $dog = Dog->new(name => "Rex", age => 3, species => "Dog", breed => "Lab");
printf "\n%s says: %s\n", $dog->name, $dog->speak;
printf "%s\n", $dog->describe;
printf "isa Animal: %s\n", $dog->isa('Animal') ? "yes" : "no";

# Decorator demo
package Calculator;
sub new { bless {}, shift }
sub add { $_[1] + $_[2] }
sub expensive { my $sum=0; $sum+=$_ for 1..1000; return $sum }

package main;

Decorator->decorate('Calculator', 'add', Decorator->can('logged'));
Decorator->decorate('Calculator', 'expensive', Decorator->can('timed'), Decorator->can('cached'));

my $calc = Calculator->new;
printf "\n%d\n", $calc->add(3, 4);
printf "%d\n", $calc->expensive;
printf "%d (cached)\n", $calc->expensive;
```

---

## Step 216: Design Patterns

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Singleton
# =====================

package Singleton::Config;

my $instance;

sub instance {
    my $class = shift;
    unless ($instance) {
        $instance = bless {
            settings => {
                debug   => 0,
                timeout => 30,
                version => "1.0.0",
            }
        }, $class;
    }
    return $instance;
}

sub get { $_[0]->{settings}{$_[1]} }
sub set { $_[0]->{settings}{$_[1]} = $_[2] }
sub all { %{$_[0]->{settings}} }

# Prevent cloning
sub new   { die "Use instance()\n" }
sub clone { die "Cannot clone singleton\n" }

# =====================
# Factory
# =====================

package Shape::Factory;

sub create {
    my ($class, $type, %args) = @_;
    
    my %registry = (
        circle    => 'Shape::Circle',
        rectangle => 'Shape::Rectangle',
        triangle  => 'Shape::Triangle',
    );
    
    my $shape_class = $registry{lc $type}
        or die "Unknown shape: $type\n";
    
    return $shape_class->new(%args);
}

package Shape;
sub new { my ($c,%a)=@_; bless \%a, $c }
sub area      { die ref(shift) . "::area not implemented\n" }
sub perimeter { die ref(shift) . "::perimeter not implemented\n" }
sub describe  {
    my $self = shift;
    sprintf("%s: area=%.2f perimeter=%.2f", ref($self), $self->area, $self->perimeter);
}

package Shape::Circle;
our @ISA = ('Shape');
use POSIX qw();
sub area      { 3.14159265 * $_[0]->{radius} ** 2 }
sub perimeter { 2 * 3.14159265 * $_[0]->{radius} }

package Shape::Rectangle;
our @ISA = ('Shape');
sub area      { $_[0]->{width} * $_[0]->{height} }
sub perimeter { 2 * ($_[0]->{width} + $_[0]->{height}) }

package Shape::Triangle;
our @ISA = ('Shape');
sub area {
    my ($a,$b,$c) = ($_[0]->{a},$_[0]->{b},$_[0]->{c});
    my $s = ($a+$b+$c)/2;
    return sqrt($s*($s-$a)*($s-$b)*($s-$c));
}
sub perimeter { $_[0]->{a}+$_[0]->{b}+$_[0]->{c} }

# =====================
# Observer
# =====================

package EventEmitter;

sub new { bless { _events => {} }, shift }

sub on {
    my ($self, $event, $listener) = @_;
    push @{$self->{_events}{$event}}, $listener;
    return $self;
}

sub off {
    my ($self, $event, $listener) = @_;
    $self->{_events}{$event} = [
        grep { $_ != $listener } @{$self->{_events}{$event} // []}
    ];
}

sub emit {
    my ($self, $event, @args) = @_;
    $_->(@args) for @{$self->{_events}{$event} // []};
    return $self;
}

sub once {
    my ($self, $event, $listener) = @_;
    my $wrapper;
    $wrapper = sub {
        $listener->(@_);
        $self->off($event, $wrapper);
    };
    $self->on($event, $wrapper);
}

# =====================
# Command Pattern
# =====================

package CommandHistory;

sub new { bless { history => [], redo_stack => [] }, shift }

sub execute {
    my ($self, $cmd) = @_;
    $cmd->execute;
    push @{$self->{history}}, $cmd;
    @{$self->{redo_stack}} = ();   # clear redo on new command
}

sub undo {
    my $self = shift;
    return unless @{$self->{history}};
    my $cmd = pop @{$self->{history}};
    $cmd->undo;
    push @{$self->{redo_stack}}, $cmd;
}

sub redo {
    my $self = shift;
    return unless @{$self->{redo_stack}};
    my $cmd = pop @{$self->{redo_stack}};
    $cmd->execute;
    push @{$self->{history}}, $cmd;
}

package TextEditor;
sub new { bless { text => "" }, shift }
sub text { $_[0]->{text} }

package Command::InsertText;
sub new    { my ($c,%a)=@_; bless \%a, $c }
sub execute { $_[0]->{editor}{text} .= $_[0]->{text}; printf "  Insert: '%s'\n", $_[0]->{text} }
sub undo    { my $len=length $_[0]->{text}; substr($_[0]->{editor}{text},-$len)=""; printf "  Undo insert: '%s'\n", $_[0]->{text} }

package Command::DeleteText;
sub new    { my ($c,%a)=@_; bless {%a,deleted=>""}, $c }
sub execute {
    my $self = shift;
    $self->{deleted} = substr($self->{editor}{text}, -$self->{count});
    substr($self->{editor}{text}, -$self->{count}) = "";
    printf "  Delete %d chars: '%s'\n", $self->{count}, $self->{deleted};
}
sub undo { $_[0]->{editor}{text} .= $_[0]->{deleted}; printf "  Undo delete: '%s'\n", $_[0]->{deleted} }

package main;

# Singleton demo
my $cfg = Singleton::Config->instance;
$cfg->set('debug', 1);
$cfg->set('app_name', 'MyApp');

my $cfg2 = Singleton::Config->instance;   # Same object
printf "Same singleton: %s\n", ($cfg == $cfg2) ? "yes" : "no";
printf "debug: %d\n", $cfg2->get('debug');

# Factory demo
printf "\nShapes:\n";
for my $spec (
    [circle    => radius  => 5],
    [rectangle => width   => 4, height => 6],
    [triangle  => a => 3, b => 4, c => 5],
) {
    my ($type, %args) = @$spec;
    my $shape = Shape::Factory->create($type, %args);
    printf "  %s\n", $shape->describe;
}

# Observer demo
printf "\nObserver:\n";
my $emitter = EventEmitter->new;

$emitter->on('data', sub { printf "  Listener A: %s\n", $_[0] });
$emitter->on('data', sub { printf "  Listener B: %s\n", $_[0] });
$emitter->once('connect', sub { printf "  Connected! (fires once)\n" });

$emitter->emit('connect');
$emitter->emit('connect');   # won't fire again
$emitter->emit('data', 'hello');
$emitter->emit('data', 'world');

# Command pattern
printf "\nCommand/Undo:\n";
my $editor  = TextEditor->new;
my $history = CommandHistory->new;

$history->execute(Command::InsertText->new(editor => $editor, text => "Hello"));
$history->execute(Command::InsertText->new(editor => $editor, text => " World"));
printf "Text: '%s'\n", $editor->text;

$history->undo;
printf "After undo: '%s'\n", $editor->text;

$history->redo;
printf "After redo: '%s'\n", $editor->text;

$history->execute(Command::DeleteText->new(editor => $editor, count => 5));
printf "After delete: '%s'\n", $editor->text;

$history->undo;
printf "After undo delete: '%s'\n", $editor->text;
```

---

## Step 217: Exception Hierarchy

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Scalar::Util qw(blessed);

# =====================
# Exception class hierarchy
# =====================

package Exception;
use overload '""' => \&to_string, fallback => 1;

sub new {
    my ($class, %args) = @_;
    my @stack;
    my $i = 0;
    while (my @caller = caller($i++)) {
        push @stack, sprintf "  at %s line %d", $caller[1], $caller[2];
        last if $i > 10;
    }
    return bless {
        message    => $args{message} // "An error occurred",
        code       => $args{code}    // 0,
        stack      => \@stack,
        timestamp  => time(),
    }, $class;
}

sub message  { $_[0]->{message} }
sub code     { $_[0]->{code}    }
sub stack    { @{$_[0]->{stack}} }
sub throw    { die shift->new(@_) }

sub to_string {
    my $self = shift;
    return sprintf("[%s] %s (code: %d)", ref($self), $self->{message}, $self->{code});
}

sub trace {
    my $self = shift;
    return $self->to_string . "\n" . join("\n", $self->stack);
}

package Exception::IO;
our @ISA = ('Exception');
sub new {
    my ($class, %args) = @_;
    $args{code} //= 500;
    return $class->SUPER::new(%args, filename => $args{filename});
}
sub filename { $_[0]->{filename} }

package Exception::FileNotFound;
our @ISA = ('Exception::IO');
sub new { my ($c,%a)=@_; $a{code}//=404; $c->SUPER::new(%a) }

package Exception::PermissionDenied;
our @ISA = ('Exception::IO');
sub new { my ($c,%a)=@_; $a{code}//=403; $c->SUPER::new(%a) }

package Exception::Network;
our @ISA = ('Exception');
sub new {
    my ($class, %args) = @_;
    $args{code} //= 503;
    return $class->SUPER::new(%args, url => $args{url});
}

package Exception::Validation;
our @ISA = ('Exception');
sub new {
    my ($class, %args) = @_;
    $args{code} //= 422;
    my $self = $class->SUPER::new(%args);
    $self->{errors} = $args{errors} // {};
    return $self;
}
sub errors { %{$_[0]->{errors}} }

# =====================
# try/catch pattern
# =====================

sub try_catch {
    my ($try, %handlers) = @_;
    
    eval { $try->() };
    
    if (my $err = $@) {
        my $type = blessed($err) // 'string';
        
        # Find most specific handler
        for my $exc_class (keys %handlers) {
            if (blessed($err) && $err->isa($exc_class)) {
                return $handlers{$exc_class}->($err);
            }
        }
        
        # Default handler
        if ($handlers{default}) {
            return $handlers{default}->($err);
        }
        
        die $err;   # re-throw
    }
}

# =====================
# Usage examples
# =====================

package main;

sub open_file {
    my $path = shift;
    die Exception::FileNotFound->new(
        message  => "File not found: $path",
        filename => $path,
    ) unless -e $path;
    
    open(my $fh, '<', $path) or die Exception::PermissionDenied->new(
        message  => "Cannot open $path: $!",
        filename => $path,
    );
    return $fh;
}

sub validate_user {
    my %data = @_;
    my %errors;
    
    $errors{username} = "required"     unless $data{username};
    $errors{email}    = "invalid"      unless ($data{email}//"") =~ /\@/;
    $errors{age}      = "must be >= 0" if     defined $data{age} && $data{age} < 0;
    
    die Exception::Validation->new(
        message => "Validation failed",
        errors  => \%errors,
    ) if %errors;
}

# Test exceptions
print "=== Exception Handling ===\n\n";

# File not found
try_catch(
    sub { open_file("/nonexistent/file.txt") },
    'Exception::FileNotFound' => sub {
        my $e = shift;
        printf "FileNotFound: %s\n", $e->message;
        printf "  file: %s\n", $e->filename;
    },
    'Exception::IO' => sub {
        my $e = shift;
        printf "IO Error: %s\n", $e->message;
    },
    'default' => sub {
        printf "Unknown error: %s\n", shift;
    }
);

# Validation error
try_catch(
    sub { validate_user(username => "", email => "not-an-email", age => -1) },
    'Exception::Validation' => sub {
        my $e = shift;
        printf "Validation failed:\n";
        my %errs = $e->errors;
        printf "  %s: %s\n", $_, $errs{$_} for sort keys %errs;
    },
);

# Re-throw
try_catch(
    sub {
        try_catch(
            sub { Exception::Network->new(message => "Connection timeout", url => "http://x.com")->throw },
            'Exception::IO' => sub { printf "IO (wrong type)\n" },
        );
    },
    'Exception::Network' => sub {
        my $e = shift;
        printf "Caught at outer level: %s\n", $e->message;
    }
);

# Exception hierarchy check
my $err = Exception::FileNotFound->new(message => "test");
printf "\nException type checks:\n";
printf "  isa Exception:      %s\n", $err->isa('Exception') ? "yes" : "no";
printf "  isa Exception::IO:  %s\n", $err->isa('Exception::IO') ? "yes" : "no";
printf "  isa FileNotFound:   %s\n", $err->isa('Exception::FileNotFound') ? "yes" : "no";
printf "  isa Network:        %s\n", $err->isa('Exception::Network') ? "yes" : "no";
```

---

## Step 218: Builder Pattern

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Builder Pattern
# =====================

package Query::Builder;

sub new {
    my $class = shift;
    return bless {
        _select  => ['*'],
        _from    => undef,
        _joins   => [],
        _where   => [],
        _group   => [],
        _having  => [],
        _order   => [],
        _limit   => undef,
        _offset  => undef,
        _params  => [],
    }, $class;
}

sub select {
    my ($self, @cols) = @_;
    my $clone = $self->_clone;
    $clone->{_select} = \@cols;
    return $clone;
}

sub from {
    my ($self, $table, $alias) = @_;
    my $clone = $self->_clone;
    $clone->{_from} = $alias ? "$table AS $alias" : $table;
    return $clone;
}

sub join_table {
    my ($self, $table, $on, $type) = @_;
    my $clone = $self->_clone;
    $type //= "INNER";
    push @{$clone->{_joins}}, "$type JOIN $table ON $on";
    return $clone;
}

sub left_join  { my ($self,$t,$on)=@_; $self->join_table($t,$on,"LEFT") }
sub right_join { my ($self,$t,$on)=@_; $self->join_table($t,$on,"RIGHT") }

sub where {
    my ($self, $cond, @params) = @_;
    my $clone = $self->_clone;
    push @{$clone->{_where}},  $cond;
    push @{$clone->{_params}}, @params;
    return $clone;
}

sub and_where { shift->where(@_) }

sub or_where {
    my ($self, $cond, @params) = @_;
    my $clone = $self->_clone;
    if (@{$clone->{_where}}) {
        my $last = pop @{$clone->{_where}};
        push @{$clone->{_where}}, "($last OR $cond)";
    } else {
        push @{$clone->{_where}}, $cond;
    }
    push @{$clone->{_params}}, @params;
    return $clone;
}

sub group_by {
    my ($self, @cols) = @_;
    my $clone = $self->_clone;
    push @{$clone->{_group}}, @cols;
    return $clone;
}

sub having {
    my ($self, $cond, @params) = @_;
    my $clone = $self->_clone;
    push @{$clone->{_having}}, $cond;
    push @{$clone->{_params}}, @params;
    return $clone;
}

sub order_by {
    my ($self, $col, $dir) = @_;
    my $clone = $self->_clone;
    push @{$clone->{_order}}, "$col " . uc($dir//"ASC");
    return $clone;
}

sub limit  { my ($self,$n)=@_; my $c=$self->_clone; $c->{_limit}=$n;  $c }
sub offset { my ($self,$n)=@_; my $c=$self->_clone; $c->{_offset}=$n; $c }

sub page {
    my ($self, $page, $per_page) = @_;
    $per_page //= 20;
    return $self->limit($per_page)->offset(($page-1)*$per_page);
}

sub _clone {
    my $self = shift;
    require Storable;
    return Storable::dclone($self);
}

sub build {
    my $self = shift;
    
    die "No table specified\n" unless $self->{_from};
    
    my $sql = "SELECT " . join(", ", @{$self->{_select}});
    $sql .= " FROM " . $self->{_from};
    $sql .= " " . join(" ", @{$self->{_joins}}) if @{$self->{_joins}};
    
    if (@{$self->{_where}}) {
        $sql .= " WHERE " . join(" AND ", @{$self->{_where}});
    }
    
    if (@{$self->{_group}}) {
        $sql .= " GROUP BY " . join(", ", @{$self->{_group}});
    }
    
    if (@{$self->{_having}}) {
        $sql .= " HAVING " . join(" AND ", @{$self->{_having}});
    }
    
    if (@{$self->{_order}}) {
        $sql .= " ORDER BY " . join(", ", @{$self->{_order}});
    }
    
    $sql .= " LIMIT "  . $self->{_limit}  if defined $self->{_limit};
    $sql .= " OFFSET " . $self->{_offset} if defined $self->{_offset};
    
    return ($sql, @{$self->{_params}});
}

sub to_sql { (shift->build)[0] }

package main;

use DBI;

my $q = Query::Builder->new;

# Simple query
my $simple = $q->from('users')
               ->where('active = ?', 1)
               ->order_by('name')
               ->limit(10);

my ($sql, @params) = $simple->build;
printf "Simple:\n  %s\n  Params: %s\n\n", $sql, join(",", @params);

# Complex query
my $complex = $q->select('u.id', 'u.name', 'COUNT(o.id) AS orders', 'SUM(o.total) AS revenue')
               ->from('users', 'u')
               ->left_join('orders o', 'u.id = o.user_id')
               ->where('u.role = ?', 'customer')
               ->where('u.created_at > ?', '2024-01-01')
               ->group_by('u.id', 'u.name')
               ->having('COUNT(o.id) > ?', 0)
               ->order_by('revenue', 'desc')
               ->page(1, 20);

($sql, @params) = $complex->build;
printf "Complex:\n  %s\n  Params: %s\n\n", $sql, join(",", @params);

# Reuse base query (immutable)
my $base = $q->from('products')->where('active = ?', 1);

my $cheap = $base->where('price < ?', 50)->order_by('price');
my $premium = $base->where('price >= ?', 100)->order_by('price', 'desc');

printf "Cheap:   %s\n", $cheap->to_sql;
printf "Premium: %s\n", $premium->to_sql;
```

---

## Step 219: Template Method Pattern

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Template Method Pattern
# =====================

package Report::Base;

sub new {
    my ($class, %args) = @_;
    return bless {
        title  => $args{title} // "Report",
        data   => $args{data}  // [],
    }, $class;
}

# Template method — defines the algorithm skeleton
sub generate {
    my $self = shift;
    
    my @output;
    push @output, $self->header;
    push @output, $self->process_data($self->{data});
    push @output, $self->footer;
    
    return join("\n", @output);
}

# Default implementations (can be overridden)
sub header { "=== $_[0]->{title} ===" }
sub footer { "--- End of Report ---" }

# Abstract methods (subclasses MUST implement)
sub process_data { die ref(shift) . " must implement process_data\n" }

sub format_number { sprintf("%.2f", $_[1]) }
sub format_date   { my @t = localtime; sprintf("%04d-%02d-%02d", $t[5]+1900, $t[4]+1, $t[3]) }

package Report::TextTable;
our @ISA = ('Report::Base');

sub process_data {
    my ($self, $data) = @_;
    return "(no data)" unless @$data;
    
    my @cols = sort keys %{$data->[0]};
    my %widths;
    
    # Calculate column widths
    for my $col (@cols) {
        $widths{$col} = length($col);
        for my $row (@$data) {
            my $len = length($row->{$col} // "");
            $widths{$col} = $len if $len > $widths{$col};
        }
    }
    
    my $fmt  = join(" | ", map { "%-$widths{$_}s" } @cols);
    my $sep  = join("-+-", map { "-" x $widths{$_} } @cols);
    
    my @lines = (
        sprintf($fmt, @cols),
        $sep,
        (map { sprintf($fmt, map { $_ // "" } @{$_}{@cols}) } @$data),
    );
    
    return @lines;
}

package Report::CSV;
our @ISA = ('Report::Base');

sub header { "" }
sub footer { "" }

sub process_data {
    my ($self, $data) = @_;
    return () unless @$data;
    
    my @cols = sort keys %{$data->[0]};
    my @lines = (join(",", @cols));
    
    for my $row (@$data) {
        push @lines, join(",", map {
            my $v = $row->{$_} // "";
            $v =~ s/"/""/g;
            $v =~ /[,"]/ ? "\"$v\"" : $v;
        } @cols);
    }
    
    return @lines;
}

package Report::HTML;
our @ISA = ('Report::Base');

sub header {
    my $self = shift;
    return "<html><body><h1>$self->{title}</h1><table border='1'>";
}

sub footer { "</table></body></html>" }

sub process_data {
    my ($self, $data) = @_;
    return ("<tr><td>No data</td></tr>") unless @$data;
    
    my @cols = sort keys %{$data->[0]};
    my @lines = ("<tr>" . join("", map { "<th>$_</th>" } @cols) . "</tr>");
    
    for my $row (@$data) {
        push @lines, "<tr>" . join("", map {
            "<td>" . ($row->{$_}//"") . "</td>"
        } @cols) . "</tr>";
    }
    
    return @lines;
}

package main;

my @data = (
    { name => "Alice", dept => "Engineering", salary => 85000 },
    { name => "Bob",   dept => "Marketing",   salary => 72000 },
    { name => "Carol", dept => "Engineering", salary => 92000 },
);

printf "=== Text Table ===\n";
print Report::TextTable->new(title => "Employee Report", data => \@data)->generate;
print "\n\n";

printf "=== CSV ===\n";
print Report::CSV->new(title => "Employee Report", data => \@data)->generate;
print "\n\n";

printf "=== HTML (first 200 chars) ===\n";
my $html = Report::HTML->new(title => "Employee Report", data => \@data)->generate;
print substr($html, 0, 200), "...\n";
```

---

## Step 220: โปรแกรมสรุป — Plugin Architecture

```perl
#!/usr/bin/perl
# plugin_arch.pl — Extensible plugin architecture
use strict;
use warnings;

# =====================
# Plugin Base
# =====================

package Plugin;

sub new {
    my ($class, %args) = @_;
    return bless {
        name        => $args{name}    // ref($class),
        version     => $args{version} // "1.0",
        description => $args{description} // "",
        enabled     => 1,
    }, $class;
}

sub name        { $_[0]->{name}        }
sub version     { $_[0]->{version}     }
sub description { $_[0]->{description} }
sub enabled     { $_[0]->{enabled}     }
sub enable      { $_[0]->{enabled} = 1 }
sub disable     { $_[0]->{enabled} = 0 }

# Lifecycle hooks (override in plugins)
sub on_load    {}
sub on_unload  {}
sub on_enable  {}
sub on_disable {}

# =====================
# Plugin Manager
# =====================

package PluginManager;

sub new {
    my ($class) = @_;
    return bless {
        plugins  => {},
        hooks    => {},
        events   => {},
    }, $class;
}

sub register {
    my ($self, $plugin) = @_;
    my $name = $plugin->name;
    
    die "Plugin '$name' already registered\n" if $self->{plugins}{$name};
    
    $self->{plugins}{$name} = $plugin;
    $plugin->on_load($self);
    printf "  [PM] Loaded plugin: %s v%s\n", $name, $plugin->version;
    return $self;
}

sub unregister {
    my ($self, $name) = @_;
    if (my $plugin = $self->{plugins}{$name}) {
        $plugin->on_unload($self);
        delete $self->{plugins}{$name};
        printf "  [PM] Unloaded plugin: %s\n", $name;
    }
}

sub get    { $_[0]->{plugins}{$_[1]} }
sub list   { values %{$_[0]->{plugins}} }

sub hook {
    my ($self, $hook_name, $fn) = @_;
    push @{$self->{hooks}{$hook_name}}, $fn;
}

sub apply_hook {
    my ($self, $hook_name, @args) = @_;
    my $value = $args[0];
    for my $fn (@{$self->{hooks}{$hook_name} // []}) {
        $value = $fn->($value, @args[1..$#args]);
    }
    return $value;
}

sub trigger {
    my ($self, $event, @args) = @_;
    $_->(@args) for @{$self->{events}{$event} // []};
}

sub on {
    my ($self, $event, $fn) = @_;
    push @{$self->{events}{$event}}, $fn;
}

# =====================
# Concrete plugins
# =====================

package Plugin::Logger;
our @ISA = ('Plugin');

sub new {
    my ($class) = @_;
    return $class->SUPER::new(name => "logger", version => "1.0", description => "Request logging");
}

sub on_load {
    my ($self, $pm) = @_;
    $pm->hook('request.before', sub {
        my ($req) = @_;
        printf "  [LOG] %s %s\n", $req->{method}, $req->{path};
        return $req;
    });
    $pm->hook('request.after', sub {
        my ($res) = @_;
        printf "  [LOG] Response: %d\n", $res->{status};
        return $res;
    });
}

package Plugin::Auth;
our @ISA = ('Plugin');

sub new {
    my ($class) = @_;
    return $class->SUPER::new(name => "auth", version => "2.0", description => "Authentication");
}

sub on_load {
    my ($self, $pm) = @_;
    $pm->hook('request.before', sub {
        my ($req) = @_;
        if ($req->{path} =~ m{^/private}) {
            my $token = $req->{headers}{'Authorization'} // "";
            unless ($token eq "Bearer valid_token") {
                $req->{_reject} = { status => 401, body => "Unauthorized" };
            }
        }
        return $req;
    });
}

package Plugin::Cache;
our @ISA = ('Plugin');

sub new {
    my ($class) = @_;
    return $class->SUPER::new(name => "cache", version => "1.1", description => "Response caching");
}

my %_cache;

sub on_load {
    my ($self, $pm) = @_;
    $pm->hook('request.before', sub {
        my ($req) = @_;
        if ($req->{method} eq 'GET' && exists $_cache{$req->{path}}) {
            $req->{_cached} = $_cache{$req->{path}};
            printf "  [CACHE] Hit: %s\n", $req->{path};
        }
        return $req;
    });
    $pm->hook('request.after', sub {
        my ($res, $req) = @_;
        if ($req && $req->{method} eq 'GET' && !$res->{_no_cache}) {
            $_cache{$req->{path}} = $res;
            printf "  [CACHE] Stored: %s\n", $req->{path};
        }
        return $res;
    });
}

# =====================
# Application
# =====================

package App;

sub new {
    my ($class) = @_;
    my $pm = PluginManager->new;
    return bless {
        pm     => $pm,
        routes => {},
    }, $class;
}

sub use_plugin { my ($self,$p)=@_; $self->{pm}->register($p); $self }

sub get {
    my ($self, $path, $handler) = @_;
    $self->{routes}{"GET:$path"} = $handler;
    return $self;
}

sub handle {
    my ($self, %req_data) = @_;
    
    my $req = { method => 'GET', path => '/', headers => {}, %req_data };
    my $pm  = $self->{pm};
    
    # Apply request hooks
    $req = $pm->apply_hook('request.before', $req);
    
    # Check for rejection or cache hit
    my $res;
    if ($req->{_reject}) {
        $res = $req->{_reject};
    } elsif ($req->{_cached}) {
        $res = $req->{_cached};
    } else {
        # Route handler
        my $key     = "$req->{method}:$req->{path}";
        my $handler = $self->{routes}{$key};
        if ($handler) {
            $res = $handler->($req);
        } else {
            $res = { status => 404, body => "Not found" };
        }
    }
    
    # Apply response hooks
    $res = $pm->apply_hook('request.after', $res, $req);
    
    return $res;
}

package main;

print "=== Plugin Architecture Demo ===\n\n";

my $app = App->new;

print "Loading plugins:\n";
$app->use_plugin(Plugin::Logger->new)
    ->use_plugin(Plugin::Auth->new)
    ->use_plugin(Plugin::Cache->new);

$app->get('/', sub { { status => 200, body => "Welcome home!" } });
$app->get('/api/users', sub { { status => 200, body => '["alice","bob"]' } });
$app->get('/private/data', sub { { status => 200, body => "Secret data" } });

print "\nHandling requests:\n";

for my $request (
    { method => 'GET', path => '/' },
    { method => 'GET', path => '/api/users' },
    { method => 'GET', path => '/api/users' },   # cached
    { method => 'GET', path => '/private/data', headers => {} },
    { method => 'GET', path => '/private/data', headers => { Authorization => "Bearer valid_token" } },
) {
    printf "\n--- %s %s ---\n", $request->{method}, $request->{path};
    my $res = $app->handle(%$request);
    printf "Response %d: %s\n", $res->{status}, $res->{body};
}

# Plugin info
printf "\n=== Plugin Status ===\n";
for my $plugin ($app->{pm}->list) {
    printf "  %s v%s — %s\n", $plugin->name, $plugin->version, $plugin->description;
}
```

---

## สรุป Part 22

ใน Part นี้คุณได้เรียนรู้:
- ✅ OOP พื้นฐาน ทบทวน
- ✅ Accessor generator (mk_accessors)
- ✅ Role/Mixin pattern
- ✅ Operator overloading
- ✅ Metaprogramming (AUTOLOAD, eval)
- ✅ Design Patterns: Singleton, Factory, Observer, Command
- ✅ Exception hierarchy
- ✅ Builder Pattern
- ✅ Template Method Pattern
- ✅ Plugin Architecture

**ถัดไป: [Part 23 — Moose OOP Framework](part_23.md)**
