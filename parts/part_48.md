# Part 48: Advanced CPAN Module Development
## Steps 471-480: Module layout, POD, testing, distribution

---

## Step 471: Module Structure & Layout

```
Module directory layout:
MyModule-1.00/
├── lib/
│   └── MyModule/
│       ├── MyModule.pm       (main package)
│       ├── Utils.pm          (utility subpackage)
│       └── Types.pm          (type definitions)
├── t/
│   ├── 00_load.t             (basic load test)
│   ├── 01_basic.t            (unit tests)
│   └── 99_pod.t              (POD coverage)
├── xt/
│   └── author/
│       └── 00_critic.t       (Perl::Critic)
├── Changes                   (changelog)
├── MANIFEST                  (file list)
├── MANIFEST.SKIP             (exclusion patterns)
├── Makefile.PL               (build system)
├── META.json                 (metadata)
└── README.md
```

```perl
#!/usr/bin/perl
# lib/MyModule.pm

package MyModule;

use strict;
use warnings;
our $VERSION = "1.00";

# Constructor
sub new {
    my ($class, %args) = @_;
    my $self = bless {
        name    => $args{name}    // "default",
        verbose => $args{verbose} // 0,
        _data   => {},
    }, $class;
    $self->_init(%args);
    return $self;
}

sub _init {
    my ($self, %args) = @_;
    $self->{_data}{created_at} = time();
}

# Accessors (read-write)
sub name {
    my $self = shift;
    $self->{name} = shift if @_;
    return $self->{name};
}

sub verbose {
    my $self = shift;
    $self->{verbose} = shift if @_;
    return $self->{verbose};
}

# Methods
sub process {
    my ($self, $input) = @_;
    die "Input required" unless defined $input;
    return uc $input;
}

sub to_hash {
    my $self = shift;
    return {
        name       => $self->{name},
        verbose    => $self->{verbose},
        created_at => $self->{_data}{created_at},
    };
}

# Class method
sub version { $VERSION }

# Overload operators
use overload
    '""'  => sub { "MyModule(name=" . $_[0]->{name} . ")" },
    'eq'  => sub { $_[0]->{name} eq (ref $_[1] ? $_[1]->{name} : $_[1]) },
    '0+'  => sub { 1 },
    ;

1;  # Module must end with 1; or a true value

__END__

=head1 NAME

MyModule - Example CPAN module with full documentation

=head1 SYNOPSIS

    use MyModule;

    my $obj = MyModule->new(name => "test");
    my $result = $obj->process("hello");  # Returns "HELLO"
    print $obj;  # Prints: MyModule(name=test)

=head1 DESCRIPTION

MyModule demonstrates proper CPAN module structure with documentation,
accessors, operator overloading, and OOP best practices.

=head1 METHODS

=head2 new(%args)

Creates a new MyModule instance.

    my $obj = MyModule->new(name => "example", verbose => 1);

B<Arguments:>

=over 4

=item name => $string (optional, default: "default")

The name for this module instance.

=item verbose => $bool (optional, default: 0)

Whether to enable verbose output.

=back

=head2 process($input)

Processes the given input string.

    my $result = $obj->process("hello");  # "HELLO"

=head2 name([$new_name])

Get or set the name attribute.

=head2 to_hash()

Returns a hashref representation of the object.

=head1 AUTHOR

Your Name <your@email.com>

=head1 LICENSE

This library is free software; you can redistribute it and/or modify
it under the same terms as Perl itself.

=cut
```

---

## Step 472: Makefile.PL & Build System

```perl
#!/usr/bin/perl
# Makefile.PL — ExtUtils::MakeMaker based

use strict;
use warnings;
use ExtUtils::MakeMaker;

WriteMakefile(
    NAME             => "MyModule",
    VERSION_FROM     => "lib/MyModule.pm",
    ABSTRACT_FROM    => "lib/MyModule.pm",
    AUTHOR           => "Your Name <your\@email.com>",
    LICENSE          => "perl",
    MIN_PERL_VERSION => "5.020",
    
    PREREQ_PM => {
        "Scalar::Util"  => "1.50",
        "List::Util"    => "1.56",
        "Carp"          => "0",
        "POSIX"         => "0",
    },
    
    TEST_REQUIRES => {
        "Test::More"       => "1.302",
        "Test::Exception"  => "0.43",
        "Test::Deep"       => "1.130",
    },
    
    BUILD_REQUIRES => {
        "ExtUtils::MakeMaker" => "7.00",
    },
    
    META_MERGE => {
        "meta-spec" => { version => 2 },
        resources => {
            repository => {
                type => "git",
                url  => "https://github.com/yourusername/MyModule.git",
                web  => "https://github.com/yourusername/MyModule",
            },
            bugtracker => {
                web => "https://github.com/yourusername/MyModule/issues",
            },
        },
        keywords => ["example", "perl", "module"],
    },
    
    dist => { COMPRESS => "gzip -9f", SUFFIX => "gz" },
    clean => { FILES => "MyModule-*" },
);
```

---

## Step 473: Test Suite

```perl
#!/usr/bin/perl
# t/01_basic.t — comprehensive test suite

use strict;
use warnings;
use Test::More;

# Basic loading
BEGIN { use_ok("MyModule") }

# Construction
my $obj = MyModule->new(name => "test");
isa_ok($obj, "MyModule", "new() returns object");

# Accessors
is($obj->name, "test", "name() getter");
$obj->name("new_name");
is($obj->name, "new_name", "name() setter");

is($obj->verbose, 0, "verbose() default");
$obj->verbose(1);
is($obj->verbose, 1, "verbose() setter");

# Methods
is($obj->process("hello"), "HELLO", "process() uppercases");
is($obj->process("world"), "WORLD", "process() uppercases 2");

eval { $obj->process(undef) };
like($@, qr/Input required/, "process() dies without input");

# to_hash
my $h = $obj->to_hash;
is(ref $h, "HASH", "to_hash() returns hashref");
is($h->{name}, "new_name", "to_hash() name");
ok(defined $h->{created_at}, "to_hash() created_at exists");

# Overloading
my $str = "$obj";
like($str, qr/MyModule\(name=/, "stringify overload");

my $obj2 = MyModule->new(name => "same");
my $obj3 = MyModule->new(name => "same");
ok($obj2 eq $obj3, "eq overload (same name)");
ok(!($obj2 eq $obj), "eq overload (different name)");

# Class method
like(MyModule->version, qr/^\d+\.\d+/, "version() returns version string");

done_testing();

# --- Exception tests ---
package TestExceptions;
use Test::More;

sub run_exception_tests {
    eval { MyModule->new->process() };
    # Test dies with useful error
    ok $@, "dies without argument";
}
```

---

## Step 474: POD Documentation Best Practices

```perl
#!/usr/bin/perl
# lib/MyModule/HTTP.pm — Example with complete POD

package MyModule::HTTP;

use strict;
use warnings;
our $VERSION = "1.00";

=head1 NAME

MyModule::HTTP - HTTP utilities for MyModule

=head1 VERSION

Version 1.00

=head1 SYNOPSIS

    use MyModule::HTTP;

    my $client = MyModule::HTTP->new(
        base_url => "https://api.example.com",
        timeout  => 30,
    );

    my $response = $client->get("/users");
    if ($response->ok) {
        my $users = $response->json;
        for my $user (@$users) {
            printf "%s: %s\n", $user->{id}, $user->{name};
        }
    }

=head1 DESCRIPTION

C<MyModule::HTTP> provides a simple, chainable HTTP client built on top
of L<LWP::UserAgent>. It supports JSON request/response handling,
authentication helpers, and retry logic.

=head1 CONSTRUCTOR

=head2 new(%options)

    my $client = MyModule::HTTP->new(
        base_url  => "https://api.example.com",  # Base URL prefix
        timeout   => 30,                          # Seconds (default: 30)
        retries   => 3,                           # Retry count (default: 3)
        user_agent=> "MyApp/1.0",                # UA string
    );

=head1 METHODS

=head2 get($path, %options)

Send a GET request.

    my $resp = $client->get("/users", headers => { "X-Custom" => "value" });

=head2 post($path, %options)

Send a POST request.

    my $resp = $client->post("/users",
        json => { name => "Alice", email => "alice\@example.com" },
    );

=head2 auth_bearer($token)

Set Bearer token authentication for subsequent requests.

    $client->auth_bearer("my-jwt-token");

=head2 auth_basic($username, $password)

Set Basic authentication.

    $client->auth_basic("admin", "secret");

=head1 RESPONSE METHODS

The response object supports:

=over 4

=item C<< $resp->ok >> — True if status 2xx

=item C<< $resp->status >> — HTTP status code

=item C<< $resp->body >> — Response body as string

=item C<< $resp->json >> — Decoded JSON body (hashref or arrayref)

=item C<< $resp->header($name) >> — Response header value

=back

=head1 ERROR HANDLING

Methods throw exceptions on network errors. Use eval to catch them:

    my $resp = eval { $client->get("/endpoint") };
    if ($@) {
        warn "Network error: $@";
    }

=head1 EXAMPLES

=head2 Pagination

    my $page = 1;
    while (1) {
        my $resp = $client->get("/items", query => { page => $page, per_page => 100 });
        last unless $resp->ok;
        my $items = $resp->json;
        last unless @$items;
        process($_) for @$items;
        $page++;
    }

=head2 Upload file

    my $resp = $client->post("/upload",
        multipart => [
            file => ["path/to/file.pdf", "file.pdf", Content_Type => "application/pdf"],
        ],
    );

=head1 SEE ALSO

L<LWP::UserAgent>, L<HTTP::Request>, L<HTTP::Response>, L<MyModule>

=head1 AUTHOR

Your Name <your\@email.com>

=head1 COPYRIGHT AND LICENSE

Copyright (C) 2024 by Your Name

This library is free software; you can redistribute it and/or modify
it under the same terms as Perl itself, either Perl version 5.20.0 or,
at your option, any later version of Perl 5 you may have installed.

=cut

# Implementation follows (abbreviated for example)
sub new {
    my ($class, %opts) = @_;
    return bless {
        base_url  => $opts{base_url} // "",
        timeout   => $opts{timeout}  // 30,
        retries   => $opts{retries}  // 3,
        headers   => {},
    }, $class;
}

sub auth_bearer {
    my ($self, $token) = @_;
    $self->{headers}{"Authorization"} = "Bearer $token";
    return $self;
}

sub auth_basic {
    my ($self, $user, $pass) = @_;
    require MIME::Base64;
    $self->{headers}{"Authorization"} = "Basic " . MIME::Base64::encode_base64("$user:$pass", "");
    return $self;
}

1;
```

---

## Step 475: Type System & Validation

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Types;
use Carp qw(croak);

# Type constructors
sub Str  { my $v=shift; croak "Not a string: ".ref($v) if ref $v; $v }
sub Int  {
    my $v=shift;
    croak "Not an integer: $v" unless defined $v && $v =~ /^-?\d+$/;
    $v+0
}
sub Num  {
    my $v=shift;
    croak "Not a number: $v" unless defined $v && $v =~ /^-?\d+(?:\.\d+)?(?:[eE][+-]?\d+)?$/;
    $v+0
}
sub Bool { $_[0] ? 1 : 0 }
sub ArrayRef {
    my $v = shift;
    croak "Not an arrayref" unless ref $v eq "ARRAY";
    $v
}
sub HashRef {
    my $v = shift;
    croak "Not a hashref" unless ref $v eq "HASH";
    $v
}
sub CodeRef {
    my $v = shift;
    croak "Not a coderef" unless ref $v eq "CODE";
    $v
}
sub Maybe {
    my ($type_fn, $v) = @_;
    return undef unless defined $v;
    return $type_fn->($v);
}
sub Enum {
    my ($allowed, $v) = @_;
    croak "Not in enum (".join("|",@$allowed)."): $v" unless grep { $_ eq $v } @$allowed;
    $v
}
sub NonEmptyStr {
    my $v = Str(shift);
    croak "Cannot be empty" unless length $v;
    $v
}
sub PositiveInt {
    my $v = Int(shift);
    croak "Must be positive" unless $v > 0;
    $v
}
sub Email {
    my $v = Str(shift);
    croak "Invalid email: $v" unless $v =~ /^[^@\s]+@[^@\s]+\.[^@\s]+$/;
    $v
}
}

{
package Validator;
use Carp qw(croak);

sub new {
    my ($class, %schema) = @_;
    return bless { schema => \%schema }, $class;
}

sub validate {
    my ($self, %data) = @_;
    my %result;
    my @errors;
    
    for my $field (keys %{$self->{schema}}) {
        my $rules = $self->{schema}{$field};
        my $value = $data{$field};
        
        # Required check
        if ($rules->{required} && !defined $value) {
            push @errors, "$field is required";
            next;
        }
        next unless defined $value;
        
        # Default
        $value //= $rules->{default};
        
        # Type check
        if (my $type = $rules->{type}) {
            eval { $value = $type->($value) };
            if ($@) { push @errors, "$field: $@"; next; }
        }
        
        # Custom validator
        if (my $validator = $rules->{validator}) {
            eval { $validator->($value) };
            if ($@) { push @errors, "$field: $@"; next; }
        }
        
        $result{$field} = $value;
    }
    
    return { ok => !@errors, errors => \@errors, data => \%result };
}
}

package main;

printf "=== Type System & Validation ===\n\n";

# Type tests
printf "Type coercions:\n";
eval { printf "  Int('42') = %d\n", Types::Int("42") };
eval { printf "  Num('3.14') = %s\n", Types::Num("3.14") };
eval { printf "  Bool(0) = %d, Bool('yes') = %d\n", Types::Bool(0), Types::Bool("yes") };
eval { printf "  Enum(['a','b','c'], 'b') = %s\n", Types::Enum([qw(a b c)], 'b') };
eval { printf "  PositiveInt(5) = %d\n", Types::PositiveInt(5) };

printf "\nType errors:\n";
for my $test (
    ['Int', sub { Types::Int("abc") }],
    ['PositiveInt', sub { Types::PositiveInt(-1) }],
    ['Email', sub { Types::Email("not-an-email") }],
    ['Enum', sub { Types::Enum([qw(a b c)], 'd') }],
) {
    eval { $test->[1]->() };
    printf "  %s error: %s\n", $test->[0], $@ =~ s/ at .*//sr;
}

# Validator
printf "\nSchema validation:\n";
my $validator = Validator->new(
    name  => { required => 1, type => \&Types::NonEmptyStr },
    email => { required => 1, type => \&Types::Email },
    age   => { required => 0, type => \&Types::PositiveInt },
    role  => { required => 0, type => sub { Types::Enum([qw(admin user guest)], $_[0]) }, default => "user" },
);

for my $data (
    { name => "Alice", email => "alice\@test.com", age => 30, role => "admin" },
    { name => "",      email => "alice\@test.com" },
    { name => "Bob",   email => "not-email",       age => -5 },
    { email => "carol\@test.com" },
) {
    my $r = $validator->validate(%$data);
    if ($r->{ok}) {
        printf "  VALID: %s\n", join(", ", map{"$_=$r->{data}{$_}"} sort keys %{$r->{data}});
    } else {
        printf "  INVALID: %s\n", join("; ", @{$r->{errors}});
    }
}
```

---

## Step 476-480: Roles, Mixins, Meta API, Plugins, Capstone

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Role system without Moose
{
package Role;

sub apply_to {
    my ($role_class, $target_class) = @_;
    no strict "refs";
    for my $method (keys %{"${role_class}::"}) {
        next unless defined &{"${role_class}::${method}"};
        next if $method =~ /^(apply_to|requires|import)$/;
        *{"${target_class}::${method}"} = \&{"${role_class}::${method}"};
    }
}

sub requires {
    my ($role_class, @methods) = @_;
    push @{"${role_class}::REQUIRED"}, @methods;
}
}

{
package Role::Printable;

sub to_string { "Object(" . ref($_[0]) . ")" }
sub print_self { printf "%s\n", $_[0]->to_string }
sub inspect {
    my $self = shift;
    printf "%s {\n", ref($self);
    for my $key (sort keys %$self) {
        next if $key =~ /^_/;
        printf "  %s: %s\n", $key, $self->{$key} // "undef";
    }
    printf "}\n";
}
}

{
package Role::Serializable;

sub to_json {
    my $self = shift;
    require JSON::PP;
    my %h = map { $_ => $self->{$_} } grep { !/^_/ } keys %$self;
    return JSON::PP->new->encode(\%h);
}

sub from_json {
    my ($class, $json) = @_;
    require JSON::PP;
    my $data = JSON::PP->new->decode($json);
    return $class->new(%$data);
}

sub clone {
    my $self = shift;
    my %copy = %$self;
    return bless \%copy, ref $self;
}
}

{
package Role::Observable;

my %_observers;

sub on {
    my ($self, $event, $handler) = @_;
    my $id = refaddr($self);
    push @{$_observers{$id}{$event}}, $handler;
    return $self;
}

sub emit {
    my ($self, $event, @args) = @_;
    my $id = refaddr($self);
    my $handlers = $_observers{$id}{$event} // [];
    $_->(@args) for @$handlers;
}

sub off {
    my ($self, $event) = @_;
    delete $_observers{refaddr($self)}{$event};
}

sub refaddr { 0+$_[0] }
}

# Apply roles to a class
{
package Product;

sub new {
    my ($class, %args) = @_;
    return bless {
        id    => $args{id},
        name  => $args{name},
        price => $args{price},
        stock => $args{stock} // 0,
    }, $class;
}

sub name  { $_[0]->{name}  }
sub price { $_[0]->{price} }
sub stock { $_[0]->{stock} }

Role::Printable->apply_to("Product");
Role::Serializable->apply_to("Product");
Role::Observable->apply_to("Product");

sub to_string { sprintf "Product(%s, \$%.2f)", $_[0]->{name}, $_[0]->{price} }

sub update_stock {
    my ($self, $delta) = @_;
    my $old = $self->{stock};
    $self->{stock} += $delta;
    $self->emit("stock_changed", $old, $self->{stock});
}

sub apply_discount {
    my ($self, $pct) = @_;
    my $old = $self->{price};
    $self->{price} *= (1 - $pct/100);
    $self->emit("price_changed", $old, $self->{price});
}
}

package main;

printf "=== Roles & Mixins ===\n\n";

my $product = Product->new(id=>1, name=>"Perl Book", price=>49.99, stock=>100);

# Observable events
$product->on("stock_changed", sub {
    my ($old, $new) = @_;
    printf "  Stock changed: %d -> %d\n", $old, $new;
});

$product->on("price_changed", sub {
    my ($old, $new) = @_;
    printf "  Price changed: \$%.2f -> \$%.2f\n", $old, $new;
});

printf "Product: %s\n\n", $product->to_string;

printf "Updating stock:\n";
$product->update_stock(-10);
$product->update_stock(50);

printf "\nApplying 20%% discount:\n";
$product->apply_discount(20);

printf "\nInspect:\n";
$product->inspect;

printf "\nSerialized:\n%s\n\n", $product->to_json;

# Clone
my $copy = $product->clone;
$copy->{name} = "Perl Book (Copy)";
printf "Original: %s\n", $product->to_string;
printf "Clone:    %s\n\n", $copy->to_string;

# Plugin system
printf "=== Plugin System ===\n\n";

{
package PluginManager;

my %_plugins = ();

sub register {
    my ($class, $name, $plugin) = @_;
    $_plugins{$name} = $plugin;
    printf "  Plugin '%s' registered\n", $name;
}

sub load {
    my ($class, $name, $app) = @_;
    my $plugin = $_plugins{$name} or die "Unknown plugin: $name";
    $plugin->install($app) if $plugin->can("install");
    printf "  Plugin '%s' loaded\n", $name;
    return $plugin;
}

sub list { sort keys %_plugins }
}

{
package Plugin::Logger;

sub install {
    my ($self, $app) = @_;
    $app->{_logger} = $self;
}

sub log {
    my ($self, $level, $msg) = @_;
    printf "[%s] %s\n", uc $level, $msg;
}
}

{
package Plugin::Cache;

sub new { bless { store => {} }, shift }

sub install {
    my ($self, $app) = @_;
    $app->{_cache} = $self;
}

sub get { $_[0]->{store}{$_[1]} }
sub set { $_[0]->{store}{$_[1]} = $_[2] }
sub clear { delete $_[0]->{store}{$_[1]} }
}

PluginManager->register("logger", Plugin::Logger->new);
PluginManager->register("cache",  Plugin::Cache->new);

my $app = { plugins => {} };
PluginManager->load("logger", $app);
PluginManager->load("cache",  $app);

printf "\nAvailable plugins: %s\n", join(", ", PluginManager->list);

$app->{_logger}->log("info", "Application started");
$app->{_cache}->set("user:1", { name => "Alice" });
my $user = $app->{_cache}->get("user:1");
printf "Cache get user:1 -> name=%s\n", $user->{name};
$app->{_logger}->log("debug", "User retrieved from cache");
```

---

## สรุป Part 48 — Advanced CPAN Module Development

### สิ่งที่เรียนรู้:
- **Module Layout** — Directory structure, `lib/`, `t/`, `xt/`, MANIFEST
- **Makefile.PL** — ExtUtils::MakeMaker, prereqs, META_MERGE, dist config
- **Test Suite** — Test::More, isa_ok, like, done_testing, eval/exception
- **POD Documentation** — NAME, SYNOPSIS, DESCRIPTION, METHODS, SEE ALSO
- **Type System** — Custom type functions, coercion, validation schema
- **Roles/Mixins** — No-Moose role system via symbol table injection
- **Observable Pattern** — Event emitter with on/emit/off
- **Plugin System** — Registry-based plugin loader with install hooks

**ถัดไป: [Part 49 — Moose Advanced & Meta API](part_49.md)**
