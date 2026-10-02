# Part 11: Subroutines (Functions)
## Steps 101-110: การสร้างและใช้งาน Subroutines

---

## Step 101: การประกาศ Subroutine

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# การประกาศ subroutine
# =====================

# แบบ forward declaration
sub greet;      # ประกาศล่วงหน้า

# แบบปกติ
sub say_hello {
    print "Hello, World!\n";
}

# เรียกใช้งาน
say_hello();
say_hello;      # ไม่ใส่ () ก็ได้ถ้าไม่มี argument

# =====================
# Subroutine ที่รับค่า
# =====================

sub greet_person {
    my $name = shift;       # รับค่าแรกจาก @_
    print "Hello, $name!\n";
}

greet_person("Alice");
greet_person("Bob");

# =====================
# @_ — argument list
# =====================

sub show_args {
    print "จำนวน args: ", scalar @_, "\n";
    for my $i (0..$#_) {
        print "  arg[$i] = $_[$i]\n";
    }
}

show_args(1, "hello", 3.14, [1,2,3]);

# =====================
# ใช้ shift ทีละตัว
# =====================

sub full_name {
    my $first = shift;
    my $last  = shift;
    return "$first $last";
}

print full_name("John", "Doe"), "\n";

# =====================
# รับทั้งหมดพร้อมกัน
# =====================

sub add_numbers {
    my ($a, $b, $c) = @_;
    return $a + $b + ($c // 0);
}

print add_numbers(1, 2, 3), "\n";  # 6
print add_numbers(10, 20), "\n";    # 30

# Forward declaration จาก ข้างบน
sub greet {
    print "Hi!\n";
}
```

---

## Step 102: Return Values

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Single return value
# =====================

sub square {
    my $n = shift;
    return $n ** 2;
}

my $result = square(5);
print "5^2 = $result\n";

# =====================
# Multiple return values
# =====================

sub min_max {
    my @nums = @_;
    my $min = $nums[0];
    my $max = $nums[0];
    
    for my $n (@nums) {
        $min = $n if $n < $min;
        $max = $n if $n > $max;
    }
    
    return ($min, $max);  # ส่งกลับเป็น list
}

my ($min, $max) = min_max(3, 1, 4, 1, 5, 9, 2, 6);
print "min=$min, max=$max\n";

# =====================
# Return hash
# =====================

sub get_user_info {
    my $id = shift;
    # simulate DB lookup
    my %users = (
        1 => { name => "Alice", email => "alice@example.com", age => 28 },
        2 => { name => "Bob",   email => "bob@example.com",   age => 35 },
    );
    return %{$users{$id} // {}};
}

my %info = get_user_info(1);
printf "Name: %s, Email: %s, Age: %d\n", 
    $info{name}, $info{email}, $info{age};

# =====================
# Implicit return (last evaluated)
# =====================

sub is_even {
    my $n = shift;
    $n % 2 == 0;   # ไม่มี return — ค่าสุดท้ายเป็น return value
}

print is_even(4) ? "even" : "odd", "\n";
print is_even(7) ? "even" : "odd", "\n";

# =====================
# Return reference
# =====================

sub create_person {
    my ($name, $age) = @_;
    return {
        name => $name,
        age  => $age,
        greet => sub { print "Hi, I'm $name!\n" },
    };
}

my $person = create_person("Carol", 30);
print "$person->{name} is $person->{age}\n";
$person->{greet}->();

# =====================
# Wantarray — context-aware return
# =====================

sub flexible {
    my @data = (1..5);
    
    if (wantarray) {
        return @data;            # list context
    } else {
        return scalar @data;     # scalar context
    }
}

my @list = flexible();     # list context
my $count = flexible();    # scalar context

print "List: @list\n";     # 1 2 3 4 5
print "Count: $count\n";   # 5
```

---

## Step 103: Default Arguments และ Named Parameters

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Default arguments
# =====================

sub create_greeting {
    my $name    = shift // "World";
    my $greeting = shift // "Hello";
    return "$greeting, $name!";
}

print create_greeting(), "\n";               # Hello, World!
print create_greeting("Alice"), "\n";        # Hello, Alice!
print create_greeting("Bob", "Hi"), "\n";   # Hi, Bob!

# =====================
# Named parameters (hash style)
# =====================

sub create_user {
    my %opts = @_;
    
    my $user = {
        name   => $opts{name}   // die "name required",
        email  => $opts{email}  // die "email required",
        age    => $opts{age}    // 0,
        role   => $opts{role}   // "user",
        active => $opts{active} // 1,
    };
    
    return $user;
}

my $user = create_user(
    name  => "Alice",
    email => "alice@example.com",
    age   => 28,
);
printf "%s (%s) - role: %s\n", $user->{name}, $user->{email}, $user->{role};

# =====================
# Named parameters กับ hashref
# =====================

sub connect_db {
    my ($opts) = @_;  # รับ hashref
    
    my $host   = $opts->{host}   // "localhost";
    my $port   = $opts->{port}   // 5432;
    my $dbname = $opts->{dbname} // die "dbname required";
    
    print "Connecting to $dbname @ $host:$port\n";
    return { connected => 1, dsn => "dbi:Pg:dbname=$dbname;host=$host;port=$port" };
}

my $conn = connect_db({ dbname => "myapp", host => "db.example.com" });
print "DSN: $conn->{dsn}\n";

# =====================
# Validation ด้วย Carp
# =====================

use Carp qw(croak carp confess cluck);

sub safe_divide {
    my ($num, $den) = @_;
    croak "Division by zero!" if $den == 0;
    return $num / $den;
}

my $q = eval { safe_divide(10, 2) };
print "10/2 = $q\n";

eval { safe_divide(10, 0) };
print "Error: $@\n" if $@;
```

---

## Step 104: Scope และ Lexical Variables

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# my — lexical scope
# =====================

{
    my $x = 10;
    print "inside: x=$x\n";
}
# print $x;   # Error: $x is not accessible here

# =====================
# Subroutine scope
# =====================

my $global_count = 0;

sub increment {
    $global_count++;   # ใช้ตัวแปร outer scope ได้
    my $local = "I am local";
    return $global_count;
}

print increment(), "\n";   # 1
print increment(), "\n";   # 2
print increment(), "\n";   # 3

# =====================
# Closures
# =====================

sub make_counter {
    my $count = 0;
    
    return sub {
        $count++;
        return $count;
    };
}

my $counter1 = make_counter();
my $counter2 = make_counter();

print $counter1->(), "\n";   # 1
print $counter1->(), "\n";   # 2
print $counter2->(), "\n";   # 1 (independent)
print $counter1->(), "\n";   # 3

# =====================
# Closure with parameter
# =====================

sub make_adder {
    my $n = shift;
    return sub { return $_[0] + $n };
}

my $add5  = make_adder(5);
my $add10 = make_adder(10);

print $add5->(3),  "\n";   # 8
print $add10->(3), "\n";   # 13

# =====================
# Closure สร้าง accessor
# =====================

sub make_getter_setter {
    my $value = shift;
    
    return (
        sub { $value },           # getter
        sub { $value = $_[0] },   # setter
    );
}

my ($get_name, $set_name) = make_getter_setter("Alice");
print $get_name->(), "\n";   # Alice
$set_name->("Bob");
print $get_name->(), "\n";   # Bob

# =====================
# our — package global
# =====================

our $VERSION = "1.0.0";

sub get_version { return $VERSION }
print "Version: ", get_version(), "\n";

# =====================
# local — dynamic scope
# =====================

our $separator = ", ";

sub print_list {
    print join($separator, @_), "\n";
}

print_list("a", "b", "c");   # a, b, c

{
    local $separator = " | ";
    print_list("a", "b", "c");   # a | b | c
}

print_list("a", "b", "c");   # a, b, c (restored)
```

---

## Step 105: References ใน Subroutines

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# ส่ง array reference
# =====================

sub sum_array {
    my $arr_ref = shift;
    my $total = 0;
    $total += $_ for @$arr_ref;
    return $total;
}

my @nums = (1..10);
print sum_array(\@nums), "\n";   # 55

# =====================
# ส่ง hash reference
# =====================

sub format_person {
    my $p = shift;
    return sprintf "%s <%s> age %d", $p->{name}, $p->{email}, $p->{age};
}

my %alice = (name => "Alice", email => "a@example.com", age => 28);
print format_person(\%alice), "\n";

# =====================
# ส่ง multiple refs
# =====================

sub merge_hashes {
    my @hash_refs = @_;
    my %result;
    %result = (%result, %$_) for @hash_refs;
    return %result;
}

my %defaults = (color => "blue", size => "medium", weight => 10);
my %overrides = (color => "red", weight => 20);

my %merged = merge_hashes(\%defaults, \%overrides);
printf "  %s = %s\n", $_, $merged{$_} for sort keys %merged;

# =====================
# Returning modified array
# =====================

sub filter_and_transform {
    my ($arr_ref, $filter, $transform) = @_;
    return [ map { $transform->($_) } grep { $filter->($_) } @$arr_ref ];
}

my @data = 1..20;
my $result = filter_and_transform(
    \@data,
    sub { $_[0] % 3 == 0 },      # filter: divisible by 3
    sub { $_[0] ** 2 },           # transform: square
);
print "Result: @$result\n";

# =====================
# In-place modification
# =====================

sub normalize_strings {
    my $arr_ref = shift;
    for my $s (@$arr_ref) {
        $s = lc($s);
        $s =~ s/^\s+|\s+$//g;  # trim
        $s =~ s/\s+/ /g;       # collapse spaces
    }
}

my @names = ("  ALICE  ", "BOB   ", "  charlie BROWN ");
normalize_strings(\@names);
print "$_\n" for @names;
```

---

## Step 106: Prototypes และ Signatures

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Subroutine Prototypes
# =====================

# แบบเก่า (ไม่แนะนำ แต่ต้องรู้)
sub add ($$) {
    return $_[0] + $_[1];
}

print add(3, 4), "\n";

# =====================
# Perl 5.20+ Subroutine Signatures
# =====================

use feature 'signatures';
no warnings 'experimental::signatures';

sub greet($name) {
    print "Hello, $name!\n";
}

sub add_numbers($a, $b) {
    return $a + $b;
}

sub greet_all($greeting, @names) {
    print "$greeting, $_!\n" for @names;
}

greet("Alice");
print add_numbers(10, 20), "\n";
greet_all("Hi", "Bob", "Carol", "Dave");

# =====================
# Default values ใน signatures
# =====================

sub create_point($x = 0, $y = 0) {
    return { x => $x, y => $y };
}

my $p1 = create_point();       # (0, 0)
my $p2 = create_point(3, 4);   # (3, 4)

printf "p1: (%d,%d)\n", $p1->{x}, $p1->{y};
printf "p2: (%d,%d)\n", $p2->{x}, $p2->{y};

# =====================
# Checking argument count
# =====================

sub strict_add {
    die "Need exactly 2 args" unless @_ == 2;
    return $_[0] + $_[1];
}

eval { strict_add(1, 2, 3) };
print "Error: $@\n" if $@;

# =====================
# Argument validation
# =====================

sub safe_sqrt {
    my ($n) = @_;
    die "Argument must be a number\n" unless $n =~ /^-?\d+\.?\d*$/;
    die "Cannot sqrt negative\n" if $n < 0;
    return sqrt($n);
}

for my $val (4, 9, -1, "abc") {
    my $result = eval { safe_sqrt($val) };
    if ($@) {
        print "safe_sqrt($val): ERROR: $@";
    } else {
        printf "safe_sqrt(%s) = %.4f\n", $val, $result;
    }
}
```

---

## Step 107: Higher-Order Functions

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Functions as values
# =====================

my $double = sub { $_[0] * 2 };
my $square = sub { $_[0] ** 2 };
my $negate = sub { -$_[0] };

print $double->(5), "\n";   # 10
print $square->(5), "\n";   # 25
print $negate->(5), "\n";   # -5

# =====================
# Function composition
# =====================

sub compose {
    my @fns = reverse @_;  # apply right to left
    return sub {
        my @args = @_;
        my $result;
        for my $fn (@fns) {
            @args = ($fn->(@args));
        }
        return $args[0];
    };
}

my $double_then_square = compose($square, $double);
my $square_then_double = compose($double, $square);

print $double_then_square->(3), "\n";   # (3*2)^2 = 36
print $square_then_double->(3), "\n";   # (3^2)*2 = 18

# =====================
# Currying
# =====================

sub curry {
    my $fn = shift;
    my @partial = @_;
    return sub { $fn->(@partial, @_) };
}

sub multiply { $_[0] * $_[1] }

my $triple = curry(\&multiply, 3);
my $times10 = curry(\&multiply, 10);

print $triple->(7), "\n";    # 21
print $times10->(7), "\n";   # 70

# =====================
# Map, filter, reduce ด้วย custom functions
# =====================

sub my_map {
    my ($fn, @list) = @_;
    return map { $fn->($_) } @list;
}

sub my_filter {
    my ($pred, @list) = @_;
    return grep { $pred->($_) } @list;
}

sub my_reduce {
    my ($fn, $init, @list) = @_;
    my $acc = $init;
    $acc = $fn->($acc, $_) for @list;
    return $acc;
}

my @nums = 1..10;

my @doubled   = my_map(sub { $_[0] * 2 }, @nums);
my @evens     = my_filter(sub { $_[0] % 2 == 0 }, @nums);
my $total     = my_reduce(sub { $_[0] + $_[1] }, 0, @nums);

print "doubled: @doubled\n";
print "evens:   @evens\n";
print "total:   $total\n";

# =====================
# Memoize
# =====================

sub memoize {
    my $fn = shift;
    my %cache;
    return sub {
        my $key = join(",", @_);
        unless (exists $cache{$key}) {
            $cache{$key} = $fn->(@_);
        }
        return $cache{$key};
    };
}

my $fib;
$fib = memoize(sub {
    my $n = $_[0];
    return $n if $n <= 1;
    return $fib->($n-1) + $fib->($n-2);
});

print "fib(10) = ", $fib->(10), "\n";   # 55
print "fib(20) = ", $fib->(20), "\n";   # 6765
```

---

## Step 108: Recursion

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Factorial
# =====================

sub factorial {
    my $n = shift;
    return 1 if $n <= 1;
    return $n * factorial($n - 1);
}

printf "%d! = %d\n", $_, factorial($_) for (0..10);

# =====================
# Fibonacci
# =====================

sub fib {
    my $n = shift;
    return $n if $n <= 1;
    return fib($n-1) + fib($n-2);
}

# =====================
# Tower of Hanoi
# =====================

my $moves = 0;
sub hanoi {
    my ($n, $from, $to, $via) = @_;
    if ($n == 1) {
        $moves++;
        print "Move disk 1 from $from to $to\n";
        return;
    }
    hanoi($n-1, $from, $via, $to);
    $moves++;
    print "Move disk $n from $from to $to\n";
    hanoi($n-1, $via, $to, $from);
}

hanoi(3, 'A', 'C', 'B');
print "Total moves: $moves\n";

# =====================
# Tree traversal
# =====================

sub tree_sum {
    my $node = shift;
    return 0 unless defined $node;
    
    my $sum = $node->{val};
    $sum += tree_sum($node->{left});
    $sum += tree_sum($node->{right});
    return $sum;
}

my $tree = {
    val   => 1,
    left  => { val => 2, left => { val => 4 }, right => { val => 5 } },
    right => { val => 3, left => { val => 6 }, right => { val => 7 } },
};

print "Tree sum: ", tree_sum($tree), "\n";   # 28

# =====================
# Flatten nested array
# =====================

sub flatten {
    my @result;
    for my $item (@_) {
        if (ref $item eq 'ARRAY') {
            push @result, flatten(@$item);
        } else {
            push @result, $item;
        }
    }
    return @result;
}

my @nested = (1, [2, [3, 4], 5], [6, 7], 8);
my @flat = flatten(@nested);
print "Flat: @flat\n";   # 1 2 3 4 5 6 7 8

# =====================
# Tail recursion simulation
# =====================

sub factorial_iter {
    my ($n, $acc) = @_;
    $acc //= 1;
    return $acc if $n <= 1;
    return factorial_iter($n - 1, $acc * $n);
}

printf "%d! = %d\n", 10, factorial_iter(10);
```

---

## Step 109: Error Handling ใน Subroutines

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Carp;

# =====================
# die / eval
# =====================

sub divide {
    my ($a, $b) = @_;
    die "Division by zero\n" if $b == 0;
    return $a / $b;
}

my $r = eval { divide(10, 2) };
print "10/2 = $r\n";

eval { divide(10, 0) };
print "Error: $@" if $@;

# =====================
# Error objects
# =====================

package MathError;
sub new {
    my ($class, %opts) = @_;
    return bless {
        message => $opts{message} // "Math error",
        code    => $opts{code}    // 0,
    }, $class;
}
sub message { $_[0]->{message} }
sub code    { $_[0]->{code} }
sub stringify { "MathError[$_[0]->{code}]: $_[0]->{message}" }

package main;

sub safe_sqrt {
    my $n = shift;
    if ($n < 0) {
        die MathError->new(message => "Cannot take sqrt of negative number",
                           code    => 101);
    }
    return sqrt($n);
}

my $result = eval { safe_sqrt(-4) };
if (my $err = $@) {
    if (ref $err && $err->isa('MathError')) {
        printf "Caught MathError %d: %s\n", $err->code, $err->message;
    } else {
        print "Unknown error: $err\n";
    }
}

# =====================
# Carp functions
# =====================

sub level3 { croak "Something went wrong!" }
sub level2 { level3() }
sub level1 { level2() }

eval { level1() };
print "croak: $@\n";

# confess gives full stack trace
sub deep { confess "Deep error!" }
eval { deep() };
print "confess: $@\n";

# =====================
# Return error code style
# =====================

sub read_config {
    my $file = shift;
    
    unless (-e $file) {
        return (undef, "File not found: $file");
    }
    
    open(my $fh, '<', $file) or return (undef, "Cannot open: $!");
    my %config;
    while (<$fh>) {
        next if /^\s*#/;
        if (/^(\w+)\s*=\s*(.+)$/) {
            $config{$1} = $2;
        }
    }
    close $fh;
    return (\%config, undef);
}

my ($cfg, $err) = read_config("/nonexistent.conf");
if ($err) {
    print "Config error: $err\n";
} else {
    print "Config loaded\n";
}
```

---

## Step 110: โปรแกรมสรุป — Library System

```perl
#!/usr/bin/perl
#
# library.pl — ระบบจัดการห้องสมุด
#

use strict;
use warnings;

# =====================
# Book operations
# =====================

my @books;
my $next_id = 1;

sub add_book {
    my (%opts) = @_;
    push @books, {
        id      => $next_id++,
        title   => $opts{title}   // die "title required",
        author  => $opts{author}  // die "author required",
        year    => $opts{year}    // 0,
        isbn    => $opts{isbn}    // "",
        copies  => $opts{copies}  // 1,
        checked => 0,
    };
    return $books[-1]{id};
}

sub find_by_id { (grep { $_->{id} == $_[0] } @books)[0] }
sub find_by_title {
    my $q = lc shift;
    return grep { index(lc($_->{title}), $q) >= 0 } @books;
}
sub find_by_author {
    my $q = lc shift;
    return grep { index(lc($_->{author}), $q) >= 0 } @books;
}

sub checkout_book {
    my $id = shift;
    my $book = find_by_id($id) or return (0, "Book not found");
    return (0, "No copies available") if $book->{copies} <= $book->{checked};
    $book->{checked}++;
    return (1, "Checked out: $book->{title}");
}

sub return_book {
    my $id = shift;
    my $book = find_by_id($id) or return (0, "Book not found");
    return (0, "No copies checked out") if $book->{checked} <= 0;
    $book->{checked}--;
    return (1, "Returned: $book->{title}");
}

sub display_book {
    my $b = shift;
    printf "  [%2d] %-35s %-20s %4d (avail: %d/%d)\n",
        $b->{id}, substr($b->{title},0,35), 
        substr($b->{author},0,20), $b->{year},
        $b->{copies} - $b->{checked}, $b->{copies};
}

sub list_books {
    my @list = @_ ? @_ : @books;
    print "\n";
    printf "  %-4s %-35s %-20s %-4s %-10s\n",
        "ID", "Title", "Author", "Year", "Available";
    print "  " . "-" x 75 . "\n";
    display_book($_) for @list;
    printf "\n  Total: %d books\n", scalar @list;
}

# =====================
# Statistics
# =====================

sub library_stats {
    my $total_books   = scalar @books;
    my $total_copies  = 0;
    my $total_checked = 0;
    my %by_year;
    
    for my $b (@books) {
        $total_copies  += $b->{copies};
        $total_checked += $b->{checked};
        $by_year{int($b->{year}/10)*10}++;
    }
    
    print "\n=== Library Statistics ===\n";
    printf "  Unique titles:  %d\n", $total_books;
    printf "  Total copies:   %d\n", $total_copies;
    printf "  Checked out:    %d\n", $total_checked;
    printf "  Available:      %d\n", $total_copies - $total_checked;
    
    print "\n  Books by decade:\n";
    printf "    %ds: %d books\n", $_, $by_year{$_}
        for sort keys %by_year;
}

# =====================
# Demo data
# =====================

add_book(title => "Learning Perl",          author => "Schwartz",     year => 2021, copies => 3);
add_book(title => "Programming Perl",       author => "Wall",         year => 2000, copies => 2);
add_book(title => "Modern Perl",            author => "Trout",        year => 2016, copies => 2);
add_book(title => "Perl Cookbook",          author => "Christiansen", year => 2003, copies => 1);
add_book(title => "Higher-Order Perl",      author => "Dominus",      year => 2005, copies => 2);
add_book(title => "Intermediate Perl",      author => "Schwartz",     year => 2012, copies => 3);
add_book(title => "Perl Best Practices",    author => "Conway",       year => 2005, copies => 1);

# =====================
# Run demo
# =====================

print "=== ระบบห้องสมุด Perl ===\n";
list_books();

print "\n--- Checkout books ---\n";
my ($ok, $msg);
($ok, $msg) = checkout_book(1); print "$msg\n";
($ok, $msg) = checkout_book(1); print "$msg\n";
($ok, $msg) = checkout_book(4); print "$msg\n";
($ok, $msg) = checkout_book(4); print "$msg\n";  # error: no copies

print "\n--- Search by author: Schwartz ---\n";
my @found = find_by_author("Schwartz");
list_books(@found);

print "\n--- Return book 1 ---\n";
($ok, $msg) = return_book(1); print "$msg\n";

library_stats();
```

---

## สรุป Part 11

ใน Part นี้คุณได้เรียนรู้:
- ✅ การประกาศและเรียกใช้ subroutine
- ✅ @_ และ shift/unshift
- ✅ return values (เดี่ยว, หลายค่า, reference)
- ✅ Default arguments และ Named parameters
- ✅ Lexical scope และ closures
- ✅ Higher-order functions
- ✅ Recursion
- ✅ Error handling ใน subroutines
- ✅ Prototypes และ Signatures (Perl 5.20+)

**ถัดไป: [Part 12 — Regular Expressions ขั้นสูง](part_12.md)**
