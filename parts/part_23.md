# Part 23: Moose OOP Framework
## Steps 221-230: Modern Perl OOP ด้วย Moose

---

## Step 221: Moose พื้นฐาน

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Moose — Modern Perl OO
# =====================

{
package Person;
use Moose;

# has — attribute declaration
has 'name' => (
    is       => 'rw',           # read-write
    isa      => 'Str',          # type constraint
    required => 1,
);

has 'age' => (
    is      => 'rw',
    isa     => 'Int',
    default => 0,
);

has 'email' => (
    is        => 'rw',
    isa       => 'Str',
    predicate => 'has_email',   # generates has_email() method
    clearer   => 'clear_email', # generates clear_email() method
);

has 'tags' => (
    is      => 'rw',
    isa     => 'ArrayRef[Str]',
    default => sub { [] },      # MUST use sub{} for refs
    traits  => ['Array'],
    handles => {
        add_tag    => 'push',
        all_tags   => 'elements',
        tag_count  => 'count',
        has_tags   => 'count',
    },
);

has 'metadata' => (
    is      => 'rw',
    isa     => 'HashRef',
    default => sub { {} },
    traits  => ['Hash'],
    handles => {
        set_meta  => 'set',
        get_meta  => 'get',
        meta_keys => 'keys',
    },
);

# Methods
sub greet {
    my $self = shift;
    return sprintf("Hello, I'm %s, age %d", $self->name, $self->age);
}

sub describe {
    my $self = shift;
    my $email = $self->has_email ? " <" . $self->email . ">" : "";
    return $self->name . $email;
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

# =====================
# Employee extends Person
# =====================

{
package Employee;
use Moose;
extends 'Person';

has 'company' => (
    is       => 'rw',
    isa      => 'Str',
    required => 1,
);

has 'salary' => (
    is      => 'rw',
    isa     => 'Num',
    default => 0,
);

has 'department' => (
    is      => 'rw',
    isa     => 'Str',
    default => 'General',
);

# Override method
override 'greet' => sub {
    my $self = shift;
    return super() . " at " . $self->company;
};

sub give_raise {
    my ($self, $amount) = @_;
    $self->salary($self->salary + $amount);
    return $self;
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

package main;

# Create Person
my $alice = Person->new(
    name  => "Alice",
    age   => 28,
    email => "alice\@example.com",
);

printf "Name:  %s\n",  $alice->name;
printf "Age:   %d\n",  $alice->age;
printf "Email: %s\n",  $alice->email if $alice->has_email;
printf "Greet: %s\n",  $alice->greet;

# Modify
$alice->age(29);
printf "New age: %d\n", $alice->age;

# Array trait
$alice->add_tag("developer");
$alice->add_tag("perl");
$alice->add_tag("admin");

printf "Tags: %s\n", join(", ", $alice->all_tags);
printf "Tag count: %d\n", $alice->tag_count;

# Hash trait
$alice->set_meta("level", "senior");
$alice->set_meta("team",  "backend");
printf "Level: %s\n", $alice->get_meta("level");
printf "Meta keys: %s\n", join(", ", sort $alice->meta_keys);

# Clear
$alice->clear_email;
printf "Has email: %s\n", $alice->has_email ? "yes" : "no";

# Employee
my $bob = Employee->new(
    name       => "Bob",
    age        => 35,
    company    => "TechCorp",
    salary     => 72000,
    department => "Engineering",
);

printf "\n%s\n", $bob->greet;
$bob->give_raise(5000);
printf "Salary after raise: \$%d\n", $bob->salary;

# Type checking
eval { Person->new(name => "Charlie", age => "not_a_number") };
printf "Type error: %s", $@ if $@;

# Immutable
printf "\nPerson is immutable: %s\n", Person->meta->is_immutable ? "yes" : "no";
```

---

## Step 222: Moose Types

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Moose;
use Moose::Util::TypeConstraints;

# =====================
# Custom Types
# =====================

# Subtype of existing type
subtype 'PositiveInt',
    as 'Int',
    where { $_ > 0 },
    message { "Must be a positive integer, got $_" };

subtype 'Email',
    as 'Str',
    where { /\A[^@]+\@[^@]+\.[^@]+\z/ },
    message { "'$_' is not a valid email address" };

subtype 'NonEmptyStr',
    as 'Str',
    where { length($_) > 0 },
    message { "String must not be empty" };

# Enum type
enum 'Role', [qw(admin editor viewer guest)];

# Class type (auto-created for any class)
class_type 'DateTime';

# Duck type
duck_type 'Printable', ['print_info', 'to_string'];

# Coercions
coerce 'Email',
    from 'Str',
    via  { lc $_ };

coerce 'PositiveInt',
    from 'Str',
    via  { abs(int($_)) || 1 };

# =====================
# User class with rich types
# =====================

{
package User;
use Moose;
use Moose::Util::TypeConstraints;

has 'id' => (
    is  => 'ro',
    isa => 'PositiveInt',
);

has 'name' => (
    is       => 'rw',
    isa      => 'NonEmptyStr',
    required => 1,
);

has 'email' => (
    is     => 'rw',
    isa    => 'Email',
    coerce => 1,    # enable coercions
);

has 'role' => (
    is      => 'rw',
    isa     => 'Role',
    default => 'viewer',
);

has 'score' => (
    is      => 'rw',
    isa     => 'PositiveInt',
    coerce  => 1,
    default => 1,
);

has 'permissions' => (
    is      => 'ro',
    isa     => 'ArrayRef[Str]',
    lazy    => 1,
    builder => '_build_permissions',
);

sub _build_permissions {
    my $self = shift;
    my %perm_map = (
        admin  => [qw(read write delete admin)],
        editor => [qw(read write)],
        viewer => [qw(read)],
        guest  => [],
    );
    return $perm_map{$self->role} // [];
}

sub can_do {
    my ($self, $action) = @_;
    return !!grep { $_ eq $action } @{$self->permissions};
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

package main;

# Valid user
my $admin = User->new(
    id    => 1,
    name  => "Alice",
    email => "ALICE\@EXAMPLE.COM",   # will be coerced to lowercase
    role  => "admin",
    score => "-5",   # will be coerced to abs(int) = 5
);

printf "Email (coerced): %s\n", $admin->email;   # alice@example.com
printf "Score (coerced): %d\n", $admin->score;    # 5
printf "Role: %s\n",            $admin->role;
printf "Permissions: %s\n",     join(", ", @{$admin->permissions});
printf "Can delete: %s\n",      $admin->can_do("delete") ? "yes" : "no";
printf "Can admin:  %s\n",      $admin->can_do("admin")  ? "yes" : "no";

# Type errors
for my $case (
    [id => -5,  name => "Test", email => "t\@t.com"],
    [id => 1,   name => "",     email => "t\@t.com"],
    [id => 1,   name => "Test", email => "not-email"],
    [id => 1,   name => "Test", email => "t\@t.com", role => "superuser"],
) {
    eval { User->new(@$case) };
    printf "Error (expected): %s\n", (split /\n/, $@)[0] if $@;
}
```

---

## Step 223: Moose Roles

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Moose Roles
# =====================

{
package Role::Printable;
use Moose::Role;

requires 'to_string';

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
}

{
package Role::Serializable;
use Moose::Role;

requires 'to_hash';

sub to_json {
    my $self = shift;
    require JSON::PP;
    return JSON::PP->new->utf8->canonical->encode($self->to_hash);
}

sub to_yaml {
    my $self = shift;
    my %data = $self->to_hash;
    return join("\n", map { "$_: $data{$_}" } sort keys %data);
}
}

{
package Role::Timestamped;
use Moose::Role;
use Moose::Util::TypeConstraints;

has 'created_at' => (
    is      => 'ro',
    isa     => 'Int',
    default => sub { time() },
);

has 'updated_at' => (
    is        => 'rw',
    isa       => 'Maybe[Int]',
    predicate => 'has_updated_at',
);

sub touch {
    my $self = shift;
    $self->updated_at(time());
    return $self;
}

sub age_seconds {
    my $self = shift;
    return time() - $self->created_at;
}
}

{
package Role::Validatable;
use Moose::Role;

requires 'validate';

sub is_valid {
    my $self = shift;
    my @errors = $self->validate;
    return !@errors;
}

sub validation_errors {
    my $self = shift;
    return $self->validate;
}

sub assert_valid {
    my $self = shift;
    my @errors = $self->validate;
    die "Validation failed:\n" . join("\n", map { "  - $_" } @errors) . "\n"
        if @errors;
    return $self;
}
}

# =====================
# Article class with multiple roles
# =====================

{
package Article;
use Moose;
with 'Role::Printable', 'Role::Serializable', 'Role::Timestamped', 'Role::Validatable';

has 'id'      => (is => 'ro', isa => 'Int');
has 'title'   => (is => 'rw', isa => 'Str', required => 1);
has 'content' => (is => 'rw', isa => 'Str', default => '');
has 'author'  => (is => 'rw', isa => 'Str', required => 1);
has 'status'  => (is => 'rw', isa => 'Str', default => 'draft');

sub to_string {
    my $self = shift;
    return sprintf('"%s" by %s [%s]', $self->title, $self->author, $self->status);
}

sub to_hash {
    my $self = shift;
    return (
        id      => $self->id // 0,
        title   => $self->title,
        content => substr($self->content, 0, 50),
        author  => $self->author,
        status  => $self->status,
    );
}

sub validate {
    my $self = shift;
    my @errors;
    push @errors, "Title is required"         unless length($self->title) > 0;
    push @errors, "Title too short (min 5)"   unless length($self->title) >= 5;
    push @errors, "Author is required"        unless length($self->author) > 0;
    push @errors, "Content cannot be empty"   unless length($self->content) > 0;
    return @errors;
}

sub publish {
    my $self = shift;
    $self->assert_valid;
    $self->status('published');
    $self->touch;
    return $self;
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

package main;

my $article = Article->new(
    id      => 1,
    title   => "Learning Perl Moose",
    content => "Moose is a postmodern object system for Perl 5 that provides a powerful OO system.",
    author  => "Alice",
);

$article->print_info;
$article->print_verbose;

printf "\nJSON: %s\n", $article->to_json;
printf "YAML:\n%s\n", $article->to_yaml;

printf "\nIs valid: %s\n", $article->is_valid ? "yes" : "no";

eval { $article->publish };
printf "Publish error: %@\n" if $@;

$article->status("published");
$article->touch;
printf "Status: %s\n", $article->status;
printf "Updated: %s\n", $article->has_updated_at ? "yes" : "no";

# Invalid article
my $bad = Article->new(title => "Hi", content => "", author => "");
printf "\nBad article errors:\n";
printf "  - %s\n", $_ for $bad->validation_errors;

# Role checking
printf "\nRole checks:\n";
printf "  Does Printable:     %s\n", Article->does('Role::Printable') ? "yes" : "no";
printf "  Does Serializable:  %s\n", Article->does('Role::Serializable') ? "yes" : "no";
printf "  Does Timestamped:   %s\n", Article->does('Role::Timestamped') ? "yes" : "no";
```

---

## Step 224: Moose Method Modifiers

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Method modifiers
# =====================

{
package Service::Base;
use Moose;

has 'name' => (is => 'ro', isa => 'Str', required => 1);
has '_log' => (is => 'rw', isa => 'ArrayRef', default => sub { [] });

sub process {
    my ($self, $data) = @_;
    return { result => "processed: $data", service => $self->name };
}

sub log  { push @{$_[0]->{_log}}, $_[1] }
sub logs { @{$_[0]->{_log}} }

no Moose;
__PACKAGE__->meta->make_immutable(inline_constructor => 0);
}

{
package Service::Logged;
use Moose;
extends 'Service::Base';

# before — runs before the original method
before 'process' => sub {
    my ($self, $data) = @_;
    $self->log("[BEFORE] Starting process with: $data");
    printf "  [before] process called\n";
};

# after — runs after the original method
after 'process' => sub {
    my ($self, $data) = @_;
    $self->log("[AFTER] Process complete");
    printf "  [after] process returned\n";
};

no Moose;
__PACKAGE__->meta->make_immutable(inline_constructor => 0);
}

{
package Service::Validated;
use Moose;
extends 'Service::Logged';

# around — wraps the method, can modify args/return
around 'process' => sub {
    my ($orig, $self, $data) = @_;
    
    # Pre-processing
    printf "  [around] validating data\n";
    unless (defined $data && length($data) > 0) {
        return { error => "Data cannot be empty" };
    }
    
    # Call original
    my $result = $self->$orig($data);
    
    # Post-processing
    $result->{processed_at} = time();
    printf "  [around] enriching result\n";
    
    return $result;
};

no Moose;
__PACKAGE__->meta->make_immutable(inline_constructor => 0);
}

{
package Service::Cached;
use Moose;
extends 'Service::Validated';

has '_cache' => (is => 'rw', isa => 'HashRef', default => sub { {} });

# Override with super
around 'process' => sub {
    my ($orig, $self, $data) = @_;
    
    if (exists $self->_cache->{$data}) {
        printf "  [cache] HIT for '$data'\n";
        return $self->_cache->{$data};
    }
    
    printf "  [cache] MISS for '$data'\n";
    my $result = $self->$orig($data);
    $self->_cache->{$data} = $result;
    return $result;
};

no Moose;
__PACKAGE__->meta->make_immutable(inline_constructor => 0);
}

package main;

printf "=== Method Modifiers ===\n\n";

my $svc = Service::Cached->new(name => "MyService");

printf "--- First call: 'hello' ---\n";
my $r1 = $svc->process("hello");
printf "Result: %s (service: %s)\n", $r1->{result}, $r1->{service};

printf "\n--- Second call: 'hello' (cached) ---\n";
my $r2 = $svc->process("hello");
printf "Result: %s\n", $r2->{result};

printf "\n--- Empty data ---\n";
my $r3 = $svc->process("");
printf "Error: %s\n", $r3->{error};

printf "\nLogs:\n";
printf "  %s\n", $_ for $svc->logs;
```

---

## Step 225: MooseX Extensions

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Simulating MooseX-like extensions
# (without requiring additional CPAN)
# =====================

{
package MyMoose::Singleton;
use Moose ();
use Moose::Exporter;

Moose::Exporter->setup_import_methods(
    also => 'Moose',
);

sub init_meta {
    my ($class, %opts) = @_;
    my $meta = Moose->init_meta(%opts);
    
    my $target = $opts{for_class};
    
    # Install singleton pattern
    my $instance;
    
    no strict 'refs';
    *{"${target}::instance"} = sub {
        my $class = shift;
        unless ($instance) {
            $instance = $class->new(@_);
        }
        return $instance;
    };
    
    *{"${target}::reset_instance"} = sub {
        undef $instance;
    };
    
    return $meta;
}
}

# =====================
# Immutable value objects
# =====================

{
package ValueObject;
use Moose;

sub new {
    my ($class, %args) = @_;
    my $self = $class->SUPER::new(%args);
    
    # Make all attributes read-only after construction
    for my $attr ($self->meta->get_all_attributes) {
        next unless $attr->has_write_method;
        # Freeze after init
    }
    
    return $self;
}

sub equals {
    my ($self, $other) = @_;
    return 0 unless ref($other) eq ref($self);
    
    for my $attr ($self->meta->get_all_attributes) {
        my $name = $attr->name;
        my $v1 = $self->$name  // '';
        my $v2 = $other->$name // '';
        return 0 unless "$v1" eq "$v2";
    }
    
    return 1;
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

{
package Money;
use Moose;
extends 'ValueObject';
use Moose::Util::TypeConstraints;

subtype 'Currency',
    as 'Str',
    where { /^[A-Z]{3}$/ };

has 'amount'   => (is => 'ro', isa => 'Num',      required => 1);
has 'currency' => (is => 'ro', isa => 'Currency',  required => 1);

sub add {
    my ($self, $other) = @_;
    die "Currency mismatch\n" unless $self->currency eq $other->currency;
    return Money->new(amount => $self->amount + $other->amount, currency => $self->currency);
}

sub subtract {
    my ($self, $other) = @_;
    die "Currency mismatch\n" unless $self->currency eq $other->currency;
    return Money->new(amount => $self->amount - $other->amount, currency => $self->currency);
}

sub multiply {
    my ($self, $factor) = @_;
    return Money->new(amount => $self->amount * $factor, currency => $self->currency);
}

sub to_string { sprintf "%.2f %s", $_[0]->amount, $_[0]->currency }
use overload '""' => \&to_string, fallback => 1;

no Moose;
__PACKAGE__->meta->make_immutable;
}

{
package Address;
use Moose;
extends 'ValueObject';

has 'street'  => (is => 'ro', isa => 'Str', required => 1);
has 'city'    => (is => 'ro', isa => 'Str', required => 1);
has 'country' => (is => 'ro', isa => 'Str', required => 1);
has 'zip'     => (is => 'ro', isa => 'Str');

sub to_string {
    my $self = shift;
    return sprintf "%s, %s %s, %s", $self->street, $self->city, $self->zip//"", $self->country;
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

# =====================
# Aggregate root
# =====================

{
package Order;
use Moose;

has 'id'       => (is => 'ro', isa => 'Int');
has 'customer' => (is => 'ro', isa => 'Str', required => 1);
has 'address'  => (is => 'ro', isa => 'Address', required => 1);
has 'items'    => (
    is      => 'ro',
    isa     => 'ArrayRef',
    default => sub { [] },
    traits  => ['Array'],
    handles => {
        add_item   => 'push',
        all_items  => 'elements',
        item_count => 'count',
    },
);
has 'status' => (
    is      => 'rw',
    isa     => 'Str',
    default => 'pending',
);

sub total {
    my $self = shift;
    my $total = Money->new(amount => 0, currency => "USD");
    for my $item ($self->all_items) {
        $total = $total->add($item->{price}->multiply($item->{qty}));
    }
    return $total;
}

sub add_product {
    my ($self, $name, $price, $qty) = @_;
    $self->add_item({ name => $name, price => $price, qty => $qty });
    return $self;
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

package main;

# Money value objects
my $price1 = Money->new(amount => 29.99, currency => "USD");
my $price2 = Money->new(amount => 9.99,  currency => "USD");
my $total  = $price1->add($price2);

printf "Price 1: %s\n", $price1;
printf "Price 2: %s\n", $price2;
printf "Total:   %s\n", $total;
printf "Doubled: %s\n", $price1->multiply(2);

# Equality
my $p3 = Money->new(amount => 29.99, currency => "USD");
printf "\np1 == p3: %s\n", $price1->equals($p3) ? "yes" : "no";
printf "p1 == p2: %s\n",   $price1->equals($price2) ? "yes" : "no";

# Address
my $addr = Address->new(
    street  => "123 Sukhumvit Rd",
    city    => "Bangkok",
    country => "Thailand",
    zip     => "10110",
);
printf "\nAddress: %s\n", $addr->to_string;

# Order aggregate
my $order = Order->new(
    id       => 1001,
    customer => "Alice",
    address  => $addr,
);

$order->add_product("Learning Perl", Money->new(amount => 39.99, currency => "USD"), 2);
$order->add_product("CGI Guide",     Money->new(amount => 34.99, currency => "USD"), 1);

printf "\nOrder #%d for %s\n", $order->id, $order->customer;
printf "Items: %d\n", $order->item_count;
printf "Total: %s\n", $order->total;

$order->status("confirmed");
printf "Status: %s\n", $order->status;
```

---

## Step 226: Moose Traits

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Moose Array trait handles
# =====================

{
package Cart;
use Moose;

has 'items' => (
    is      => 'rw',
    isa     => 'ArrayRef[HashRef]',
    default => sub { [] },
    traits  => ['Array'],
    handles => {
        add_item      => 'push',
        remove_item   => 'delete',   # by index
        all_items     => 'elements',
        item_count    => 'count',
        clear_items   => 'clear',
        has_items     => 'count',
        get_item      => 'get',
        sort_items    => 'sort_in_place',
        filter_items  => 'grep',
        map_items     => 'map',
        find_item     => 'first',
        item_at       => 'get',
    }
);

sub total {
    my $self = shift;
    my $sum = 0;
    $sum += $_->{price} * $_->{qty} for $self->all_items;
    return $sum;
}

sub add_product {
    my ($self, %args) = @_;
    # Check if already in cart
    my $existing = $self->find_item(sub { $_->{sku} eq $args{sku} });
    if ($existing) {
        $existing->{qty} += $args{qty} // 1;
    } else {
        $self->add_item({ sku => $args{sku}, name => $args{name}, price => $args{price}, qty => $args{qty}//1 });
    }
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

# =====================
# Moose Hash trait
# =====================

{
package Config;
use Moose;

has '_store' => (
    is      => 'rw',
    isa     => 'HashRef',
    default => sub { {} },
    traits  => ['Hash'],
    handles => {
        set     => 'set',
        get     => 'get',
        has_key => 'exists',
        delete  => 'delete',
        keys    => 'keys',
        values  => 'values',
        pairs   => 'kv',
        clear   => 'clear',
        count   => 'count',
    },
);

sub load_defaults {
    my $self = shift;
    $self->set('debug',   0);
    $self->set('timeout', 30);
    $self->set('version', '1.0');
    return $self;
}

sub to_string {
    my $self = shift;
    return join(", ", map { "$_->[0]=$_->[1]" } $self->pairs);
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

# =====================
# Moose Number trait
# =====================

{
package Counter;
use Moose;

has 'count' => (
    is      => 'rw',
    isa     => 'Int',
    default => 0,
    traits  => ['Counter'],
    handles => {
        increment  => 'inc',
        decrement  => 'dec',
        reset      => 'reset',
    }
);

has 'name' => (is => 'ro', isa => 'Str', default => 'Counter');

sub status {
    my $self = shift;
    return sprintf("%s: %d", $self->name, $self->count);
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

package main;

# Cart demo
my $cart = Cart->new;

$cart->add_product(sku => "SKU001", name => "Learning Perl",    price => 39.99, qty => 1);
$cart->add_product(sku => "SKU002", name => "Programming Perl", price => 54.99, qty => 1);
$cart->add_product(sku => "SKU001", name => "Learning Perl",    price => 39.99, qty => 1);  # duplicate

printf "Cart items: %d\n", $cart->item_count;
printf "Cart total: \$%.2f\n", $cart->total;

for my $item ($cart->all_items) {
    printf "  %s x%d = \$%.2f\n", $item->{name}, $item->{qty}, $item->{price}*$item->{qty};
}

# Sorted by price
$cart->sort_items(sub { $_[0]->{price} <=> $_[1]->{price} });
printf "\nSorted by price:\n";
for my $item ($cart->all_items) {
    printf "  %s: \$%.2f\n", $item->{name}, $item->{price};
}

# Config demo
my $cfg = Config->new;
$cfg->load_defaults;
$cfg->set('app_name', 'MyApp');
$cfg->set('db_host',  'localhost');

printf "\nConfig: %s\n", $cfg->to_string;
printf "Has debug: %s\n", $cfg->has_key('debug') ? "yes" : "no";
printf "Debug: %s\n",     $cfg->get('debug');

$cfg->delete('debug');
printf "After delete: %s\n", $cfg->has_key('debug') ? "yes" : "no";

# Counter demo
my $hits = Counter->new(name => "Page Hits");
$hits->increment for 1..10;
$hits->decrement for 1..3;
printf "\n%s\n", $hits->status;

$hits->reset;
printf "After reset: %s\n", $hits->status;
```

---

## Step 227: Moose Meta API

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Moose Meta API — reflection
# =====================

{
package Product;
use Moose;
use Moose::Util::TypeConstraints;

subtype 'PositiveNum', as 'Num', where { $_ >= 0 };

has 'id'       => (is => 'ro', isa => 'Int');
has 'name'     => (is => 'rw', isa => 'Str', required => 1);
has 'price'    => (is => 'rw', isa => 'PositiveNum', default => 0);
has 'stock'    => (is => 'rw', isa => 'Int', default => 0);
has 'category' => (is => 'rw', isa => 'Str', default => 'general');
has 'active'   => (is => 'rw', isa => 'Bool', default => 1);

sub display { sprintf "%s (%s) \$%.2f", $_[0]->name, $_[0]->category, $_[0]->price }

no Moose;
__PACKAGE__->meta->make_immutable;
}

package main;

# Get meta object
my $meta = Product->meta;

printf "Class: %s\n", $meta->name;
printf "Superclasses: %s\n", join(", ", $meta->superclasses) || "(none)";
printf "Is immutable: %s\n", $meta->is_immutable ? "yes" : "no";

# List all attributes
printf "\nAttributes:\n";
for my $attr (sort { $a->name cmp $b->name } $meta->get_all_attributes) {
    my $type    = $attr->type_constraint ? $attr->type_constraint->name : "Any";
    my $ro_rw   = $attr->is_required ? "required" : $attr->has_default ? "default" : "optional";
    my $access  = $attr->is_lazy ? "lazy" : "eager";
    printf "  %-12s %-15s %-10s %s\n", $attr->name, $type, $ro_rw, $access;
}

# List methods
printf "\nMethods:\n";
for my $method (sort { $a->name cmp $b->name } $meta->get_all_methods) {
    next if $method->name =~ /^(new|meta|DEMOLISH|BUILDARGS|DESTROY|can|isa|VERSION|import|unimport|dump|Dumper)$/;
    printf "  %s\n", $method->name;
}

# Dynamic introspection
my $product = Product->new(id => 1, name => "Widget", price => 9.99, stock => 100);

printf "\nDynamic attribute access:\n";
for my $attr_name (qw(name price stock active)) {
    my $attr = $meta->get_attribute($attr_name);
    my $val  = $attr->get_value($product);
    printf "  %s = %s\n", $attr_name, $val;
}

# Dynamic modification (without make_immutable)
{
    package DynamicClass;
    use Moose;
    has 'x' => (is => 'rw', isa => 'Int', default => 0);
}

DynamicClass->meta->add_attribute('y' => (
    is      => 'rw',
    isa     => 'Int',
    default => 0,
));

DynamicClass->meta->add_method('sum' => sub {
    my $self = shift;
    return $self->x + $self->y;
});

my $obj = DynamicClass->new(x => 3, y => 4);
printf "\nDynamic class: x=%d y=%d sum=%d\n", $obj->x, $obj->y, $obj->sum;

# Serialization via meta
sub serialize {
    my $obj  = shift;
    my $meta = $obj->meta;
    my %data;
    
    for my $attr ($meta->get_all_attributes) {
        my $name = $attr->name;
        next if $name =~ /^_/;
        $data{$name} = $attr->get_value($obj);
    }
    
    return %data;
}

my %serialized = serialize($product);
printf "\nSerialized:\n";
printf "  %s: %s\n", $_, $serialized{$_}//"undef" for sort keys %serialized;
```

---

## Step 228: Lazy Attributes

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Lazy attributes — computed on first access
# =====================

{
package Report;
use Moose;

has 'raw_data' => (is => 'ro', isa => 'ArrayRef', required => 1);
has 'title'    => (is => 'ro', isa => 'Str', default => 'Report');

# Lazy computed attributes
has 'processed_data' => (
    is      => 'ro',
    isa     => 'ArrayRef',
    lazy    => 1,
    builder => '_build_processed_data',
);

has 'statistics' => (
    is      => 'ro',
    isa     => 'HashRef',
    lazy    => 1,
    builder => '_build_statistics',
);

has 'summary' => (
    is      => 'ro',
    isa     => 'Str',
    lazy    => 1,
    builder => '_build_summary',
);

# Builders (called only when accessed)
sub _build_processed_data {
    my $self = shift;
    printf "  [building processed_data]\n";
    return [
        map { { %$_, total => $_->{qty} * $_->{price} } }
        @{$self->raw_data}
    ];
}

sub _build_statistics {
    my $self = shift;
    printf "  [building statistics]\n";
    my @data = @{$self->processed_data};   # reuses cached processed_data
    
    use List::Util qw(sum min max);
    my @totals = map { $_->{total} } @data;
    
    return {
        count   => scalar @data,
        sum     => sum(@totals),
        average => sum(@totals) / @totals,
        min     => min(@totals),
        max     => max(@totals),
    };
}

sub _build_summary {
    my $self = shift;
    printf "  [building summary]\n";
    my $stats = $self->statistics;   # reuses cached statistics
    return sprintf "%s: %d items, total \$%.2f",
        $self->title, $stats->{count}, $stats->{sum};
}

sub clear_cache {
    my $self = shift;
    # Clear lazy attrs to force rebuild
    $self->clear_processed_data if $self->can('clear_processed_data');
    $self->clear_statistics      if $self->can('clear_statistics');
    $self->clear_summary         if $self->can('clear_summary');
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

package main;

my @data = (
    { name => "Widget A", qty => 10, price => 9.99  },
    { name => "Widget B", qty => 5,  price => 14.99  },
    { name => "Gadget X", qty => 3,  price => 24.99  },
);

my $report = Report->new(title => "Sales Report", raw_data => \@data);

printf "Accessing report (lazy builds happen on first access):\n";
printf "\nFirst access to summary:\n";
printf "  %s\n", $report->summary;

printf "\nSecond access (already built, no rebuild):\n";
printf "  %s\n", $report->summary;

printf "\nStatistics (already partly built):\n";
my $stats = $report->statistics;
printf "  Count: %d\n",   $stats->{count};
printf "  Sum:   \$%.2f\n", $stats->{sum};
printf "  Avg:   \$%.2f\n", $stats->{average};

# =====================
# Lazy with trigger
# =====================

{
package User;
use Moose;

has 'email' => (
    is      => 'rw',
    isa     => 'Str',
    trigger => sub {
        my ($self, $new, $old) = @_;
        printf "  [trigger] email changed from '%s' to '%s'\n",
            $old//"(none)", $new;
    },
);

has 'domain' => (
    is      => 'ro',
    isa     => 'Maybe[Str]',
    lazy    => 1,
    default => sub {
        my $self = shift;
        my $email = $self->email // return undef;
        return (split /@/, $email)[1];
    },
);

no Moose;
__PACKAGE__->meta->make_immutable;
}

package main;

printf "\nTrigger demo:\n";
my $user = User->new(email => "alice\@example.com");
printf "Domain: %s\n", $user->domain;

$user->email("bob\@company.org");

print "\nDone.\n";
```

---

## Step 229: Moose Best Practices

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Best practices demonstration
# =====================

# 1. Always make_immutable
# 2. Use coerce for flexible input
# 3. Use builders for complex defaults
# 4. Use predicate/clearer for optional attrs
# 5. Use roles for shared behavior
# 6. Type hierarchies

{
package Role::HasId;
use Moose::Role;

has 'id' => (
    is        => 'ro',
    isa       => 'Int',
    lazy      => 1,
    builder   => '_build_id',
    predicate => 'has_id',
);

my $counter = 0;
sub _build_id { ++$counter }
}

{
package Role::HasTimestamps;
use Moose::Role;

has 'created_at' => (
    is      => 'ro',
    isa     => 'Int',
    default => sub { time() },
);

has 'updated_at' => (
    is        => 'rw',
    isa       => 'Maybe[Int]',
    predicate => 'was_updated',
);

sub touch  { $_[0]->updated_at(time()) }
}

{
package Role::HasSlug;
use Moose::Role;

requires 'name';

has 'slug' => (
    is      => 'ro',
    isa     => 'Str',
    lazy    => 1,
    builder => '_build_slug',
);

sub _build_slug {
    my $name = lc $_[0]->name;
    $name =~ s/[^a-z0-9]+/-/g;
    $name =~ s/^-|-$//g;
    return $name;
}
}

{
package Category;
use Moose;
use Moose::Util::TypeConstraints;
with 'Role::HasId', 'Role::HasTimestamps', 'Role::HasSlug';

has 'name'        => (is => 'rw', isa => 'Str', required => 1);
has 'description' => (is => 'rw', isa => 'Str', default  => '');
has 'parent'      => (
    is        => 'rw',
    isa       => 'Maybe[Category]',
    predicate => 'has_parent',
    weak_ref  => 1,
);
has 'children' => (
    is      => 'rw',
    isa     => 'ArrayRef[Category]',
    default => sub { [] },
    traits  => ['Array'],
    handles => {
        add_child  => 'push',
        all_children => 'elements',
        child_count  => 'count',
    },
);

sub full_path {
    my $self = shift;
    my @path;
    my $current = $self;
    while ($current) {
        unshift @path, $current->name;
        $current = $current->has_parent ? $current->parent : undef;
    }
    return join " > ", @path;
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

{
package Article;
use Moose;
use Moose::Util::TypeConstraints;
with 'Role::HasId', 'Role::HasTimestamps', 'Role::HasSlug';

subtype 'ArticleStatus', as 'Str',
    where { /^(draft|published|archived)$/ };

has 'title'       => (is => 'rw', isa => 'Str', required => 1);
has 'content'     => (is => 'rw', isa => 'Str', default => '');
has 'status'      => (is => 'rw', isa => 'ArticleStatus', default => 'draft');
has 'category'    => (is => 'rw', isa => 'Maybe[Category]');
has 'tags'        => (
    is      => 'rw',
    isa     => 'ArrayRef[Str]',
    default => sub { [] },
    traits  => ['Array'],
    handles => { add_tag => 'push', all_tags => 'elements' },
);

sub publish {
    my $self = shift;
    die "Content required\n" unless length $self->content > 0;
    $self->status('published');
    $self->touch;
}

sub word_count {
    my @words = split /\s+/, $_[0]->content;
    return scalar @words;
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

package main;

# Categories
my $tech = Category->new(name => "Technology");
my $perl = Category->new(name => "Perl Programming");

$perl->parent($tech);
$tech->add_child($perl);

printf "Category: %s\n", $tech->name;
printf "Slug:     %s\n", $tech->slug;
printf "ID:       %d\n", $tech->id;

printf "\nPerl path: %s\n", $perl->full_path;
printf "Tech children: %d\n", $tech->child_count;

# Articles
my $article = Article->new(
    title    => "Learning Moose in Perl",
    content  => "Moose is a postmodern object system for Perl 5 that provides declarative OOP.",
    category => $perl,
);

$article->add_tag("perl");
$article->add_tag("moose");
$article->add_tag("oop");

printf "\nArticle: %s\n", $article->title;
printf "Slug:     %s\n",  $article->slug;
printf "Status:   %s\n",  $article->status;
printf "Words:    %d\n",  $article->word_count;
printf "Tags:     %s\n",  join(", ", $article->all_tags);

$article->publish;
printf "Published status: %s\n", $article->status;
printf "Was updated: %s\n", $article->was_updated ? "yes" : "no";
```

---

## Step 230: โปรแกรมสรุป — Moose CMS

```perl
#!/usr/bin/perl
# moose_cms.pl — Content Management System with Moose
use strict;
use warnings;

# =====================
# Role definitions
# =====================

{
package Role::Identifiable;
use Moose::Role;
my $seq = 0;
has 'id' => (is=>'ro', isa=>'Int', default=>sub{++$seq});
}

{
package Role::Auditable;
use Moose::Role;
has 'created_at' => (is=>'ro', isa=>'Int', default=>sub{time()});
has 'updated_at' => (is=>'rw', isa=>'Maybe[Int]');
sub touch { $_[0]->updated_at(time()); $_[0] }
sub age   { time() - $_[0]->created_at }
}

# =====================
# Models
# =====================

{
package CMS::User;
use Moose;
use Moose::Util::TypeConstraints;
with 'Role::Identifiable', 'Role::Auditable';

enum 'UserRole', [qw(admin editor author viewer)];

has 'username'  => (is=>'ro', isa=>'Str', required=>1);
has 'email'     => (is=>'rw', isa=>'Str', required=>1);
has 'role'      => (is=>'rw', isa=>'UserRole', default=>'viewer');
has 'active'    => (is=>'rw', isa=>'Bool', default=>1);
has '_password' => (is=>'rw', isa=>'Str');

sub set_password {
    my ($self, $pwd) = @_;
    require Digest::SHA;
    $self->_password(Digest::SHA::sha256_hex($pwd));
}

sub check_password {
    my ($self, $pwd) = @_;
    require Digest::SHA;
    return Digest::SHA::sha256_hex($pwd) eq ($self->_password//"");
}

sub can_do {
    my ($self, $action) = @_;
    my %perms = (
        admin  => [qw(create read update delete publish manage)],
        editor => [qw(create read update publish)],
        author => [qw(create read update)],
        viewer => [qw(read)],
    );
    return !!grep { $_ eq $action } @{$perms{$self->role}//[]};
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

{
package CMS::Category;
use Moose;
with 'Role::Identifiable', 'Role::Auditable';

has 'name'   => (is=>'rw', isa=>'Str', required=>1);
has 'slug'   => (is=>'ro', isa=>'Str', lazy=>1, builder=>'_build_slug');
has 'parent' => (is=>'rw', isa=>'Maybe[CMS::Category]', weak_ref=>1);

sub _build_slug {
    my $s = lc $_[0]->name;
    $s =~ s/\s+/-/g;
    return $s;
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

{
package CMS::Post;
use Moose;
use Moose::Util::TypeConstraints;
with 'Role::Identifiable', 'Role::Auditable';

enum 'PostStatus', [qw(draft review published archived)];

has 'title'    => (is=>'rw', isa=>'Str', required=>1);
has 'content'  => (is=>'rw', isa=>'Str', default=>'');
has 'slug'     => (is=>'ro', isa=>'Str', lazy=>1, builder=>'_build_slug');
has 'status'   => (is=>'rw', isa=>'PostStatus', default=>'draft');
has 'author'   => (is=>'ro', isa=>'CMS::User', required=>1);
has 'category' => (is=>'rw', isa=>'Maybe[CMS::Category]');
has 'tags'     => (
    is=>'rw', isa=>'ArrayRef[Str]', default=>sub{[]},
    traits=>['Array'],
    handles => { add_tag=>'push', all_tags=>'elements', tag_count=>'count' }
);
has 'views'    => (is=>'rw', isa=>'Int', default=>0, traits=>['Counter'],
    handles => { view => 'inc' });
has 'published_at' => (is=>'rw', isa=>'Maybe[Int]', predicate=>'is_published');

sub _build_slug {
    my $s = lc $_[0]->title;
    $s =~ s/[^a-z0-9]+/-/g;
    $s =~ s/^-|-$//g;
    return $s;
}

sub word_count { scalar split /\s+/, $_[0]->content }
sub excerpt    { length($_[0]->content) > 200 ? substr($_[0]->content,0,197)."..." : $_[0]->content }

sub publish {
    my $self = shift;
    die "Need content to publish\n" unless length($self->content) > 10;
    $self->status('published');
    $self->published_at(time());
    $self->touch;
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

# =====================
# CMS Application
# =====================

{
package CMS;
use Moose;

has 'posts'      => (is=>'rw', isa=>'ArrayRef', default=>sub{[]}, traits=>['Array'],
    handles => { add_post=>'push', all_posts=>'elements', post_count=>'count' });
has 'categories' => (is=>'rw', isa=>'ArrayRef', default=>sub{[]}, traits=>['Array'],
    handles => { add_category=>'push', all_categories=>'elements' });
has 'users'      => (is=>'rw', isa=>'ArrayRef', default=>sub{[]}, traits=>['Array'],
    handles => { add_user=>'push', all_users=>'elements', user_count=>'count' });

sub find_post_by_slug {
    my ($self, $slug) = @_;
    return (grep { $_->slug eq $slug } $self->all_posts)[0];
}

sub published_posts {
    my $self = shift;
    return grep { $_->status eq 'published' } $self->all_posts;
}

sub posts_by_category {
    my ($self, $cat) = @_;
    return grep { $_->category && $_->category->id == $cat->id } $self->all_posts;
}

sub stats {
    my $self = shift;
    return {
        total_posts => $self->post_count,
        published   => scalar(grep { $_->status eq 'published' } $self->all_posts),
        drafts      => scalar(grep { $_->status eq 'draft' }     $self->all_posts),
        users       => $self->user_count,
        total_views => do { my $v=0; $v+=$_->views for $self->all_posts; $v },
    };
}

no Moose;
__PACKAGE__->meta->make_immutable;
}

# =====================
# Demo
# =====================

package main;

my $cms = CMS->new;

# Users
my $admin  = CMS::User->new(username=>"admin",  email=>"admin\@cms.com",  role=>"admin");
my $alice  = CMS::User->new(username=>"alice",  email=>"alice\@cms.com",  role=>"editor");
my $bob    = CMS::User->new(username=>"bob",    email=>"bob\@cms.com",    role=>"author");

$admin->set_password("admin123");
printf "Admin password OK: %s\n", $admin->check_password("admin123") ? "yes" : "no";
printf "Admin can publish: %s\n", $admin->can_do("publish") ? "yes" : "no";
printf "Bob can delete:    %s\n", $bob->can_do("delete") ? "yes" : "no";

$cms->add_user($_) for ($admin, $alice, $bob);

# Categories
my $tech    = CMS::Category->new(name => "Technology");
my $web_dev = CMS::Category->new(name => "Web Development");
$web_dev->parent($tech);

$cms->add_category($tech);
$cms->add_category($web_dev);

# Posts
my $post1 = CMS::Post->new(
    title    => "Getting Started with Moose",
    content  => "Moose is a postmodern object system for Perl 5. It provides a powerful set of features for object-oriented programming including attributes, roles, type constraints, and method modifiers.",
    author   => $alice,
    category => $tech,
);
$post1->add_tag("perl"); $post1->add_tag("moose"); $post1->add_tag("oop");

my $post2 = CMS::Post->new(
    title   => "CGI Programming with Perl",
    content => "CGI (Common Gateway Interface) is a standard way for web servers to interface with executable programs. Perl has excellent CGI support through the CGI.pm module.",
    author  => $bob,
    category => $web_dev,
);
$post2->add_tag("perl"); $post2->add_tag("cgi"); $post2->add_tag("web");

$post1->publish;
$post2->publish;

$cms->add_post($post1);
$cms->add_post($post2);

# Simulate views
$post1->view for 1..42;
$post2->view for 1..18;

# Stats
my $stats = $cms->stats;
printf "\n=== CMS Statistics ===\n";
printf "Total posts:  %d\n", $stats->{total_posts};
printf "Published:    %d\n", $stats->{published};
printf "Total views:  %d\n", $stats->{total_views};
printf "Users:        %d\n", $stats->{users};

printf "\n=== Published Posts ===\n";
for my $post ($cms->published_posts) {
    printf "  [%s] %s by %s (%d views, %d words)\n",
        $post->slug, $post->title, $post->author->username,
        $post->views, $post->word_count;
    printf "    Tags: %s\n", join(", ", $post->all_tags);
}

printf "\n=== Technology Category ===\n";
for my $post ($cms->posts_by_category($tech)) {
    printf "  - %s\n", $post->title;
}

print "\nMoose CMS demo complete!\n";
```

---

## สรุป Part 23

ใน Part นี้คุณได้เรียนรู้:
- ✅ Moose has, is, isa, required, default
- ✅ Array/Hash/Counter traits with handles
- ✅ Custom types, subtypes, coercions
- ✅ Moose Roles (with, requires)
- ✅ Method modifiers (before, after, around, override)
- ✅ MooseX extensions (Singleton, value objects)
- ✅ Moose Meta API
- ✅ Lazy attributes, builders, triggers
- ✅ Best practices
- ✅ Full CMS application

**ถัดไป: [Part 24 — Testing with Perl](part_24.md)**
