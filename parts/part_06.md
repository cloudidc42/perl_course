# Part 06: Hashes — แฮช
## Steps 51-60: การใช้งาน Hash อย่างสมบูรณ์

---

## Step 51: Hash พื้นฐาน

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# การสร้าง Hash
# =====================

# Empty hash
my %empty = ();

# Hash พื้นฐาน (key-value pairs)
my %person = (
    "name"  => "Somchai",
    "age"   => 25,
    "city"  => "Bangkok",
    "email" => "somchai@example.com",
);

# ใช้ , แทน => ก็ได้ (=> เป็น fat comma — แปลง left side เป็น string)
my %colors = (
    red   => "#FF0000",
    green => "#00FF00",
    blue  => "#0000FF",
);

# =====================
# การเข้าถึง Values
# =====================

print $person{name}, "\n";   # Somchai
print $person{age}, "\n";    # 25
print $colors{red}, "\n";    # #FF0000

# =====================
# Hash Slice
# =====================
my @some_values = @person{qw(name city)};
print "@some_values\n";  # Somchai Bangkok

# Hash slice assignment
@person{qw(phone mobile)} = ("02-111-2222", "081-333-4444");
print $person{phone}, "\n";

# =====================
# keys, values, each
# =====================

# keys — คืน list ของ keys
my @keys = keys %person;
print "Keys: @keys\n";  # ลำดับไม่แน่นอน

# values — คืน list ของ values
my @vals = values %person;
print "Values: @vals\n";

# each — iterate key-value pairs
while (my ($key, $val) = each %person) {
    print "$key: $val\n";
}

# =====================
# การแสดงผล Hash อย่างสวยงาม
# =====================
foreach my $key (sort keys %person) {
    printf "%-10s: %s\n", $key, $person{$key};
}
```

---

## Step 52: การแก้ไข Hash

```perl
#!/usr/bin/perl
use strict;
use warnings;

my %inventory = (
    apple  => 50,
    banana => 30,
    cherry => 100,
    date   => 20,
);

# =====================
# เพิ่ม / แก้ไข
# =====================

$inventory{elderberry} = 75;  # เพิ่ม key ใหม่
$inventory{apple} = 60;       # แก้ไข value

print "apple: $inventory{apple}\n";      # 60
print "elderberry: $inventory{elderberry}\n";  # 75

# =====================
# delete
# =====================

my $deleted = delete $inventory{date};
print "deleted: $deleted\n";  # 20
print exists $inventory{date} ? "still exists\n" : "deleted!\n";

# =====================
# exists vs defined
# =====================

my %data = (
    name  => "Somchai",
    score => undef,  # exists แต่ undefined
);

# exists — ตรวจว่า key มีอยู่
print exists $data{name}  ? "name exists\n"  : "no name\n";     # exists
print exists $data{score} ? "score exists\n" : "no score\n";    # exists
print exists $data{age}   ? "age exists\n"   : "no age\n";      # no age

# defined — ตรวจว่า value มีค่า
print defined $data{name}  ? "name defined\n"  : "name undef\n";   # defined
print defined $data{score} ? "score defined\n" : "score undef\n";  # undef

# =====================
# Counting with Hash
# =====================

my @words = qw(the quick brown fox jumps over the lazy dog the fox);

my %word_count;
$word_count{$_}++ for @words;

print "\nWord frequency:\n";
foreach my $word (sort { $word_count{$b} <=> $word_count{$a} || $a cmp $b } 
                  keys %word_count) {
    printf "  %-10s: %d\n", $word, $word_count{$word};
}

# =====================
# Inverting Hash
# =====================

my %lang_country = (
    Thai    => "Thailand",
    English => "UK",
    French  => "France",
    German  => "Germany",
);

# Invert key-value
my %country_lang = reverse %lang_country;

foreach my $country (sort keys %country_lang) {
    print "$country: $country_lang{$country}\n";
}
```

---

## Step 53: Hash Functions ขั้นสูง

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Merging Hashes
# =====================

my %defaults = (
    color    => "blue",
    size     => "medium",
    debug    => 0,
    timeout  => 30,
);

my %user_settings = (
    color   => "red",
    debug   => 1,
    custom  => "value",
);

# Merge: user settings override defaults
my %merged = (%defaults, %user_settings);

foreach my $key (sort keys %merged) {
    printf "%-12s: %s\n", $key, $merged{$key};
}

# =====================
# Hash Slices Advanced
# =====================

my %config = (
    host     => "localhost",
    port     => 3306,
    dbname   => "mydb",
    username => "root",
    password => "secret",
);

# Extract subset
my %db_config;
@db_config{qw(host port dbname)} = @config{qw(host port dbname)};

print "DB config:\n";
foreach my $k (sort keys %db_config) {
    print "  $k: $db_config{$k}\n";
}

# =====================
# Hash of Arrays
# =====================

my %class = (
    math    => [qw(Alice Bob Charlie)],
    science => [qw(Diana Eve Frank)],
    english => [qw(Grace Henry Iris)],
);

foreach my $subject (sort keys %class) {
    print "$subject: ", join(", ", @{$class{$subject}}), "\n";
}

# เพิ่มนักเรียน
push @{$class{math}}, "Jack";
print "math: ", join(", ", @{$class{math}}), "\n";

# =====================
# Hash of Hashes
# =====================

my %employees = (
    E001 => { name => "Alice",   dept => "IT",      salary => 80000 },
    E002 => { name => "Bob",     dept => "HR",      salary => 60000 },
    E003 => { name => "Charlie", dept => "IT",      salary => 90000 },
    E004 => { name => "Diana",   dept => "Finance", salary => 75000 },
);

# เข้าถึง
print $employees{E001}{name}, "\n";  # Alice
print $employees{E003}{salary}, "\n"; # 90000

# แก้ไข
$employees{E001}{salary} = 85000;

# วนซ้ำ
foreach my $id (sort keys %employees) {
    my $emp = $employees{$id};
    printf "%-6s %-12s %-10s %,d\n",
        $id, $emp->{name}, $emp->{dept}, $emp->{salary};
}

# =====================
# grep กับ Hash
# =====================

# กรอง hash ตาม condition
my %it_dept = map { $_ => $employees{$_} }
              grep { $employees{$_}{dept} eq "IT" }
              keys %employees;

print "\nIT Department:\n";
foreach my $id (sort keys %it_dept) {
    print "  $id: $it_dept{$id}{name}\n";
}
```

---

## Step 54: Hash Patterns ที่ใช้บ่อย

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Lookup Table
# =====================

my %roman = (
    1    => 'I',   4    => 'IV',
    5    => 'V',   9    => 'IX',
    10   => 'X',   40   => 'XL',
    50   => 'L',   90   => 'XC',
    100  => 'C',   400  => 'CD',
    500  => 'D',   900  => 'CM',
    1000 => 'M',
);

sub to_roman {
    my $num = shift;
    my $result = '';
    
    foreach my $val (reverse sort { $a <=> $b } keys %roman) {
        while ($num >= $val) {
            $result .= $roman{$val};
            $num -= $val;
        }
    }
    return $result;
}

for my $n (1, 4, 9, 14, 40, 90, 399, 1994, 2024) {
    printf "%5d = %s\n", $n, to_roman($n);
}

# =====================
# Dispatch Table (ตารางเรียกใช้งาน)
# =====================

my %commands = (
    help    => sub { print "Available commands: help, quit, version\n" },
    quit    => sub { print "Goodbye!\n"; exit },
    version => sub { print "Version 1.0.0\n" },
    
    add  => sub { print "Sum: ", $_[0] + $_[1], "\n" },
    mult => sub { print "Product: ", $_[0] * $_[1], "\n" },
);

sub dispatch {
    my ($cmd, @args) = @_;
    
    if (exists $commands{$cmd}) {
        $commands{$cmd}->(@args);
    } else {
        print "Unknown command: $cmd\n";
    }
}

dispatch("help");
dispatch("version");
dispatch("add", 3, 4);
dispatch("mult", 5, 6);
dispatch("unknown");

# =====================
# Caching (Memoization)
# =====================

my %fib_cache;

sub fibonacci {
    my $n = shift;
    return $n if $n <= 1;
    return $fib_cache{$n} if exists $fib_cache{$n};
    $fib_cache{$n} = fibonacci($n-1) + fibonacci($n-2);
    return $fib_cache{$n};
}

# ช้ามาก ถ้าไม่มี cache
print "Fibonacci(30) = ", fibonacci(30), "\n";
print "Fibonacci(40) = ", fibonacci(40), "\n";

# =====================
# Grouping
# =====================

my @people = (
    { name => "Alice",   city => "Bangkok",   age => 25 },
    { name => "Bob",     city => "Chiang Mai", age => 30 },
    { name => "Charlie", city => "Bangkok",   age => 28 },
    { name => "Diana",   city => "Phuket",    age => 22 },
    { name => "Eve",     city => "Bangkok",   age => 35 },
    { name => "Frank",   city => "Chiang Mai", age => 27 },
);

# Group by city
my %by_city;
push @{$by_city{$_->{city}}}, $_ for @people;

foreach my $city (sort keys %by_city) {
    print "\n$city:\n";
    foreach my $p (@{$by_city{$city}}) {
        printf "  %-15s age: %d\n", $p->{name}, $p->{age};
    }
}
```

---

## Step 55: Hash ในการจัดการข้อมูล

```perl
#!/usr/bin/perl
use strict;
use warnings;
use List::Util qw(sum max min);

# =====================
# Histogram
# =====================

my @scores = (75, 82, 91, 68, 85, 79, 93, 62, 87, 76,
              83, 71, 88, 95, 64, 78, 90, 73, 86, 81);

my %histogram;
for my $score (@scores) {
    my $bucket = int($score / 10) * 10;
    $histogram{"${bucket}-" . ($bucket+9)}++;
}

print "Score Distribution:\n";
foreach my $range (sort keys %histogram) {
    my $bar = "#" x ($histogram{$range} * 2);
    printf "%-8s: %-20s (%d)\n", $range, $bar, $histogram{$range};
}

# =====================
# Frequency Analysis
# =====================

sub frequency_analysis {
    my @items = @_;
    my %freq;
    $freq{$_}++ for @items;
    
    my $total = scalar @items;
    my @result;
    
    foreach my $item (sort { $freq{$b} <=> $freq{$a} || $a cmp $b } keys %freq) {
        push @result, {
            item    => $item,
            count   => $freq{$item},
            percent => $freq{$item} / $total * 100,
        };
    }
    return @result;
}

my @fruits = qw(apple banana apple cherry banana apple mango cherry banana kiwi);

print "\nFruit Frequency:\n";
printf "%-12s %6s %8s\n", "Fruit", "Count", "Percent";
print "-" x 28 . "\n";
for my $item (frequency_analysis(@fruits)) {
    printf "%-12s %6d %7.1f%%\n",
        $item->{item}, $item->{count}, $item->{percent};
}

# =====================
# Multi-index Data
# =====================

my @records = (
    { id => 1, name => "Alice",   dept => "IT",   level => "Senior" },
    { id => 2, name => "Bob",     dept => "HR",   level => "Junior" },
    { id => 3, name => "Charlie", dept => "IT",   level => "Senior" },
    { id => 4, name => "Diana",   dept => "IT",   level => "Junior" },
    { id => 5, name => "Eve",     dept => "HR",   level => "Senior" },
);

# Build multiple indexes
my %by_id   = map { $_->{id}   => $_ } @records;
my %by_name = map { $_->{name} => $_ } @records;
my %by_dept;
push @{$by_dept{$_->{dept}}}, $_ for @records;

# Lookup by id
my $emp = $by_id{3};
print "\nEmployee ID 3: $emp->{name}\n";

# Lookup by name
my $alice = $by_name{Alice};
print "Alice's dept: $alice->{dept}\n";

# All IT employees
print "\nIT Department:\n";
foreach my $e (@{$by_dept{IT}}) {
    print "  $e->{name} ($e->{level})\n";
}
```

---

## Step 56: Complex Hash Operations

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Deep copy of Hash
# =====================

use Storable qw(dclone);

my %original = (
    name    => "Config",
    options => [1, 2, 3],
    nested  => { a => 1, b => 2 },
);

# Shallow copy — nested refs ยังชี้ที่เดิม
my %shallow = %original;

# Deep copy
my %deep = %{ dclone(\%original) };

# แก้ไข deep copy ไม่กระทบ original
$deep{options}[0] = 99;
print "original options[0]: $original{options}[0]\n";  # 1 (ไม่เปลี่ยน)
print "shallow options[0]: $shallow{options}[0]\n";     # 99 (เปลี่ยน! shallow)
print "deep options[0]: $deep{options}[0]\n";           # 99

# =====================
# Hash comparison
# =====================

sub hashes_equal {
    my ($h1, $h2) = @_;
    
    return 0 unless keys %$h1 == keys %$h2;
    
    for my $key (keys %$h1) {
        return 0 unless exists $h2->{$key};
        return 0 unless ($h1->{$key} // '') eq ($h2->{$key} // '');
    }
    return 1;
}

my %h1 = (a => 1, b => 2, c => 3);
my %h2 = (a => 1, b => 2, c => 3);
my %h3 = (a => 1, b => 2, d => 3);

print hashes_equal(\%h1, \%h2) ? "equal\n" : "not equal\n";   # equal
print hashes_equal(\%h1, \%h3) ? "equal\n" : "not equal\n";   # not equal

# =====================
# Hash diff
# =====================

sub hash_diff {
    my ($old, $new) = @_;
    
    my %result;
    
    # Keys ที่เพิ่มมา
    for my $key (keys %$new) {
        unless (exists $old->{$key}) {
            $result{added}{$key} = $new->{$key};
        }
    }
    
    # Keys ที่ลบออก
    for my $key (keys %$old) {
        unless (exists $new->{$key}) {
            $result{removed}{$key} = $old->{$key};
        }
    }
    
    # Keys ที่เปลี่ยนค่า
    for my $key (keys %$old) {
        if (exists $new->{$key} && $old->{$key} ne $new->{$key}) {
            $result{changed}{$key} = { old => $old->{$key}, new => $new->{$key} };
        }
    }
    
    return %result;
}

my %v1 = (name => "Alice", age => 25, city => "Bangkok");
my %v2 = (name => "Alice", age => 26, email => "alice@example.com");

my %diff = hash_diff(\%v1, \%v2);

print "\nHash Diff:\n";
if ($diff{added}) {
    print "Added:\n";
    print "  $_ = $diff{added}{$_}\n" for keys %{$diff{added}};
}
if ($diff{removed}) {
    print "Removed:\n";
    print "  $_\n" for keys %{$diff{removed}};
}
if ($diff{changed}) {
    print "Changed:\n";
    for my $k (keys %{$diff{changed}}) {
        print "  $k: $diff{changed}{$k}{old} → $diff{changed}{$k}{new}\n";
    }
}

# =====================
# Transforming Hash
# =====================

my %prices = (
    apple  => 25,
    banana => 15,
    cherry => 80,
    date   => 120,
);

# Apply discount
my %discounted = map { $_ => int($prices{$_} * 0.9) } keys %prices;

# Filter expensive items
my %expensive = map  { $_ => $prices{$_} }
                grep { $prices{$_} > 50 }
                keys %prices;

print "\nOriginal prices:\n";
printf "  %-10s: %d\n", $_, $prices{$_} for sort keys %prices;

print "\nDiscounted (10% off):\n";
printf "  %-10s: %d\n", $_, $discounted{$_} for sort keys %discounted;

print "\nExpensive (>50):\n";
printf "  %-10s: %d\n", $_, $expensive{$_} for sort keys %expensive;
```

---

## Step 57: Hash ใน OOP Style

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Hash เป็น Object
# =====================

sub new_person {
    my (%args) = @_;
    return {
        name   => $args{name}   // "Unknown",
        age    => $args{age}    // 0,
        email  => $args{email}  // "",
        skills => $args{skills} // [],
    };
}

sub person_to_string {
    my $p = shift;
    return sprintf "Person{name=%s, age=%d, email=%s}",
        $p->{name}, $p->{age}, $p->{email};
}

sub person_add_skill {
    my ($p, $skill) = @_;
    push @{$p->{skills}}, $skill unless grep { $_ eq $skill } @{$p->{skills}};
}

# สร้าง objects
my $alice = new_person(name => "Alice", age => 30, email => "alice@example.com");
my $bob   = new_person(name => "Bob",   age => 25);

person_add_skill($alice, "Perl");
person_add_skill($alice, "Python");
person_add_skill($alice, "SQL");
person_add_skill($bob, "Java");

print person_to_string($alice), "\n";
print "Skills: ", join(", ", @{$alice->{skills}}), "\n";
print person_to_string($bob), "\n";

# =====================
# Hash เป็น Config Object
# =====================

sub load_config {
    my (%overrides) = @_;
    
    my %config = (
        # Defaults
        host     => "localhost",
        port     => 8080,
        debug    => 0,
        max_conn => 100,
        timeout  => 30,
        log_file => "/var/log/app.log",
    );
    
    # Apply overrides
    @config{keys %overrides} = values %overrides;
    
    return \%config;
}

sub validate_config {
    my $config = shift;
    my @errors;
    
    push @errors, "port must be 1-65535" 
        unless $config->{port} =~ /^\d+$/ && $config->{port} >= 1 && $config->{port} <= 65535;
    
    push @errors, "timeout must be positive"
        unless $config->{timeout} > 0;
    
    push @errors, "max_conn must be positive"
        unless $config->{max_conn} > 0;
    
    return @errors;
}

my $config = load_config(
    host  => "production.example.com",
    port  => 443,
    debug => 1,
);

my @errors = validate_config($config);
if (@errors) {
    print "Config errors:\n";
    print "  - $_\n" for @errors;
} else {
    print "Config OK:\n";
    for my $key (sort keys %$config) {
        printf "  %-12s: %s\n", $key, $config->{$key};
    }
}
```

---

## Step 58: Hash Performance

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Time::HiRes qw(time);

# =====================
# Hash vs Array Lookup
# =====================

# สร้างข้อมูลทดสอบ
my @large_array = map { "item_$_" } 1..100000;
my %large_hash  = map { "item_$_" => 1 } 1..100000;

my $target = "item_99999";

# Array linear search
my $start = time();
my $found = 0;
for (1..100) {
    for my $item (@large_array) {
        if ($item eq $target) { $found = 1; last }
    }
}
printf "Array search: %.4f sec\n", time() - $start;

# Hash lookup
$start = time();
for (1..100) {
    $found = exists $large_hash{$target} ? 1 : 0;
}
printf "Hash lookup:  %.4f sec\n", time() - $start;

# =====================
# Pre-building Hash for speed
# =====================

my @data = map { { id => $_, name => "Item $_", value => $_ * 10 } } 1..1000;

# ถ้าต้อง lookup หลายครั้ง build hash ก่อน
my %by_id = map { $_->{id} => $_ } @data;

# Lookup ครั้งเดียว O(1)
my $item = $by_id{500};
print "Found: $item->{name} = $item->{value}\n";

# =====================
# Tie Hash (advanced — persistent hash)
# =====================
# use DB_File;
# tie my %db, 'DB_File', 'data.db' or die $!;
# $db{key} = "value";
# untie %db;
```

---

## Step 59: Hash Recipes

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Recipe 1: Index array
# =====================
my @items = qw(alpha beta gamma delta);
my %index = map { $items[$_] => $_ } 0..$#items;

print "index of gamma: $index{gamma}\n";  # 2

# =====================
# Recipe 2: Count occurrences
# =====================
my @words = qw(a b a c b a d c a);
my %count;
$count{$_}++ for @words;

# Top 3
my @top = (sort { $count{$b} <=> $count{$a} } keys %count)[0..2];
print "Top 3: @top\n";

# =====================
# Recipe 3: Set operations
# =====================
my @set1 = qw(a b c d e);
my @set2 = qw(c d e f g);

my %h1 = map { $_ => 1 } @set1;
my %h2 = map { $_ => 1 } @set2;

my @union        = keys %{{ map { $_ => 1 } @set1, @set2 }};
my @intersection = grep { $h2{$_} } @set1;
my @diff1        = grep { !$h2{$_} } @set1;  # in set1 but not set2
my @diff2        = grep { !$h1{$_} } @set2;  # in set2 but not set1
my @sym_diff     = (@diff1, @diff2);

print "Union:        ", join(", ", sort @union), "\n";
print "Intersection: ", join(", ", sort @intersection), "\n";
print "Diff (1-2):   ", join(", ", sort @diff1), "\n";
print "Sym diff:     ", join(", ", sort @sym_diff), "\n";

# =====================
# Recipe 4: Multi-level grouping
# =====================
my @employees = (
    { name => "Alice",   dept => "IT",   level => "Senior", salary => 90000 },
    { name => "Bob",     dept => "IT",   level => "Junior", salary => 60000 },
    { name => "Charlie", dept => "HR",   level => "Senior", salary => 75000 },
    { name => "Diana",   dept => "IT",   level => "Senior", salary => 85000 },
    { name => "Eve",     dept => "HR",   level => "Junior", salary => 55000 },
);

# Group by dept → level
my %grouped;
for my $emp (@employees) {
    push @{$grouped{$emp->{dept}}{$emp->{level}}}, $emp;
}

foreach my $dept (sort keys %grouped) {
    print "\n$dept Department:\n";
    foreach my $level (sort keys %{$grouped{$dept}}) {
        print "  $level:\n";
        foreach my $e (@{$grouped{$dept}{$level}}) {
            printf "    %-12s %d\n", $e->{name}, $e->{salary};
        }
    }
}

# =====================
# Recipe 5: Pivot table
# =====================

# Sales data: month, product, amount
my @sales = (
    { month => "Jan", product => "A", amount => 100 },
    { month => "Jan", product => "B", amount => 150 },
    { month => "Feb", product => "A", amount => 120 },
    { month => "Feb", product => "B", amount => 180 },
    { month => "Mar", product => "A", amount => 90  },
    { month => "Mar", product => "B", amount => 200 },
);

my %pivot;
my (%months, %products);
for my $s (@sales) {
    $pivot{$s->{month}}{$s->{product}} += $s->{amount};
    $months{$s->{month}}++;
    $products{$s->{product}}++;
}

my @months   = sort keys %months;
my @products = sort keys %products;

# Print pivot table
printf "%-8s", "Month";
printf "%8s", $_ for @products;
print "   Total\n";
print "-" x (8 + 8 * @products + 8) . "\n";

my %col_totals;
foreach my $month (@months) {
    printf "%-8s", $month;
    my $row_total = 0;
    for my $prod (@products) {
        my $val = $pivot{$month}{$prod} // 0;
        printf "%8d", $val;
        $col_totals{$prod} += $val;
        $row_total += $val;
    }
    printf "%8d\n", $row_total;
}

print "-" x (8 + 8 * @products + 8) . "\n";
printf "%-8s", "Total";
printf "%8d", $col_totals{$_} for @products;
printf "%8d\n", sum(values %col_totals);

use List::Util qw(sum);
```

---

## Step 60: โปรแกรมสรุป — Mini Database

```perl
#!/usr/bin/perl
#
# โปรแกรม: mini_db.pl
# Mini in-memory database ด้วย Hash
#

use strict;
use warnings;
use List::Util qw(sum max min);
use POSIX qw(strftime);

# =====================
# Database schema
# =====================

my %db = (
    employees => {},  # id => employee record
    next_id   => 1,
);

# =====================
# CRUD Operations
# =====================

sub db_insert {
    my (%fields) = @_;
    my $id = $db{next_id}++;
    
    $db{employees}{$id} = {
        id         => $id,
        created_at => time(),
        %fields,
    };
    return $id;
}

sub db_get {
    my $id = shift;
    return $db{employees}{$id};
}

sub db_update {
    my ($id, %fields) = @_;
    return unless exists $db{employees}{$id};
    @{$db{employees}{$id}}{keys %fields} = values %fields;
    $db{employees}{$id}{updated_at} = time();
    return 1;
}

sub db_delete {
    my $id = shift;
    return delete $db{employees}{$id};
}

sub db_find {
    my (%criteria) = @_;
    
    my @results = grep {
        my $rec = $_;
        my $match = 1;
        
        while (my ($key, $val) = each %criteria) {
            if (ref $val eq 'CODE') {
                $match = 0 unless $val->($rec->{$key});
            } else {
                $match = 0 unless defined($rec->{$key}) && $rec->{$key} eq $val;
            }
        }
        $match;
    } values %{$db{employees}};
    
    return @results;
}

sub db_count {
    return scalar keys %{$db{employees}};
}

# =====================
# Populate data
# =====================

db_insert(name => "Alice",   dept => "IT",      salary => 90000, age => 30);
db_insert(name => "Bob",     dept => "HR",      salary => 65000, age => 28);
db_insert(name => "Charlie", dept => "IT",      salary => 85000, age => 32);
db_insert(name => "Diana",   dept => "Finance", salary => 78000, age => 35);
db_insert(name => "Eve",     dept => "IT",      salary => 92000, age => 27);
db_insert(name => "Frank",   dept => "HR",      salary => 70000, age => 40);

# =====================
# Queries
# =====================

print "=" x 50 . "\n";
print "All Employees:\n";
print "-" x 50 . "\n";
printf "%-4s %-12s %-10s %8s %5s\n", "ID", "Name", "Dept", "Salary", "Age";
print "-" x 50 . "\n";

foreach my $id (sort { $a <=> $b } keys %{$db{employees}}) {
    my $e = db_get($id);
    printf "%-4d %-12s %-10s %8d %5d\n",
        $e->{id}, $e->{name}, $e->{dept}, $e->{salary}, $e->{age};
}

# Query IT department
my @it = db_find(dept => "IT");
print "\nIT Department (", scalar @it, " employees):\n";
printf "  - %s (salary: %d)\n", $_->{name}, $_->{salary} for @it;

# Query high earners
my @high = db_find(salary => sub { $_[0] >= 85000 });
print "\nHigh earners (>= 85000):\n";
printf "  - %s: %d\n", $_->{name}, $_->{salary} for @high;

# Statistics
my @all_salaries = map { $_->{salary} } values %{$db{employees}};
printf "\nStatistics:\n";
printf "  Total employees: %d\n", db_count();
printf "  Total salary:    %d\n", sum(@all_salaries);
printf "  Avg salary:      %.0f\n", sum(@all_salaries) / @all_salaries;
printf "  Max salary:      %d\n", max(@all_salaries);
printf "  Min salary:      %d\n", min(@all_salaries);

# Update
db_update(1, salary => 95000);
print "\nAfter Alice's raise: ", db_get(1)->{salary}, "\n";

# Delete
db_delete(2);
print "After Bob deleted: ", db_count(), " employees remain\n";
```

---

## แบบฝึกหัด Part 06

### แบบฝึกหัดที่ 1: Word Counter
สร้างโปรแกรมอ่านไฟล์ข้อความและนับความถี่ของแต่ละคำ แสดง top 10

### แบบฝึกหัดที่ 2: Phone Book
สร้างสมุดโทรศัพท์ที่:
- เพิ่ม/แก้ไข/ลบผู้ติดต่อ
- ค้นหาด้วยชื่อ
- แสดงรายการทั้งหมด

### แบบฝึกหัดที่ 3: Grade Statistics
นำข้อมูลคะแนนของนักเรียนมาวิเคราะห์:
- จัดกลุ่มตามเกรด A/B/C/D/F
- หาค่าสถิติแต่ละกลุ่ม

---

## สรุป Part 06

ใน Part นี้คุณได้เรียนรู้:
- ✅ Hash พื้นฐาน: สร้าง, เข้าถึง, แก้ไข
- ✅ keys, values, each, delete, exists
- ✅ Hash of Arrays และ Hash of Hashes
- ✅ Merging, comparing, diffing hashes
- ✅ Dispatch tables
- ✅ Memoization
- ✅ Set operations
- ✅ Pivot table
- ✅ Mini in-memory database

**ถัดไป: [Part 07 — Operators และ Expressions](part_07.md)**
