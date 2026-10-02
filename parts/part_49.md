# Part 49: Moose Advanced & Meta API
## Steps 481-490: Type::Tiny, Meta API, Method Modifiers, Roles

---

## Step 481: Moose Basics Review & Type::Tiny

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Type::Tiny-style type system (pure Perl, no Moose required)
{
package Type::Tiny::Impl;

sub new {
    my ($class, %args) = @_;
    return bless {
        name        => $args{name} // "Unknown",
        constraint  => $args{constraint} // sub { 1 },
        coerce      => $args{coerce},
        message     => $args{message} // sub { "'$_[0]' is not a valid $_[1]->{name}" },
        parent      => $args{parent},
        _coercions  => $args{coercions} // {},
    }, $class;
}

sub check {
    my ($self, $value) = @_;
    # Check parent first
    if ($self->{parent}) {
        return 0 unless $self->{parent}->check($value);
    }
    return $self->{constraint}->($value);
}

sub validate {
    my ($self, $value) = @_;
    return undef if $self->check($value);
    return $self->{message}->($value, $self);
}

sub assert_valid {
    my ($self, $value) = @_;
    my $err = $self->validate($value);
    die $err . "\n" if $err;
    return $value;
}

sub coerce {
    my ($self, $value) = @_;
    return $value unless $self->{coerce};
    return $self->{coerce}->($value) if !$self->check($value);
    return $value;
}

sub add_coercion {
    my ($self, $from_type, $via) = @_;
    push @{$self->{_coercions}{$from_type}}, $via;
    return $self;
}

# Create subtype
sub where {
    my ($self, $constraint, %opts) = @_;
    return Type::Tiny::Impl->new(
        name       => $opts{name} // ("Anon_" . time()),
        parent     => $self,
        constraint => $constraint,
        message    => $opts{message},
    );
}
}

# Type library
{
package Types::Standard;

our $Any = Type::Tiny::Impl->new(
    name       => "Any",
    constraint => sub { 1 },
);

our $Defined = Type::Tiny::Impl->new(
    name       => "Defined",
    constraint => sub { defined $_[0] },
    message    => sub { "Value is not defined" },
);

our $Str = Type::Tiny::Impl->new(
    name       => "Str",
    parent     => $Defined,
    constraint => sub { !ref $_[0] },
    message    => sub { "'$_[0]' is not a Str" },
);

our $Num = Type::Tiny::Impl->new(
    name       => "Num",
    parent     => $Str,
    constraint => sub { $_[0] =~ /^-?\d+(?:\.\d+)?(?:[eE][+-]?\d+)?$/ },
    message    => sub { "'$_[0]' is not a Num" },
);

our $Int = Type::Tiny::Impl->new(
    name       => "Int",
    parent     => $Num,
    constraint => sub { $_[0] =~ /^-?\d+$/ },
    message    => sub { "'$_[0]' is not an Int" },
);

our $Bool = Type::Tiny::Impl->new(
    name       => "Bool",
    constraint => sub { !defined $_[0] || $_[0] =~ /^[01]$/ },
    coerce     => sub { $_[0] ? 1 : 0 },
    message    => sub { "Not a Bool" },
);

our $ArrayRef = Type::Tiny::Impl->new(
    name       => "ArrayRef",
    constraint => sub { ref $_[0] eq "ARRAY" },
    message    => sub { "Not an ArrayRef" },
);

our $HashRef = Type::Tiny::Impl->new(
    name       => "HashRef",
    constraint => sub { ref $_[0] eq "HASH" },
    message    => sub { "Not a HashRef" },
);

our $CodeRef = Type::Tiny::Impl->new(
    name       => "CodeRef",
    constraint => sub { ref $_[0] eq "CODE" },
    message    => sub { "Not a CodeRef" },
);

# Parameterized types
sub ArrayOf {
    my $item_type = shift;
    return Type::Tiny::Impl->new(
        name       => "ArrayOf[${\$item_type->{name}}]",
        parent     => $ArrayRef,
        constraint => sub {
            my $arr = shift;
            !grep { !$item_type->check($_) } @$arr;
        },
        message    => sub { "Not all array elements are $_[1]->{name}" },
    );
}

sub HashOf {
    my ($key_type, $val_type) = @_;
    return Type::Tiny::Impl->new(
        name   => "HashOf",
        parent => $HashRef,
        constraint => sub {
            my $h = shift;
            !grep { !$key_type->check($_) } keys %$h and
            !grep { !$val_type->check($_) } values %$h;
        },
    );
}

sub Maybe {
    my $type = shift;
    return Type::Tiny::Impl->new(
        name       => "Maybe[${\$type->{name}}]",
        constraint => sub { !defined $_[0] || $type->check($_[0]) },
    );
}

sub Enum {
    my @values = @_;
    my %set = map { $_ => 1 } @values;
    return Type::Tiny::Impl->new(
        name       => "Enum[" . join(",", @values) . "]",
        parent     => $Str,
        constraint => sub { $set{$_[0]} },
        message    => sub { "'$_[0]' not in (" . join(",",@values) . ")" },
    );
}
}

package main;

printf "=== Type::Tiny Implementation ===\n\n";

# Basic type checks
printf "Type checks:\n";
my %tests = (
    Str   => [$Types::Standard::Str,   ["hello", 42, undef]],
    Int   => [$Types::Standard::Int,   ["42", "3.14", "abc"]],
    Num   => [$Types::Standard::Num,   ["3.14", "abc", "1e5"]],
    Bool  => [$Types::Standard::Bool,  [0, 1, "yes"]],
);

for my $name (sort keys %tests) {
    my ($type, $vals) = @{$tests{$name}};
    printf "  %s:\n", $name;
    for my $v (@$vals) {
        printf "    check(%s) = %s\n",
            defined $v ? "'$v'" : "undef",
            $type->check($v // "") ? "true" : "false";
    }
}

# Parameterized types
printf "\nParameterized types:\n";
my $IntArray = Types::Standard::ArrayOf($Types::Standard::Int);
printf "  ArrayOf[Int] check([1,2,3]): %s\n", $IntArray->check([1,2,3]) ? "true" : "false";
printf "  ArrayOf[Int] check([1,'a']): %s\n", $IntArray->check([1,"a"]) ? "true" : "false";

my $Status = Types::Standard::Enum("active","inactive","pending");
printf "\n  Enum check('active'):  %s\n", $Status->check("active") ? "valid" : "invalid";
printf "  Enum check('deleted'): %s\n", $Status->check("deleted") ? "valid" : "invalid";
printf "  Enum error: %s\n", $Status->validate("deleted");

# Custom subtype
my $PositiveInt = $Types::Standard::Int->where(sub { $_[0] > 0 },
    name    => "PositiveInt",
    message => sub { "'$_[0]' is not a positive integer" },
);
printf "\n  PositiveInt check(5):  %s\n", $PositiveInt->check(5) ? "valid" : "invalid";
printf "  PositiveInt check(-1): %s\n", $PositiveInt->check(-1) ? "valid" : "invalid";
printf "  PositiveInt error(-1): %s\n", $PositiveInt->validate(-1)//"";
```

---

## Step 482: Meta API (Introspection)

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Meta::Class;

sub new {
    my ($class, $target) = @_;
    return bless { class => $target, _methods => {}, _attributes => {} }, $class;
}

sub name       { $_[0]->{class} }
sub superclass {
    no strict "refs";
    my @isa = @{"$_[0]->{class}::ISA"};
    return @isa;
}
sub all_superclasses {
    my $self = shift;
    my @supers;
    my @queue = ($self->superclass);
    while (@queue) {
        my $s = shift @queue;
        push @supers, $s;
        no strict "refs";
        push @queue, @{"${s}::ISA"};
    }
    return @supers;
}

sub methods {
    my $self = shift;
    no strict "refs";
    return grep { defined &{"$self->{class}::$_"} }
           grep { !/^(import|new|BEGIN|END|DESTROY|AUTOLOAD)$/ }
           keys %{"$self->{class}::"};
}

sub has_method {
    my ($self, $method) = @_;
    no strict "refs";
    return defined &{"$self->{class}::$method"};
}

sub add_method {
    my ($self, $name, $code) = @_;
    no strict "refs";
    *{"$self->{class}::$name"} = $code;
    return $self;
}

sub remove_method {
    my ($self, $name) = @_;
    no strict "refs";
    delete ${"$self->{class}::"}{$name};
    return $self;
}

sub add_attribute {
    my ($self, $name, %opts) = @_;
    $self->{_attributes}{$name} = { name => $name, %opts };
    
    # Generate accessor
    my $ro = $opts{is} eq "ro";
    no strict "refs";
    *{"$self->{class}::$name"} = sub {
        my $obj = shift;
        if (@_ && !$ro) {
            my $val = shift;
            if (my $type = $opts{isa}) {
                $type->assert_valid($val);
            }
            if (my $trigger = $opts{trigger}) {
                $trigger->($obj, $val, $obj->{$name});
            }
            $obj->{$name} = $val;
        }
        return $obj->{$name} // $opts{default};
    };
    
    # Predicate
    if ($opts{predicate}) {
        *{"$self->{class}::$opts{predicate}"} = sub { defined $_[0]->{$name} };
    }
    
    # Clearer
    if ($opts{clearer}) {
        *{"$self->{class}::$opts{clearer}"} = sub { delete $_[0]->{$name} };
    }
    
    return $self;
}

sub attributes { keys %{$_[0]->{_attributes}} }

sub make_immutable {
    my $self = shift;
    no strict "refs";
    # Freeze constructor (simplified)
    my $orig_new = \&{"$self->{class}::new"};
    *{"$self->{class}::new"} = sub {
        my ($cls, %args) = @_;
        # Validate required attributes
        for my $attr (values %{$self->{_attributes}}) {
            die "$attr->{name} is required" if $attr->{required} && !defined $args{$attr->{name}};
        }
        my $obj = $orig_new->($cls, %args);
        return $obj;
    };
}
}

{
package Animal;

sub new {
    my ($class, %args) = @_;
    return bless { %args }, $class;
}

sub speak { "..." }
}

{
package Dog;
our @ISA = ("Animal");

sub new {
    my ($class, %args) = @_;
    return bless { %args }, $class;
}

sub fetch { "fetching!" }
sub bark  { "Woof!" }
}

package main;

printf "=== Meta API ===\n\n";

my $meta = Meta::Class->new("Dog");
printf "Class: %s\n", $meta->name;
printf "Superclasses: %s\n", join(", ", $meta->superclass);
printf "All superclasses: %s\n", join(", ", $meta->all_superclasses);
printf "Methods: %s\n\n", join(", ", sort $meta->methods);

printf "has_method 'bark':  %s\n", $meta->has_method("bark")  ? "yes" : "no";
printf "has_method 'meow':  %s\n", $meta->has_method("meow")  ? "yes" : "no";

# Add method dynamically
$meta->add_method("meow", sub { "Dogs don't meow, but: " . $_[0]->bark });
my $dog = Dog->new(name => "Rex");
printf "\nDynamically added method:\n";
printf "  dog->meow: %s\n", $dog->meow;

# Add attributes with accessor generation
my $meta2 = Meta::Class->new("Dog");
$meta2->add_attribute("name",
    is       => "rw",
    default  => "unnamed",
    trigger  => sub { printf "  [trigger] name changed to: %s\n", $_[1] },
);
$meta2->add_attribute("age",
    is        => "rw",
    predicate => "has_age",
    clearer   => "clear_age",
);

printf "\nAttribute accessors:\n";
$dog->name("Buddy");
printf "  name = %s\n", $dog->name;
printf "  has_age = %s\n", $dog->has_age ? "yes" : "no";
$dog->age(3);
printf "  age = %d\n", $dog->age;
printf "  has_age = %s\n", $dog->has_age ? "yes" : "no";
$dog->clear_age;
printf "  has_age after clear = %s\n", $dog->has_age ? "yes" : "no";
```

---

## Step 483-490: Method Modifiers, Roles with Meta, Capstone

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Method modifiers (before/after/around)
{
package MethodModifiers;

sub apply {
    my ($class, $target_class, $type, $method_name, $modifier) = @_;
    no strict "refs";
    my $orig = \&{"${target_class}::${method_name}"};
    
    if ($type eq "before") {
        *{"${target_class}::${method_name}"} = sub {
            $modifier->(@_);
            goto &$orig;
        };
    }
    elsif ($type eq "after") {
        *{"${target_class}::${method_name}"} = sub {
            my @result = wantarray ? $orig->(@_) : (scalar $orig->(@_));
            $modifier->(@_);
            return wantarray ? @result : $result[0];
        };
    }
    elsif ($type eq "around") {
        *{"${target_class}::${method_name}"} = sub {
            $modifier->($orig, @_);
        };
    }
}
}

{
package Service;

sub new { bless { calls => 0, _log => [] }, shift }

sub process {
    my ($self, $data) = @_;
    $self->{calls}++;
    return "processed: $data";
}

sub fetch {
    my ($self, $id) = @_;
    return { id => $id, data => "item_$id" };
}
}

package main;

printf "=== Method Modifiers ===\n\n";

my $svc = Service->new;

# Before modifier (logging)
MethodModifiers->apply("Service", "before", "process", sub {
    my ($self, $data) = @_;
    push @{$self->{_log}}, "[BEFORE] process called with: $data";
});

# After modifier (audit)
MethodModifiers->apply("Service", "after", "process", sub {
    my ($self) = @_;
    push @{$self->{_log}}, "[AFTER] process completed (calls=$self->{calls})";
});

# Around modifier (caching)
my %cache;
MethodModifiers->apply("Service", "around", "fetch", sub {
    my ($orig, $self, $id) = @_;
    if (exists $cache{$id}) {
        push @{$self->{_log}}, "[CACHE HIT] id=$id";
        return $cache{$id};
    }
    my $result = $orig->($self, $id);
    $cache{$id} = $result;
    push @{$self->{_log}}, "[CACHE MISS] id=$id, stored in cache";
    return $result;
});

printf "Calling process:\n";
my $r1 = $svc->process("hello");
my $r2 = $svc->process("world");
printf "  result1: %s\n", $r1;
printf "  result2: %s\n", $r2;

printf "\nCalling fetch (with caching):\n";
my $f1 = $svc->fetch(42);
my $f2 = $svc->fetch(42);   # Should hit cache
my $f3 = $svc->fetch(99);
printf "  fetch(42) = id=%s data=%s\n", $f1->{id}, $f1->{data};
printf "  fetch(42) = id=%s data=%s (from cache)\n", $f2->{id}, $f2->{data};
printf "  fetch(99) = id=%s data=%s\n", $f3->{id}, $f3->{data};

printf "\nAudit log:\n";
printf "  %s\n", $_ for @{$svc->{_log}};

# Moose-style class with full meta
printf "\n=== Full Meta-Driven Class ===\n\n";

{
package Meta::Attribute;

sub new {
    my ($class, %opts) = @_;
    return bless {
        name        => $opts{name},
        is          => $opts{is}    // "rw",
        isa         => $opts{isa},
        required    => $opts{required} // 0,
        default     => $opts{default},
        lazy        => $opts{lazy}  // 0,
        weak_ref    => $opts{weak_ref} // 0,
        init_arg    => $opts{init_arg} // $opts{name},
        predicate   => $opts{predicate},
        clearer     => $opts{clearer},
        handles     => $opts{handles} // {},
        documentation => $opts{documentation} // "",
    }, $class;
}

sub install {
    my ($self, $class) = @_;
    no strict "refs";
    my $name = $self->{name};
    
    # Main accessor
    if ($self->{is} eq "ro") {
        *{"${class}::${name}"} = sub { $_[0]->{$name} };
    } else {
        *{"${class}::${name}"} = sub {
            my $obj = shift;
            if (@_) {
                my $val = shift;
                $obj->{$name} = $val;
            }
            return $obj->{$name} // (ref $self->{default} eq "CODE" ? $self->{default}->() : $self->{default});
        };
    }
    
    # Predicate
    if (my $pred = $self->{predicate}) {
        *{"${class}::${pred}"} = sub { defined $_[0]->{$name} };
    }
    
    # Clearer
    if (my $cl = $self->{clearer}) {
        *{"${class}::${cl}"} = sub { delete $_[0]->{$name} };
    }
    
    # Delegation
    for my $method (keys %{$self->{handles}}) {
        my $target = $self->{handles}{$method};
        *{"${class}::${method}"} = sub {
            my $obj = shift;
            return $obj->{$name}->$target(@_) if ref $obj->{$name};
            return undef;
        };
    }
}
}

{
package MetaClass;

my %registry;

sub meta {
    my $class = shift;
    $registry{$class} //= bless {
        class => $class,
        attrs => {},
        roles => [],
    }, "MetaClass";
    return $registry{$class};
}

sub has {
    my ($self, $name, %opts) = @_;
    my $attr = Meta::Attribute->new(name => $name, %opts);
    $self->{attrs}{$name} = $attr;
    $attr->install($self->{class});
    return $self;
}

sub with_role {
    my ($self, $role) = @_;
    push @{$self->{roles}}, $role;
    # Apply role methods
    no strict "refs";
    for my $method (keys %{"${role}::"}) {
        next unless defined &{"${role}::${method}"};
        next if $method =~ /^(import|BEGIN|END|DESTROY)$/;
        *{"$self->{class}::${method}"} = \&{"${role}::${method}"};
    }
    return $self;
}

sub generate_constructor {
    my $self  = shift;
    my $class = $self->{class};
    my $attrs = $self->{attrs};
    
    no strict "refs";
    *{"${class}::new"} = sub {
        my ($cls, %args) = @_;
        my $obj = bless {}, $cls;
        
        for my $name (keys %$attrs) {
            my $attr = $attrs->{$name};
            my $key  = $attr->{init_arg} // $name;
            
            if (exists $args{$key}) {
                $obj->{$name} = $args{$key};
            } elsif ($attr->{required}) {
                die "Required attribute '$name' missing\n";
            } elsif (defined $attr->{default} && !$attr->{lazy}) {
                $obj->{$name} = ref $attr->{default} eq "CODE"
                    ? $attr->{default}->()
                    : $attr->{default};
            }
        }
        
        $obj->BUILD(%args) if $obj->can("BUILD");
        return $obj;
    };
}
}

# Define a class using MetaClass
{
package Person;
our @ISA;

MetaClass->meta(__PACKAGE__)
    ->has("name",    is=>"ro", required=>1, isa=>"Str")
    ->has("age",     is=>"rw", isa=>"Int", predicate=>"has_age")
    ->has("email",   is=>"rw", predicate=>"has_email", clearer=>"clear_email")
    ->has("tags",    is=>"rw", default=>sub{[]}, handles=>{add_tag=>"push",tag_count=>"scalar"})
    ->generate_constructor;

sub greet { my $self=shift; sprintf "Hi, I'm %s%s", $self->name, $self->has_age ? " (age ".$self->age.")" : "" }
sub BUILD { my($self,%a)=@_; $self->{_created}=time() }
}

my $p = Person->new(name => "Alice", age => 30, email => "alice\@test.com");
printf "Name:    %s\n", $p->name;
printf "Age:     %s\n", $p->age;
printf "Has age: %s\n", $p->has_age ? "yes" : "no";
printf "Greet:   %s\n", $p->greet;
printf "Email:   %s\n", $p->email;

$p->add_tag("perl");
$p->add_tag("developer");
printf "Tags:    %d\n", $p->tag_count(@{$p->tags});

$p->clear_email;
printf "Email after clear: %s\n", $p->has_email ? "set" : "cleared";

eval { Person->new(age => 25) };
printf "\nMissing required attr: %s", $@;
```

---

## สรุป Part 49 — Moose Advanced & Meta API

### สิ่งที่เรียนรู้:
- **Type::Tiny** — Custom types, subtypes with `where`, parameterized types, Maybe/Enum/ArrayOf
- **Meta::Class** — Introspection (methods, superclasses, ISA), dynamic method injection
- **Attribute Meta** — Auto-generate accessors, predicates, clearers, triggers
- **Method Modifiers** — `before`/`after`/`around` (pre/post hooks, caching wrapper)
- **Meta-Driven Constructor** — `generate_constructor` with required/default/lazy
- **Delegation** — `handles` auto-generates delegating methods
- **Role Application** — Symbol-table role application via MetaClass

**ถัดไป: [Part 50 — Capstone Project: Full-Stack Perl Application](part_50.md)**
