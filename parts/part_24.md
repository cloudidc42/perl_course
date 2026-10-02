# Part 24: Testing with Perl
## Steps 231-240: Test::More, Test::Deep, TDD

---

## Step 231: Test::More พื้นฐาน

```perl
#!/usr/bin/perl
# t/01_basics.t
use strict;
use warnings;
use Test::More tests => 20;

# ok() — boolean test
ok(1,      "1 is true");
ok("hello","string is true");
ok(!0,     "!0 is true");

# is() — equality test (string or number)
is(2 + 2,    4,       "arithmetic works");
is("hello", "hello",  "string equality");
is(lc("ABC"), "abc",  "lc works");

# isnt() — not equal
isnt(2 + 2, 5,       "2+2 is not 5");
isnt("a",   "b",     "different strings");

# like() — regex match
like("hello world", qr/world/,   "contains 'world'");
like("test123",     qr/\d+/,     "contains digits");

# unlike() — no regex match
unlike("hello",     qr/\d+/,     "no digits in hello");

# cmp_ok() — comparison operator
cmp_ok(5, '>', 3,  "5 > 3");
cmp_ok(3, '<', 5,  "3 < 5");
cmp_ok(4, '==', 4, "4 == 4");

# is_deeply() — deep structure comparison
my @arr = (1, 2, 3);
is_deeply(\@arr, [1, 2, 3], "array deep equal");

my %hash = (a => 1, b => 2);
is_deeply(\%hash, {a=>1, b=>2}, "hash deep equal");

# defined / undef
my $x = "value";
ok(defined $x, "x is defined");
my $y;
ok(!defined $y, "y is undef");

# can_ok and isa_ok
{
    package Animal;
    sub new { bless {}, shift }
    sub speak { "..." }
}
{
    package Dog;
    our @ISA = ('Animal');
    sub speak { "Woof" }
}

my $dog = Dog->new;
isa_ok($dog, 'Dog',    "is a Dog");
isa_ok($dog, 'Animal', "is an Animal");
can_ok($dog, 'speak',  "can speak");

done_testing() unless defined &Test::More::done_testing; # already counted above
```

---

## Step 232: Test Subroutines

```perl
#!/usr/bin/perl
# lib/Math/Utils.pm
package Math::Utils;
use strict;
use warnings;

sub add      { $_[0] + $_[1] }
sub subtract { $_[0] - $_[1] }
sub multiply { $_[0] * $_[1] }

sub divide {
    my ($a, $b) = @_;
    die "Division by zero\n" if $b == 0;
    return $a / $b;
}

sub factorial {
    my $n = shift;
    die "Negative input\n" if $n < 0;
    return 1 if $n <= 1;
    return $n * factorial($n - 1);
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

sub fibonacci {
    my $n = shift;
    return $n if $n <= 1;
    my ($a, $b) = (0, 1);
    ($a, $b) = ($b, $a + $b) for 2..$n;
    return $b;
}

1;
```

```perl
#!/usr/bin/perl
# t/02_math_utils.t
use strict;
use warnings;
use Test::More;

# Inline the module for this demo
BEGIN {
    *Math::Utils::add      = sub { $_[0] + $_[1] };
    *Math::Utils::subtract = sub { $_[0] - $_[1] };
    *Math::Utils::multiply = sub { $_[0] * $_[1] };
    *Math::Utils::divide   = sub { die "Division by zero\n" if $_[1]==0; $_[0]/$_[1] };
    *Math::Utils::factorial = sub {
        my $n = shift;
        die "Negative input\n" if $n < 0;
        return 1 if $n <= 1;
        return $n * Math::Utils::factorial($n-1);
    };
    *Math::Utils::is_prime = sub {
        my $n = shift;
        return 0 if $n < 2;
        return 1 if $n == 2;
        return 0 if $n % 2 == 0;
        for (my $i=3; $i*$i<=$n; $i+=2) { return 0 if $n%$i==0 }
        return 1;
    };
    *Math::Utils::fibonacci = sub {
        my $n = shift;
        return $n if $n <= 1;
        my ($a,$b) = (0,1);
        ($a,$b) = ($b,$a+$b) for 2..$n;
        return $b;
    };
}

subtest 'add' => sub {
    is(Math::Utils::add(2, 3), 5,  "2+3=5");
    is(Math::Utils::add(0, 0), 0,  "0+0=0");
    is(Math::Utils::add(-1,1), 0,  "-1+1=0");
    is(Math::Utils::add(100,-50), 50, "100-50=50");
};

subtest 'divide' => sub {
    is(Math::Utils::divide(10, 2), 5,    "10/2=5");
    is(Math::Utils::divide(7, 2),  3.5,  "7/2=3.5");

    eval { Math::Utils::divide(1, 0) };
    like($@, qr/zero/i, "division by zero dies");
};

subtest 'factorial' => sub {
    is(Math::Utils::factorial(0), 1,   "0! = 1");
    is(Math::Utils::factorial(1), 1,   "1! = 1");
    is(Math::Utils::factorial(5), 120, "5! = 120");
    is(Math::Utils::factorial(10), 3628800, "10! correct");

    eval { Math::Utils::factorial(-1) };
    like($@, qr/negative/i, "negative input dies");
};

subtest 'is_prime' => sub {
    my @primes = (2,3,5,7,11,13,17,19,23,29);
    ok(Math::Utils::is_prime($_), "$_ is prime") for @primes;

    my @composites = (0,1,4,6,8,9,10,12,15);
    ok(!Math::Utils::is_prime($_), "$_ is not prime") for @composites;
};

subtest 'fibonacci' => sub {
    my @fibs = (0,1,1,2,3,5,8,13,21,34);
    is(Math::Utils::fibonacci($_), $fibs[$_], "fib($_) = $fibs[$_]") for 0..$#fibs;
};

done_testing;
```

---

## Step 233: Test::Exception and die Testing

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Test::More;

# Simulating Test::Exception behavior
sub lives_ok (&$) {
    my ($code, $desc) = @_;
    eval { $code->() };
    ok(!$@, $desc);
    diag("Exception: $@") if $@;
}

sub dies_ok (&$) {
    my ($code, $desc) = @_;
    eval { $code->() };
    ok(!!$@, $desc);
}

sub throws_ok (&$$) {
    my ($code, $pattern, $desc) = @_;
    eval { $code->() };
    like($@, $pattern, $desc);
}

# =====================
# Exception class
# =====================

{
package MyException;
sub new {
    my ($class, %args) = @_;
    return bless { message => $args{message}//"Error", code => $args{code}//0 }, $class;
}
sub message { $_[0]->{message} }
sub code    { $_[0]->{code} }
sub throw   { die shift->new(@_) }
sub stringify { ref($_[0]) . ": " . $_[0]->{message} }
use overload '""' => \&stringify, fallback => 1;
}

{
package NetworkError;
our @ISA = ('MyException');
}

{
package ValidationError;
our @ISA = ('MyException');
sub new {
    my ($class, %args) = @_;
    my $self = $class->SUPER::new(%args);
    $self->{field} = $args{field};
    return $self;
}
sub field { $_[0]->{field} }
}

# Functions that throw
sub connect_db {
    my $host = shift;
    NetworkError->throw(message => "Cannot connect to $host", code => 503)
        if $host eq 'bad-host';
    return { connected => 1, host => $host };
}

sub validate_age {
    my $age = shift;
    ValidationError->throw(message => "Age must be positive", field => "age")
        if $age < 0;
    ValidationError->throw(message => "Age must be under 150", field => "age")
        if $age > 150;
    return $age;
}

# Tests
subtest 'connect_db' => sub {
    lives_ok { connect_db('localhost') } "connect to localhost lives";
    dies_ok  { connect_db('bad-host') }  "connect to bad-host dies";
    throws_ok { connect_db('bad-host') } qr/Cannot connect/, "throws NetworkError";

    eval { connect_db('bad-host') };
    if (my $e = $@) {
        ok(ref($e) && $e->isa('NetworkError'), "correct exception type");
        is($e->code, 503, "correct error code");
    }
};

subtest 'validate_age' => sub {
    lives_ok { validate_age(25) }  "valid age lives";
    lives_ok { validate_age(0) }   "zero age lives";

    dies_ok  { validate_age(-1) }  "negative age dies";
    dies_ok  { validate_age(200) } "too-old age dies";

    eval { validate_age(-5) };
    if (my $e = $@) {
        ok(ref($e) && $e->isa('ValidationError'), "ValidationError thrown");
        is($e->field, "age", "correct field");
        like($e->message, qr/positive/, "correct message");
    }
};

# try/catch pattern
sub try_connect {
    my $host = shift;
    my $conn = eval { connect_db($host) };
    if (my $e = $@) {
        return { error => $e->message, code => $e->code };
    }
    return $conn;
}

subtest 'try_connect' => sub {
    my $ok  = try_connect('localhost');
    my $bad = try_connect('bad-host');

    ok($ok->{connected}, "good host connects");
    ok($bad->{error},    "bad host returns error");
    is($bad->{code}, 503, "correct code");
};

done_testing;
```

---

## Step 234: Test Fixtures and Helpers

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Test::More;

# =====================
# Test helper utilities
# =====================

sub make_user {
    my %override = @_;
    return {
        id       => $override{id}       // int(rand(10000)),
        name     => $override{name}     // "Test User",
        email    => $override{email}    // "test\@example.com",
        age      => $override{age}      // 25,
        role     => $override{role}     // "user",
        active   => $override{active}   // 1,
    };
}

sub make_product {
    my %override = @_;
    return {
        id       => $override{id}    // int(rand(10000)),
        name     => $override{name}  // "Test Product",
        price    => $override{price} // 9.99,
        stock    => $override{stock} // 10,
        category => $override{category} // "general",
    };
}

# =====================
# Order processing system to test
# =====================

sub calculate_discount {
    my ($user, $amount) = @_;
    return 0.20 if $user->{role} eq 'admin';
    return 0.15 if $user->{role} eq 'premium';
    return 0.10 if $amount >= 100;
    return 0.05 if $amount >= 50;
    return 0;
}

sub process_order {
    my ($user, $items) = @_;
    die "Inactive user\n" unless $user->{active};
    die "No items\n"      unless @$items;

    my $subtotal = 0;
    my @order_items;

    for my $item (@$items) {
        die "Product '$item->{name}' out of stock\n" unless $item->{stock} > 0;
        my $line = $item->{price} * $item->{qty};
        $subtotal += $line;
        push @order_items, { %$item, line_total => $line };
    }

    my $discount = calculate_discount($user, $subtotal);
    my $total    = $subtotal * (1 - $discount);

    return {
        user     => $user->{name},
        items    => \@order_items,
        subtotal => $subtotal,
        discount => $discount,
        total    => $total,
        count    => scalar @$items,
    };
}

# Tests
subtest 'discount rules' => sub {
    my $admin   = make_user(role => 'admin');
    my $premium = make_user(role => 'premium');
    my $regular = make_user(role => 'user');

    is(calculate_discount($admin,   50),  0.20, "admin gets 20%");
    is(calculate_discount($premium, 50),  0.15, "premium gets 15%");
    is(calculate_discount($regular, 100), 0.10, "regular gets 10% over 100");
    is(calculate_discount($regular, 50),  0.05, "regular gets 5% over 50");
    is(calculate_discount($regular, 20),  0,    "regular gets 0% under 50");
};

subtest 'process_order basic' => sub {
    my $user  = make_user();
    my @items = (
        { %{make_product(price => 10)}, qty => 2 },
        { %{make_product(price => 5)},  qty => 1 },
    );

    my $order = process_order($user, \@items);

    is($order->{subtotal}, 25,   "correct subtotal");
    is($order->{count},    2,    "correct item count");
    ok($order->{total} <= $order->{subtotal}, "total <= subtotal");
};

subtest 'process_order validation' => sub {
    my $inactive = make_user(active => 0);
    my $active   = make_user(active => 1);
    my @items    = ({ %{make_product()}, qty => 1 });

    eval { process_order($inactive, \@items) };
    like($@, qr/inactive/i, "inactive user rejected");

    eval { process_order($active, []) };
    like($@, qr/no items/i, "empty items rejected");

    my $oos = { %{make_product(stock => 0)}, qty => 1 };
    eval { process_order($active, [$oos]) };
    like($@, qr/out of stock/i, "out of stock rejected");
};

subtest 'order total calculation' => sub {
    my $premium = make_user(role => 'premium');
    my @items = ({ %{make_product(price => 100)}, qty => 1 });

    my $order = process_order($premium, \@items);
    is($order->{discount}, 0.15,   "15% discount");
    is($order->{subtotal}, 100,    "subtotal 100");
    cmp_ok($order->{total}, '==', 85, "total = 85");
};

done_testing;
```

---

## Step 235: Test Coverage and Mock

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Test::More;

# =====================
# Simple mocking without Test::Mock
# =====================

# Store original refs for restoration
my %_originals;

sub mock_function {
    my ($pkg, $name, $mock) = @_;
    no strict 'refs';
    $_originals{"${pkg}::${name}"} = \&{"${pkg}::${name}"} if defined &{"${pkg}::${name}"};
    *{"${pkg}::${name}"} = $mock;
}

sub restore_function {
    my ($pkg, $name) = @_;
    no strict 'refs';
    if (my $orig = $_originals{"${pkg}::${name}"}) {
        *{"${pkg}::${name}"} = $orig;
    }
}

# =====================
# Spy: records calls
# =====================

sub make_spy {
    my ($pkg, $name) = @_;
    my @calls;
    my $orig = do { no strict 'refs'; \&{"${pkg}::${name}"} } if do { no strict 'refs'; defined &{"${pkg}::${name}"} };

    mock_function($pkg, $name, sub {
        push @calls, { args => [@_], time => time() };
        return $orig ? $orig->(@_) : undef;
    });

    return sub { @calls };
}

# =====================
# System under test
# =====================

{
package Email;
sub send {
    my ($to, $subject, $body) = @_;
    # Would actually send email
    return { sent => 1, to => $to, subject => $subject };
}
}

{
package Logger;
my @_log;
sub log { push @_log, { level => $_[0], msg => $_[1], ts => time() } }
sub logs { @_log }
sub clear { @_log = () }
}

{
package UserService;
sub create_user {
    my ($name, $email) = @_;
    my $user = { id => int(rand(1000)+1), name => $name, email => $email, created => 1 };
    Logger::log("info", "Created user: $name");
    Email::send($email, "Welcome!", "Welcome $name to our service.");
    return $user;
}
}

# Tests with mocking
subtest 'UserService::create_user' => sub {
    Logger::clear();
    
    # Mock Email::send
    my @email_calls;
    mock_function('Email', 'send', sub {
        push @email_calls, [@_];
        return { sent => 1, mocked => 1 };
    });

    my $user = UserService::create_user("Alice", "alice\@example.com");

    # Verify user creation
    ok($user->{id},      "user has id");
    is($user->{name},    "Alice",              "correct name");
    is($user->{email},   "alice\@example.com", "correct email");
    ok($user->{created}, "created flag set");

    # Verify email was called
    is(scalar @email_calls, 1, "email sent once");
    is($email_calls[0][0], "alice\@example.com", "email to correct recipient");
    like($email_calls[0][1], qr/Welcome/, "email subject");

    # Verify logging
    my @logs = Logger::logs();
    is(scalar @logs, 1, "one log entry");
    like($logs[0]{msg}, qr/Alice/, "log mentions Alice");
    is($logs[0]{level}, "info",    "info level");

    restore_function('Email', 'send');
};

subtest 'spy demo' => sub {
    Logger::clear();

    my $get_calls = make_spy('Logger', 'log');

    UserService::create_user("Bob",   "bob\@example.com");
    UserService::create_user("Carol", "carol\@example.com");

    my @calls = $get_calls->();
    is(scalar @calls, 2, "log called twice");
    is($calls[0]{args}[0], "info", "first call info level");

    restore_function('Logger', 'log');
};

done_testing;
```

---

## Step 236: TDD — Test-Driven Development

```perl
#!/usr/bin/perl
# TDD: write tests first, then implement
use strict;
use warnings;
use Test::More;

# =====================
# Step 1: Write failing tests first
# =====================

# Tests for Stack data structure
subtest 'Stack' => sub {
    # These tests DEFINE the interface before implementation

    # Uncomment after implementing:
    # use Stack;
    
    {
        # Inline implementation (normally would be in separate file)
        package Stack;
        sub new   { bless { items => [] }, shift }
        sub push  { push @{$_[0]->{items}}, $_[1]; $_[0] }
        sub pop   { return undef unless @{$_[0]->{items}}; CORE::pop @{$_[0]->{items}} }
        sub peek  { $_[0]->{items}[-1] }
        sub size  { scalar @{$_[0]->{items}} }
        sub empty { !@{$_[0]->{items}} }
        sub clear { $_[0]->{items} = []; $_[0] }
        sub to_array { @{$_[0]->{items}} }
    }

    my $s = Stack->new;

    # Initial state
    ok($s->empty,   "new stack is empty");
    is($s->size, 0, "new stack size is 0");
    ok(!defined $s->peek, "peek empty stack is undef");
    ok(!defined $s->pop,  "pop empty stack is undef");

    # Push
    $s->push(1)->push(2)->push(3);
    is($s->size, 3,  "size after 3 pushes");
    ok(!$s->empty,   "not empty after pushes");
    is($s->peek, 3,  "peek returns top");
    is($s->size, 3,  "peek doesn't remove");

    # Pop
    is($s->pop, 3, "pop returns top");
    is($s->pop, 2, "pop returns next");
    is($s->size, 1, "size after 2 pops");

    # Clear
    $s->push(4)->push(5);
    $s->clear;
    ok($s->empty, "empty after clear");
    is($s->size, 0, "size 0 after clear");

    # LIFO order
    $s->push($_) for (1..5);
    my @popped;
    push @popped, $s->pop while !$s->empty;
    is_deeply(\@popped, [5,4,3,2,1], "LIFO order");
};

# =====================
# TDD: Queue
# =====================

subtest 'Queue' => sub {
    {
        package Queue;
        sub new     { bless { items => [] }, shift }
        sub enqueue { push @{$_[0]->{items}}, $_[1]; $_[0] }
        sub dequeue { return undef unless @{$_[0]->{items}}; shift @{$_[0]->{items}} }
        sub front   { $_[0]->{items}[0] }
        sub size    { scalar @{$_[0]->{items}} }
        sub empty   { !@{$_[0]->{items}} }
    }

    my $q = Queue->new;

    ok($q->empty,   "new queue is empty");
    is($q->size, 0, "new queue size 0");
    ok(!defined $q->dequeue, "dequeue empty returns undef");

    $q->enqueue(1)->enqueue(2)->enqueue(3);
    is($q->front, 1, "front is first added");
    is($q->dequeue, 1, "dequeue returns first");
    is($q->dequeue, 2, "dequeue FIFO order");
    is($q->size, 1, "size after dequeues");

    # FIFO order
    my $q2 = Queue->new;
    $q2->enqueue($_) for 1..5;
    my @got;
    push @got, $q2->dequeue while !$q2->empty;
    is_deeply(\@got, [1,2,3,4,5], "FIFO order");
};

# =====================
# TDD: LRU Cache
# =====================

subtest 'LRUCache' => sub {
    {
        package LRUCache;
        sub new {
            my ($class, $cap) = @_;
            bless { cap => $cap, data => {}, order => [] }, $class;
        }
        sub get {
            my ($self, $key) = @_;
            return undef unless exists $self->{data}{$key};
            $self->_touch($key);
            return $self->{data}{$key};
        }
        sub put {
            my ($self, $key, $val) = @_;
            if (exists $self->{data}{$key}) {
                $self->{data}{$key} = $val;
                $self->_touch($key);
            } else {
                if ($self->size >= $self->{cap}) {
                    my $evict = shift @{$self->{order}};
                    delete $self->{data}{$evict};
                }
                $self->{data}{$key} = $val;
                push @{$self->{order}}, $key;
            }
        }
        sub _touch {
            my ($self, $key) = @_;
            $self->{order} = [grep { $_ ne $key } @{$self->{order}}];
            push @{$self->{order}}, $key;
        }
        sub size { scalar @{$_[0]->{order}} }
        sub has  { exists $_[0]->{data}{$_[1]} }
    }

    my $cache = LRUCache->new(3);

    $cache->put(1, "one");
    $cache->put(2, "two");
    $cache->put(3, "three");

    is($cache->size, 3,     "size 3");
    is($cache->get(1), "one", "get existing key");

    $cache->put(4, "four"); # evicts LRU (key 2)
    ok(!$cache->has(2), "key 2 evicted (LRU)");
    ok($cache->has(1),  "key 1 kept (recently accessed)");
    ok($cache->has(3),  "key 3 kept");
    ok($cache->has(4),  "key 4 inserted");

    is($cache->size, 3, "still at capacity");
};

done_testing;
```

---

## Step 237: Test Organization

```perl
#!/usr/bin/perl
# t/lib/TestHelper.pm — shared test utilities
package TestHelper;
use strict;
use warnings;
use Test::More import => [];

our @EXPORT = qw(
    assert_user_valid
    assert_error
    make_test_db
    with_test_db
);

use Exporter 'import';

sub assert_user_valid {
    my ($user, $desc) = @_;
    $desc //= "user";
    Test::More::ok(defined $user->{id},    "$desc has id");
    Test::More::ok(length $user->{name},   "$desc has name");
    Test::More::like($user->{email}, qr/@/, "$desc has valid email");
}

sub assert_error {
    my ($code, $pattern, $desc) = @_;
    eval { $code->() };
    Test::More::like($@, $pattern, $desc);
}

sub make_test_db {
    require DBI;
    my $dbh = DBI->connect("dbi:SQLite::memory:", "", "",
        { RaiseError => 1, AutoCommit => 1 });
    return $dbh;
}

sub with_test_db {
    my ($schema_sql, $code) = @_;
    my $dbh = make_test_db();
    $dbh->do($_) for split /;\s*/, $schema_sql;
    $code->($dbh);
    $dbh->disconnect;
}

1;
```

```perl
#!/usr/bin/perl
# t/03_organized.t — using subtest + setup/teardown
use strict;
use warnings;
use Test::More;

# =====================
# Test class with setup/teardown
# =====================

{
    package TestCase;
    
    sub new {
        my ($class, %args) = @_;
        return bless {
            name     => $args{name} // "Test",
            setup    => $args{setup}    // sub {},
            teardown => $args{teardown} // sub {},
            tests    => [],
        }, $class;
    }
    
    sub add_test {
        my ($self, $name, $code) = @_;
        push @{$self->{tests}}, { name => $name, code => $code };
    }
    
    sub run {
        my $self = shift;
        Test::More::subtest $self->{name} => sub {
            for my $test (@{$self->{tests}}) {
                my $ctx = $self->{setup}->();
                eval { $test->{code}->($ctx) };
                my $err = $@;
                $self->{teardown}->($ctx);
                die $err if $err;
            }
        };
    }
}

# =====================
# Counter tests with setup
# =====================

{
    package Counter;
    sub new   { bless { count => $_[1]//0, name => $_[2]//"counter" }, $_[0] }
    sub inc   { $_[0]->{count}++ }
    sub dec   { $_[0]->{count}-- }
    sub reset { $_[0]->{count} = 0 }
    sub value { $_[0]->{count} }
    sub name  { $_[0]->{name} }
}

my $tc = TestCase->new(
    name    => "Counter Tests",
    setup   => sub { Counter->new(0, "test") },
    teardown => sub { },
);

$tc->add_test("initial value" => sub {
    my $c = shift;
    is($c->value, 0, "starts at 0");
});

$tc->add_test("increment" => sub {
    my $c = shift;
    $c->inc; $c->inc;
    is($c->value, 2, "value after 2 increments");
});

$tc->add_test("decrement" => sub {
    my $c = shift;
    $c->inc; $c->inc; $c->dec;
    is($c->value, 1, "value after inc+inc+dec");
});

$tc->add_test("reset" => sub {
    my $c = shift;
    $c->inc for 1..5;
    $c->reset;
    is($c->value, 0, "value after reset");
});

$tc->run;

# =====================
# Parameterized tests
# =====================

subtest 'Parameterized' => sub {
    my @cases = (
        { input => "hello",   expected => "HELLO",   desc => "lowercase" },
        { input => "WORLD",   expected => "WORLD",   desc => "uppercase stays" },
        { input => "Perl",    expected => "PERL",    desc => "mixed case" },
        { input => "",        expected => "",        desc => "empty string" },
        { input => "123",     expected => "123",     desc => "digits unchanged" },
    );

    for my $case (@cases) {
        is(uc($case->{input}), $case->{expected}, "uc: $case->{desc}");
    }
};

# =====================
# Table-driven tests
# =====================

subtest 'FizzBuzz' => sub {
    sub fizzbuzz {
        my $n = shift;
        return "FizzBuzz" if $n % 15 == 0;
        return "Fizz"     if $n % 3  == 0;
        return "Buzz"     if $n % 5  == 0;
        return "$n";
    }

    my @table = (
        [1,  "1"],        [2,  "2"],        [3,  "Fizz"],
        [4,  "4"],        [5,  "Buzz"],     [6,  "Fizz"],
        [9,  "Fizz"],     [10, "Buzz"],     [15, "FizzBuzz"],
        [30, "FizzBuzz"], [25, "Buzz"],     [33, "Fizz"],
    );

    for my $row (@table) {
        my ($n, $expected) = @$row;
        is(fizzbuzz($n), $expected, "fizzbuzz($n) = $expected");
    }
};

done_testing;
```

---

## Step 238: Integration Testing

```perl
#!/usr/bin/perl
# Integration tests with real (in-memory) database
use strict;
use warnings;
use Test::More;

BEGIN { eval { require DBI } or plan skip_all => "DBI not available" }

my $dbh = DBI->connect("dbi:SQLite::memory:", "", "", {
    RaiseError   => 1,
    AutoCommit   => 1,
    sqlite_unicode => 1,
});

# Schema
$dbh->do(q{
    CREATE TABLE users (
        id       INTEGER PRIMARY KEY AUTOINCREMENT,
        name     TEXT NOT NULL,
        email    TEXT UNIQUE NOT NULL,
        role     TEXT DEFAULT 'user',
        active   INTEGER DEFAULT 1,
        created  INTEGER DEFAULT (strftime('%s','now'))
    )
});

$dbh->do(q{
    CREATE TABLE posts (
        id         INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id    INTEGER REFERENCES users(id),
        title      TEXT NOT NULL,
        content    TEXT,
        published  INTEGER DEFAULT 0,
        created    INTEGER DEFAULT (strftime('%s','now'))
    )
});

# =====================
# DAL (Data Access Layer)
# =====================

{
    package UserDAL;
    
    sub create {
        my ($dbh, %args) = @_;
        $dbh->do("INSERT INTO users (name, email, role) VALUES (?, ?, ?)",
            undef, $args{name}, $args{email}, $args{role}//"user");
        return $dbh->last_insert_id;
    }
    
    sub find {
        my ($dbh, $id) = @_;
        return $dbh->selectrow_hashref("SELECT * FROM users WHERE id=?", undef, $id);
    }
    
    sub find_by_email {
        my ($dbh, $email) = @_;
        return $dbh->selectrow_hashref("SELECT * FROM users WHERE email=?", undef, $email);
    }
    
    sub update_role {
        my ($dbh, $id, $role) = @_;
        return $dbh->do("UPDATE users SET role=? WHERE id=?", undef, $role, $id);
    }
    
    sub deactivate {
        my ($dbh, $id) = @_;
        return $dbh->do("UPDATE users SET active=0 WHERE id=?", undef, $id);
    }
    
    sub all_active {
        my $dbh = shift;
        return @{$dbh->selectall_arrayref("SELECT * FROM users WHERE active=1", {Slice=>{}})};
    }
}

{
    package PostDAL;
    
    sub create {
        my ($dbh, %args) = @_;
        $dbh->do("INSERT INTO posts (user_id, title, content) VALUES (?, ?, ?)",
            undef, $args{user_id}, $args{title}, $args{content}//"");
        return $dbh->last_insert_id;
    }
    
    sub publish {
        my ($dbh, $id) = @_;
        return $dbh->do("UPDATE posts SET published=1 WHERE id=?", undef, $id);
    }
    
    sub by_user {
        my ($dbh, $user_id) = @_;
        return @{$dbh->selectall_arrayref(
            "SELECT * FROM posts WHERE user_id=? ORDER BY created DESC",
            {Slice=>{}}, $user_id
        )};
    }
    
    sub published {
        my $dbh = shift;
        return @{$dbh->selectall_arrayref(
            "SELECT p.*, u.name AS author FROM posts p JOIN users u ON p.user_id=u.id WHERE p.published=1",
            {Slice=>{}}
        )};
    }
}

# =====================
# Integration Tests
# =====================

subtest 'user lifecycle' => sub {
    my $id = UserDAL::create($dbh, name => "Alice", email => "alice\@test.com", role => "admin");
    ok($id > 0, "user created with id");

    my $user = UserDAL::find($dbh, $id);
    is($user->{name},  "Alice",           "correct name");
    is($user->{email}, "alice\@test.com", "correct email");
    is($user->{role},  "admin",           "correct role");
    is($user->{active}, 1,                "active by default");

    UserDAL::update_role($dbh, $id, "editor");
    my $updated = UserDAL::find($dbh, $id);
    is($updated->{role}, "editor", "role updated");

    UserDAL::deactivate($dbh, $id);
    my $deactivated = UserDAL::find($dbh, $id);
    is($deactivated->{active}, 0, "user deactivated");
};

subtest 'post lifecycle' => sub {
    my $uid = UserDAL::create($dbh, name => "Bob", email => "bob\@test.com");

    my $pid1 = PostDAL::create($dbh, user_id => $uid, title => "Draft Post",   content => "Draft...");
    my $pid2 = PostDAL::create($dbh, user_id => $uid, title => "Published Post", content => "Published!");

    PostDAL::publish($dbh, $pid2);

    my @user_posts = PostDAL::by_user($dbh, $uid);
    is(scalar @user_posts, 2, "user has 2 posts");

    my @published = PostDAL::published($dbh);
    is(scalar @published, 1, "1 published post");
    is($published[0]{title}, "Published Post", "correct published post");
    is($published[0]{author}, "Bob",           "author joined");
};

subtest 'active users' => sub {
    # Create extra users
    my $id1 = UserDAL::create($dbh, name => "Carol", email => "carol\@test.com");
    my $id2 = UserDAL::create($dbh, name => "Dave",  email => "dave\@test.com");

    UserDAL::deactivate($dbh, $id1);

    my @active = UserDAL::all_active($dbh);
    my @names  = map { $_->{name} } @active;
    ok(grep { $_ eq "Dave"  } @names, "Dave is active");
    ok(grep { $_ eq "Bob"   } @names, "Bob is active");
    ok(!grep { $_ eq "Carol" } @names, "Carol is not active");
    ok(!grep { $_ eq "Alice" } @names, "Alice is not active (deactivated)");
};

$dbh->disconnect;
done_testing;
```

---

## Step 239: Test Coverage Report

```perl
#!/usr/bin/perl
# coverage_demo.pl — manual coverage tracking
use strict;
use warnings;

# =====================
# Simple coverage tracker
# =====================

{
    package Coverage;
    my %_covered;
    my %_total;

    sub track {
        my ($pkg, $func) = @_;
        $_total{"${pkg}::${func}"} = 0;
    }

    sub hit {
        my ($pkg, $func) = @_;
        $_covered{"${pkg}::${func}"}++;
    }

    sub report {
        my @all     = sort keys %_total;
        my $total   = scalar @all;
        my $covered = scalar grep { $_covered{$_} } @all;
        my $pct     = $total ? int($covered/$total*100) : 0;

        printf "\n=== Coverage Report ===\n";
        printf "Functions: %d/%d (%d%%)\n", $covered, $total, $pct;
        printf "\nUncovered:\n";
        printf "  - %s\n", $_ for grep { !$_covered{$_} } @all;
        printf "\nCovered:\n";
        printf "  + %s (%d calls)\n", $_, $_covered{$_}/0 for grep { $_covered{$_} } @all;
    }
}

# =====================
# Instrumented module
# =====================

{
    package Calculator;
    Coverage::track('Calculator', $_) for qw(add subtract multiply divide power modulo);

    sub add      { Coverage::hit('Calculator','add');      $_[0]+$_[1] }
    sub subtract { Coverage::hit('Calculator','subtract'); $_[0]-$_[1] }
    sub multiply { Coverage::hit('Calculator','multiply'); $_[0]*$_[1] }
    sub divide   { Coverage::hit('Calculator','divide');   die "div0\n" if !$_[1]; $_[0]/$_[1] }
    sub power    { Coverage::hit('Calculator','power');    $_[0]**$_[1] }
    sub modulo   { Coverage::hit('Calculator','modulo');   $_[0]%$_[1] }
}

use Test::More;

# Partial test suite (intentionally missing some)
subtest 'Calculator (partial coverage)' => sub {
    is(Calculator::add(3,4),       7,   "add");
    is(Calculator::subtract(10,3), 7,   "subtract");
    is(Calculator::multiply(4,5),  20,  "multiply");
    is(Calculator::divide(9,3),    3,   "divide");
    # power and modulo not tested!
};

done_testing;
Coverage::report();
```

---

## Step 240: โปรแกรมสรุป — Test Suite สมบูรณ์

```perl
#!/usr/bin/perl
# t/full_suite.pl — Complete test suite for a library system
use strict;
use warnings;
use Test::More;

# =====================
# Library system under test
# =====================

{
package Library;

my $_books = {};
my $_members = {};
my $_loans = {};
my $_seq = 0;

sub reset { $_books={}; $_members={}; $_loans={}; $_seq=0 }

sub add_book {
    my (%args) = @_;
    die "Title required\n"  unless $args{title};
    die "ISBN required\n"   unless $args{isbn};
    die "ISBN exists\n"     if $_books->{$args{isbn}};
    my $id = "B" . ++$_seq;
    $_books->{$args{isbn}} = {
        id     => $id,
        isbn   => $args{isbn},
        title  => $args{title},
        author => $args{author}//"Unknown",
        copies => $args{copies}//1,
        available => $args{copies}//1,
    };
    return $_books->{$args{isbn}};
}

sub add_member {
    my (%args) = @_;
    die "Name required\n"  unless $args{name};
    die "Email required\n" unless $args{email};
    my $id = "M" . ++$_seq;
    $_members->{$id} = {
        id      => $id,
        name    => $args{name},
        email   => $args{email},
        active  => 1,
        loans   => 0,
    };
    return $_members->{$id};
}

sub checkout {
    my ($isbn, $member_id) = @_;
    my $book   = $_books->{$isbn}   or die "Book not found\n";
    my $member = $_members->{$member_id} or die "Member not found\n";
    die "Member not active\n"   unless $member->{active};
    die "No copies available\n" unless $book->{available} > 0;
    die "Loan limit reached\n"  if $member->{loans} >= 3;

    my $loan_id = "L" . ++$_seq;
    $_loans->{$loan_id} = {
        id        => $loan_id,
        isbn      => $isbn,
        member_id => $member_id,
        loaned_at => time(),
        returned  => 0,
    };
    $book->{available}--;
    $member->{loans}++;
    return $_loans->{$loan_id};
}

sub return_book {
    my $loan_id = shift;
    my $loan = $_loans->{$loan_id} or die "Loan not found\n";
    die "Already returned\n" if $loan->{returned};

    $loan->{returned} = 1;
    $loan->{returned_at} = time();
    $_books->{$loan->{isbn}}{available}++;
    $_members->{$loan->{member_id}}{loans}--;
    return $loan;
}

sub book_info    { $_books->{$_[0]} }
sub member_info  { $_members->{$_[0]} }
sub active_loans { grep { !$_->{returned} } values %$_loans }
sub book_count   { scalar keys %$_books }
sub member_count { scalar keys %$_members }
}

# =====================
# Test Suite
# =====================

Library::reset();

subtest 'add_book' => sub {
    my $book = Library::add_book(isbn=>"978-0596000271", title=>"Learning Perl", author=>"Schwartz");
    ok($book->{id},                      "book has id");
    is($book->{title},   "Learning Perl","correct title");
    is($book->{author},  "Schwartz",     "correct author");
    is($book->{copies},  1,              "default 1 copy");
    is($book->{available}, 1,            "all copies available");

    eval { Library::add_book(title=>"No ISBN") };
    like($@, qr/ISBN/, "ISBN required");

    eval { Library::add_book(isbn=>"978-0596000271", title=>"Dup") };
    like($@, qr/exists/i, "duplicate ISBN rejected");
};

subtest 'add_member' => sub {
    my $m = Library::add_member(name=>"Alice", email=>"alice\@lib.com");
    ok($m->{id},            "member has id");
    is($m->{name}, "Alice", "correct name");
    is($m->{loans}, 0,      "no loans initially");
    is($m->{active}, 1,     "active by default");

    eval { Library::add_member(email=>"x\@x.com") };
    like($@, qr/Name/, "name required");
};

subtest 'checkout and return' => sub {
    Library::reset();
    Library::add_book(isbn=>"ISBN001", title=>"Perl Cookbook", copies=>2);
    Library::add_book(isbn=>"ISBN002", title=>"Programming Perl");
    my $m = Library::add_member(name=>"Bob", email=>"bob\@lib.com");

    my $loan = Library::checkout("ISBN001", $m->{id});
    ok($loan->{id},         "loan created");
    is($loan->{isbn}, "ISBN001", "correct book");
    ok(!$loan->{returned},  "not yet returned");

    my $info = Library::book_info("ISBN001");
    is($info->{available}, 1, "available reduced");

    my $m_info = Library::member_info($m->{id});
    is($m_info->{loans}, 1, "member loan count");

    # Return
    Library::return_book($loan->{id});
    my $updated = Library::book_info("ISBN001");
    is($updated->{available}, 2, "available restored");

    my $m2 = Library::member_info($m->{id});
    is($m2->{loans}, 0, "member loan count restored");

    eval { Library::return_book($loan->{id}) };
    like($@, qr/returned/i, "double return rejected");
};

subtest 'loan limits' => sub {
    Library::reset();
    Library::add_book(isbn=>"B$_", title=>"Book $_") for 1..5;
    my $m = Library::add_member(name=>"Carol", email=>"carol\@lib.com");

    Library::checkout("B$_", $m->{id}) for 1..3;
    is(Library::member_info($m->{id})->{loans}, 3, "3 loans active");

    eval { Library::checkout("B4", $m->{id}) };
    like($@, qr/limit/i, "4th loan rejected");
};

subtest 'inactive member' => sub {
    Library::reset();
    Library::add_book(isbn=>"X1", title=>"Test");
    my $m = Library::add_member(name=>"Dave", email=>"dave\@lib.com");
    Library::member_info($m->{id})->{active} = 0;

    eval { Library::checkout("X1", $m->{id}) };
    like($@, qr/not active/i, "inactive member blocked");
};

subtest 'stats' => sub {
    Library::reset();
    Library::add_book(isbn=>"S$_", title=>"Book $_") for 1..3;
    Library::add_member(name=>"E$_", email=>"e${_}\@test.com") for 1..5;

    is(Library::book_count(),   3, "3 books");
    is(Library::member_count(), 5, "5 members");
    is(scalar Library::active_loans(), 0, "no active loans initially");
};

done_testing;
printf "\nAll tests passed! Library system fully tested.\n";
```

---

## สรุป Part 24

ใน Part นี้คุณได้เรียนรู้:
- ✅ Test::More: ok, is, isnt, like, unlike, cmp_ok, is_deeply
- ✅ isa_ok, can_ok, subtest
- ✅ Testing exceptions: die, eval, blessed errors
- ✅ Test fixtures and helper functions
- ✅ Mocking and spying
- ✅ TDD: write tests first
- ✅ Parameterized and table-driven tests
- ✅ Integration testing with DBI
- ✅ Coverage tracking
- ✅ Complete test suite

**ถัดไป: [Part 25 — Regular Expressions Advanced](part_25.md)**
