# Part 05: Arrays — อาร์เรย์
## Steps 41-50: การใช้งาน Array อย่างสมบูรณ์

---

## Step 41: Array พื้นฐาน

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# การสร้าง Array
# =====================

# Empty array
my @empty = ();

# Array ตัวเลข
my @numbers = (1, 2, 3, 4, 5);

# Array strings
my @fruits = ("apple", "banana", "cherry");

# qw// — shortcut สำหรับ word list
my @colors = qw(red green blue yellow);
my @months = qw(Jan Feb Mar Apr May Jun Jul Aug Sep Oct Nov Dec);

# Mixed array
my @mixed = (1, "hello", 3.14, undef, [1,2,3]);

# Range
my @digits  = (0..9);     # 0, 1, 2, ..., 9
my @letters = ('a'..'z'); # a, b, c, ..., z
my @upper   = ('A'..'Z');

print "@digits\n";
print "@letters\n";

# =====================
# การเข้าถึง Elements
# =====================

my @arr = qw(apple banana cherry date elderberry);

# Index เริ่มจาก 0
print $arr[0], "\n";   # apple
print $arr[1], "\n";   # banana
print $arr[-1], "\n";  # elderberry (จากท้าย)
print $arr[-2], "\n";  # date

# Slice — หลาย elements พร้อมกัน
my @slice = @arr[1, 3];
print "@slice\n";  # banana date

# Range slice
my @range_slice = @arr[1..3];
print "@range_slice\n";  # banana cherry date

# =====================
# ความยาว Array
# =====================

my @list = (1..10);
print "length: ", scalar @list, "\n";   # 10
print "last index: $#list\n";           # 9 ($# = last index)

# Empty check
if (@empty) {
    print "not empty\n";
} else {
    print "empty\n";  # แสดงนี้
}
```

---

## Step 42: การแก้ไข Array

```perl
#!/usr/bin/perl
use strict;
use warnings;

my @arr = qw(a b c d e);

# =====================
# push / pop (ท้าย array)
# =====================

push @arr, "f";           # เพิ่มที่ท้าย
push @arr, "g", "h";     # เพิ่มหลายตัว
print "@arr\n";  # a b c d e f g h

my $last = pop @arr;      # ลบจากท้าย
print "popped: $last\n";  # h
print "@arr\n";  # a b c d e f g

# =====================
# unshift / shift (หน้า array)
# =====================

unshift @arr, "Z";         # เพิ่มที่หน้า
unshift @arr, "X", "Y";   # เพิ่มหลายตัว
print "@arr\n";  # X Y Z a b c d e f g

my $first = shift @arr;    # ลบจากหน้า
print "shifted: $first\n"; # X
print "@arr\n";  # Y Z a b c d e f g

# =====================
# splice — แก้ไขส่วนกลาง
# =====================

my @nums = (1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

# ลบ elements
my @removed = splice(@nums, 2, 3);  # ลบ 3 elements จาก index 2
print "removed: @removed\n";   # 3 4 5
print "remaining: @nums\n";    # 1 2 6 7 8 9 10

# แทรก elements
splice(@nums, 2, 0, "a", "b");  # แทรกที่ index 2 โดยไม่ลบ
print "@nums\n";  # 1 2 a b 6 7 8 9 10

# แทนที่ elements
splice(@nums, 2, 2, "X", "Y", "Z");  # แทนที่ 2 elements ด้วย 3
print "@nums\n";  # 1 2 X Y Z 6 7 8 9 10

# =====================
# แก้ไข element โดยตรง
# =====================

my @colors = qw(red green blue);
$colors[1] = "yellow";  # แก้ไข index 1
print "@colors\n";  # red yellow blue

# เพิ่ม element ที่ index ที่ยังไม่มี (auto-extend)
my @short = (1, 2, 3);
$short[5] = 99;  # index 3, 4 จะเป็น undef
print "@short\n";  # 1 2 3   99 (มี undef ตรงกลาง)
print scalar @short, "\n";  # 6

# =====================
# delete element
# =====================

my @data = (1, 2, 3, 4, 5);
delete $data[2];  # ทำให้ index 2 เป็น undef (ไม่ลด size!)
print "@data\n";        # 1 2  4 5
print scalar @data, "\n";  # 5 (size ยังเท่าเดิม)

# =====================
# Extending array
# =====================

my @auto;
$auto[10] = "ten";
print "size: ", scalar @auto, "\n";  # 11
print "value: $auto[10]\n";
```

---

## Step 43: Array Functions

```perl
#!/usr/bin/perl
use strict;
use warnings;

my @nums = (5, 2, 8, 1, 9, 3, 7, 4, 6);

# =====================
# sort
# =====================

# Sort alphabetically (default)
my @alpha = sort qw(banana apple cherry date elderberry);
print "@alpha\n";  # apple banana cherry date elderberry

# Sort numerically
my @sorted_num = sort { $a <=> $b } @nums;
print "@sorted_num\n";  # 1 2 3 4 5 6 7 8 9

# Sort reverse
my @rev_sort = sort { $b <=> $a } @nums;
print "@rev_sort\n";  # 9 8 7 6 5 4 3 2 1

# Sort by string length
my @by_len = sort { length($a) <=> length($b) || $a cmp $b }
             qw(banana apple cherry kiwi fig);
print "@by_len\n";  # fig kiwi apple banana cherry

# Complex sort
my @people = (
    {name => "Charlie", age => 25},
    {name => "Alice",   age => 30},
    {name => "Bob",     age => 25},
);

my @by_age_name = sort {
    $a->{age} <=> $b->{age} ||
    $a->{name} cmp $b->{name}
} @people;

foreach my $p (@by_age_name) {
    print "$p->{name}: $p->{age}\n";
}

# Schwartzian Transform (efficient sort)
my @words = qw(banana apple cherry date elderberry fig);
my @sorted_by_len = map  { $_->[0] }
                    sort { $a->[1] <=> $b->[1] || $a->[0] cmp $b->[0] }
                    map  { [$_, length($_)] }
                    @words;
print "@sorted_by_len\n";

# =====================
# reverse
# =====================

my @rev = reverse @nums;
print "@rev\n";  # 6 4 7 3 9 1 8 2 5

my $rev_str = reverse "Hello";
print "$rev_str\n";  # olleH

# =====================
# grep — filter
# =====================

my @evens = grep { $_ % 2 == 0 } @nums;
print "evens: @evens\n";  # 2 8 4 6

my @over_five = grep { $_ > 5 } @nums;
print ">5: @over_five\n";  # 8 9 7 6

# grep กับ strings
my @fruits = qw(apple apricot banana blueberry cherry);
my @a_fruits = grep { /^a/i } @fruits;
print "starts with a: @a_fruits\n";  # apple apricot

my @long_fruits = grep { length($_) > 6 } @fruits;
print "long names: @long_fruits\n";  # apricot blueberry cherry

# =====================
# map — transform
# =====================

my @doubled = map { $_ * 2 } @nums;
print "doubled: @doubled\n";

my @squared = map { $_ ** 2 } (1..10);
print "squared: @squared\n";

my @upper = map { uc } qw(hello world foo);
print "upper: @upper\n";  # HELLO WORLD FOO

# map สร้าง hash
my @keys = qw(a b c d);
my %hash = map { $_ => length($_) } qw(apple banana cherry);
# hash: apple=>5, banana=>6, cherry=>6

foreach my $k (sort keys %hash) {
    print "$k => $hash{$k}\n";
}

# map ที่ซับซ้อน
my @data = (1..5);
my @result = map { ($_, $_ ** 2) } @data;
print "@result\n";  # 1 1 2 4 3 9 4 16 5 25
```

---

## Step 44: Array Slices และ Operations

```perl
#!/usr/bin/perl
use strict;
use warnings;

my @arr = qw(zero one two three four five six seven eight nine);

# =====================
# Array Slices
# =====================

# เลือกหลาย elements
my @slice1 = @arr[0, 2, 4];
print "@slice1\n";  # zero two four

# Range slice
my @slice2 = @arr[3..7];
print "@slice2\n";  # three four five six seven

# Negative index slice
my @last3 = @arr[-3..-1];
print "@last3\n";  # seven eight nine

# =====================
# Array Assignment
# =====================

my @a = (1..5);
my @b = (6..10);

# Concatenate arrays
my @combined = (@a, @b);
print "@combined\n";  # 1 2 3 4 5 6 7 8 9 10

# Assign to slice
my @dest = (0) x 5;  # [0, 0, 0, 0, 0]
@dest[1, 3] = (10, 30);
print "@dest\n";  # 0 10 0 30 0

# =====================
# Flatten ไม่มีจริงใน Perl
# =====================

# Array ใน Array จะถูก flatten อัตโนมัติ
my @flat = (1, (2, 3), (4, (5, 6)));
print "@flat\n";  # 1 2 3 4 5 6 (flat!)

# ถ้าต้องการ nested ต้องใช้ references
my @nested = (1, [2, 3], [4, [5, 6]]);
print scalar @nested, "\n";  # 3 (not 6)
print $nested[1][0], "\n";   # 2
print $nested[2][1][0], "\n"; # 5

# =====================
# Wantarray
# =====================

sub flexible_return {
    my @items = (1, 2, 3, 4, 5);
    
    if (wantarray) {
        return @items;  # list context
    } else {
        return scalar @items;  # scalar context
    }
}

my @list = flexible_return();
my $count = flexible_return();
print "list: @list\n";   # 1 2 3 4 5
print "count: $count\n"; # 5

# =====================
# each กับ Array (Perl 5.12+)
# =====================
my @letters = qw(a b c d e);
while (my ($idx, $val) = each @letters) {
    print "[$idx] = $val\n";
}
```

---

## Step 45: Sorting ขั้นสูง

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Multi-key sort
# =====================

my @students = (
    { name => "Charlie", grade => "B", score => 85 },
    { name => "Alice",   grade => "A", score => 92 },
    { name => "Bob",     grade => "A", score => 88 },
    { name => "Diana",   grade => "B", score => 85 },
    { name => "Eve",     grade => "C", score => 72 },
);

# Sort by grade, then score (desc), then name
my @sorted = sort {
    $a->{grade} cmp $b->{grade} ||
    $b->{score} <=> $a->{score} ||
    $a->{name}  cmp $b->{name}
} @students;

print "Sorted students:\n";
foreach my $s (@sorted) {
    printf "  %-10s Grade: %s  Score: %d\n",
        $s->{name}, $s->{grade}, $s->{score};
}

# =====================
# Stable sort (Perl sort is stable since 5.8)
# =====================

# =====================
# Custom comparator
# =====================

my @versions = qw(1.10 1.2 2.0 1.9 1.11 0.5);

# String sort (wrong for versions)
my @str_sorted = sort @versions;
print "string sort: @str_sorted\n";  # 0.5 1.10 1.11 1.2 1.9 2.0

# Numeric version sort
my @num_sorted = sort {
    my @av = split /\./, $a;
    my @bv = split /\./, $b;
    for my $i (0..$#av) {
        my $cmp = ($av[$i] // 0) <=> ($bv[$i] // 0);
        return $cmp if $cmp;
    }
    return 0;
} @versions;
print "version sort: @num_sorted\n";  # 0.5 1.2 1.9 1.10 1.11 2.0

# =====================
# Sort with CPAN (Sort::Naturally)
# =====================
# cpanm Sort::Naturally
# use Sort::Naturally;
# my @natural = nsort @versions;

# =====================
# Unique sort
# =====================
my @dupes = qw(c a b a c d b e);
my @unique_sorted = sort { $a cmp $b } do {
    my %seen;
    grep { !$seen{$_}++ } @dupes;
};
print "unique sorted: @unique_sorted\n";  # a b c d e

# หรือใช้ hash
my %seen;
my @unique = grep { !$seen{$_}++ } @dupes;
print "unique: @unique\n";  # c a b d e (preserve order)
```

---

## Step 46: Array Manipulation Patterns

```perl
#!/usr/bin/perl
use strict;
use warnings;
use List::Util qw(sum min max first reduce any all none uniq);

# =====================
# Unique elements
# =====================
my @arr = qw(a b a c b d a e);

# วิธีที่ 1: hash
my %seen;
my @unique = grep { !$seen{$_}++ } @arr;
print "unique: @unique\n";  # a b c d e (preserve order)

# วิธีที่ 2: uniq (List::Util 1.45+)
# my @unique2 = uniq @arr;

# วิธีที่ 3: sort แล้วลบ dupes
my @sorted_unique = do {
    my $prev = '';
    grep { $prev ne $_ ? do { $prev = $_; 1 } : 0 } sort @arr;
};
print "sorted unique: @sorted_unique\n";

# =====================
# Flatten nested arrays
# =====================
sub flatten {
    map { ref $_ eq 'ARRAY' ? flatten(@$_) : $_ } @_;
}

my @nested = (1, [2, 3], [4, [5, 6]], 7);
my @flat = flatten(@nested);
print "flat: @flat\n";  # 1 2 3 4 5 6 7

# =====================
# Chunk array
# =====================
sub chunk {
    my ($n, @arr) = @_;
    my @chunks;
    while (@arr) {
        push @chunks, [splice(@arr, 0, $n)];
    }
    return @chunks;
}

my @big = (1..10);
my @chunks = chunk(3, @big);
foreach my $chunk (@chunks) {
    print "chunk: @$chunk\n";
}

# =====================
# Zip arrays
# =====================
sub zip {
    my @arrays = @_;
    my $max_len = max(map { scalar @$_ } @arrays);
    my @result;
    for my $i (0..$max_len-1) {
        push @result, [map { $_->[$i] } @arrays];
    }
    return @result;
}

my @a1 = (1, 2, 3);
my @a2 = qw(a b c);
my @a3 = qw(x y z);

my @zipped = zip(\@a1, \@a2, \@a3);
foreach my $z (@zipped) {
    print "(@$z)\n";
}
# (1 a x)
# (2 b y)
# (3 c z)

# =====================
# Rotate array
# =====================
sub rotate_left {
    my ($n, @arr) = @_;
    $n %= @arr;
    return (@arr[$n..$#arr], @arr[0..$n-1]);
}

sub rotate_right {
    my ($n, @arr) = @_;
    return rotate_left(@arr - $n, @arr);
}

my @orig = (1..5);
my @rotl = rotate_left(2, @orig);
my @rotr = rotate_right(2, @orig);
print "original: @orig\n";  # 1 2 3 4 5
print "rot left: @rotl\n";  # 3 4 5 1 2
print "rot right: @rotr\n"; # 4 5 1 2 3

# =====================
# Partition array
# =====================
sub partition {
    my ($pred, @arr) = @_;
    my (@yes, @no);
    for (@arr) {
        $pred->($_) ? push(@yes, $_) : push(@no, $_);
    }
    return (\@yes, \@no);
}

my @numbers = (1..10);
my ($evens, $odds) = partition(sub { $_[0] % 2 == 0 }, @numbers);
print "evens: @$evens\n";  # 2 4 6 8 10
print "odds: @$odds\n";    # 1 3 5 7 9
```

---

## Step 47: Arrays ใน Context ต่างๆ

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Array ใน boolean context
# =====================
my @empty = ();
my @full  = (1, 2, 3);

print "empty: ", @empty ? "true" : "false", "\n";  # false
print "full: ",  @full  ? "true" : "false", "\n";  # true

# =====================
# Array ใน numeric context
# =====================
my @arr = (1..5);
my $count = 0 + @arr;  # force numeric
print "count: $count\n";  # 5

my $sum = @arr + @arr;
print "sum of counts: $sum\n";  # 10 (count + count)

# =====================
# Interpolation ใน strings
# =====================
my @colors = qw(red green blue);

print "Colors: @colors\n";          # Colors: red green blue (space sep)
print "First: $colors[0]\n";        # First: red
print "Count: @{[scalar @colors]}\n"; # Count: 3

# เปลี่ยน separator
{
    local $" = ", ";  # $" = list separator ใน ""
    print "Colors: @colors\n";  # Colors: red, green, blue
}

# =====================
# Assignment contexts
# =====================

# List assignment
my ($a, $b, $c) = @colors;
print "$a $b $c\n";  # red green blue

# Remainder ไป array
my ($first, @rest) = @colors;
print "first: $first\n";  # red
print "rest: @rest\n";    # green blue

# Count assignment
my $n = @colors;
print "n: $n\n";  # 3

# =====================
# Arrays as stacks and queues
# =====================

# Stack (LIFO)
my @stack;
push @stack, "a";
push @stack, "b";
push @stack, "c";
print "stack top: ", $stack[-1], "\n";  # c

while (@stack) {
    print "pop: ", pop(@stack), "\n";
}
# c, b, a

# Queue (FIFO)
my @queue;
push @queue, "first";
push @queue, "second";
push @queue, "third";

while (@queue) {
    print "dequeue: ", shift(@queue), "\n";
}
# first, second, third

# Priority Queue (simple — sort)
my @pq = map { {priority => $_, task => "Task $_"} } (3, 1, 4, 1, 5, 9, 2, 6);

# เรียงตาม priority
@pq = sort { $a->{priority} <=> $b->{priority} } @pq;

while (@pq) {
    my $item = shift @pq;
    print "Process: $item->{task} (priority $item->{priority})\n";
}
```

---

## Step 48: Multi-dimensional Arrays

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# 2D Array (Array of Arrays)
# =====================

# สร้างด้วย anonymous array refs
my @matrix = (
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9],
);

# เข้าถึง
print $matrix[0][0], "\n";   # 1
print $matrix[1][2], "\n";   # 6
print $matrix[2][1], "\n";   # 8

# Loop 2D
for my $i (0..$#matrix) {
    for my $j (0..$#{$matrix[$i]}) {
        printf "%3d", $matrix[$i][$j];
    }
    print "\n";
}

# =====================
# Matrix operations
# =====================

sub matrix_add {
    my ($a, $b) = @_;
    my @result;
    for my $i (0..$#$a) {
        for my $j (0..$#{$a->[$i]}) {
            $result[$i][$j] = $a->[$i][$j] + $b->[$i][$j];
        }
    }
    return \@result;
}

sub matrix_multiply {
    my ($a, $b) = @_;
    my $rows_a = scalar @$a;
    my $cols_a = scalar @{$a->[0]};
    my $cols_b = scalar @{$b->[0]};
    
    my @result = map { [(0) x $cols_b] } 0..$rows_a-1;
    
    for my $i (0..$rows_a-1) {
        for my $j (0..$cols_b-1) {
            for my $k (0..$cols_a-1) {
                $result[$i][$j] += $a->[$i][$k] * $b->[$k][$j];
            }
        }
    }
    return \@result;
}

sub print_matrix {
    my ($m, $label) = @_;
    print "$label:\n" if $label;
    for my $row (@$m) {
        print "  [", join(", ", map { sprintf "%4d", $_ } @$row), "]\n";
    }
}

my @m1 = ([1, 2], [3, 4]);
my @m2 = ([5, 6], [7, 8]);

my $sum  = matrix_add(\@m1, \@m2);
my $prod = matrix_multiply(\@m1, \@m2);

print_matrix(\@m1, "M1");
print_matrix(\@m2, "M2");
print_matrix($sum, "M1 + M2");
print_matrix($prod, "M1 * M2");

# =====================
# 3D Array
# =====================
my @cube;
for my $i (0..2) {
    for my $j (0..2) {
        for my $k (0..2) {
            $cube[$i][$j][$k] = $i * 100 + $j * 10 + $k;
        }
    }
}

print "cube[1][2][0] = $cube[1][2][0]\n";  # 120
```

---

## Step 49: Array Algorithms

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Searching
# =====================

# Linear search
sub linear_search {
    my ($arr_ref, $target) = @_;
    for my $i (0..$#$arr_ref) {
        return $i if $arr_ref->[$i] == $target;
    }
    return -1;
}

# Binary search (sorted array)
sub binary_search {
    my ($arr_ref, $target) = @_;
    my ($lo, $hi) = (0, $#$arr_ref);
    
    while ($lo <= $hi) {
        my $mid = int(($lo + $hi) / 2);
        if    ($arr_ref->[$mid] == $target) { return $mid }
        elsif ($arr_ref->[$mid] < $target)  { $lo = $mid + 1 }
        else                                { $hi = $mid - 1 }
    }
    return -1;
}

my @sorted_arr = (1, 3, 5, 7, 9, 11, 13, 15, 17, 19);
print "linear search 11: ", linear_search(\@sorted_arr, 11), "\n"; # 5
print "binary search 11: ", binary_search(\@sorted_arr, 11), "\n"; # 5
print "search 20: ", binary_search(\@sorted_arr, 20), "\n";        # -1

# =====================
# Sorting Algorithms (เพื่อการเรียนรู้)
# =====================

# Bubble Sort
sub bubble_sort {
    my @arr = @_;
    my $n = @arr;
    
    for my $i (0..$n-2) {
        for my $j (0..$n-$i-2) {
            if ($arr[$j] > $arr[$j+1]) {
                @arr[$j, $j+1] = @arr[$j+1, $j];
            }
        }
    }
    return @arr;
}

# Merge Sort
sub merge_sort {
    my @arr = @_;
    return @arr if @arr <= 1;
    
    my $mid   = int(@arr / 2);
    my @left  = merge_sort(@arr[0..$mid-1]);
    my @right = merge_sort(@arr[$mid..$#arr]);
    
    my @merged;
    while (@left && @right) {
        push @merged, $left[0] <= $right[0] ? shift(@left) : shift(@right);
    }
    return (@merged, @left, @right);
}

my @unsorted = (64, 34, 25, 12, 22, 11, 90);
print "original:    @unsorted\n";
print "bubble sort: ", join(" ", bubble_sort(@unsorted)), "\n";
print "merge sort:  ", join(" ", merge_sort(@unsorted)), "\n";
print "perl sort:   ", join(" ", sort { $a <=> $b } @unsorted), "\n";

# =====================
# Permutations
# =====================
sub permutations {
    my @arr = @_;
    return (\@arr) if @arr <= 1;
    
    my @perms;
    for my $i (0..$#arr) {
        my @rest = (@arr[0..$i-1], @arr[$i+1..$#arr]);
        for my $perm (permutations(@rest)) {
            push @perms, [$arr[$i], @$perm];
        }
    }
    return @perms;
}

my @perms = permutations(1, 2, 3);
print "Permutations of (1,2,3):\n";
for my $p (@perms) {
    print "  (", join(", ", @$p), ")\n";
}

# =====================
# Combinations
# =====================
sub combinations {
    my ($r, @arr) = @_;
    return ([]) if $r == 0;
    return ()   if @arr == 0;
    
    my ($first, @rest) = @arr;
    my @with    = map { [$first, @$_] } combinations($r-1, @rest);
    my @without = combinations($r, @rest);
    return (@with, @without);
}

my @combos = combinations(2, 1, 2, 3, 4);
print "\nCombinations of 2 from (1,2,3,4):\n";
for my $c (@combos) {
    print "  (", join(", ", @$c), ")\n";
}
```

---

## Step 50: โปรแกรมสรุป — Student Grade System

```perl
#!/usr/bin/perl
#
# โปรแกรม: grade_system.pl
# ระบบจัดการคะแนนนักเรียน
#

use strict;
use warnings;
use List::Util qw(sum min max);
use POSIX qw(floor);

# =====================
# Data
# =====================
my @subjects = qw(Math Science English Thai Social);

my @students = (
    { name => "สมชาย",    scores => [85, 90, 78, 88, 92] },
    { name => "สมหญิง",   scores => [92, 88, 95, 82, 87] },
    { name => "วิชัย",    scores => [70, 75, 68, 80, 72] },
    { name => "สุดา",     scores => [88, 82, 90, 85, 89] },
    { name => "ประยุทธ์", scores => [60, 65, 72, 58, 62] },
    { name => "มาลี",     scores => [95, 92, 88, 96, 94] },
);

# =====================
# Grade calculation
# =====================
sub get_grade {
    my $avg = shift;
    return 'A' if $avg >= 80;
    return 'B' if $avg >= 70;
    return 'C' if $avg >= 60;
    return 'D' if $avg >= 50;
    return 'F';
}

# =====================
# Process student data
# =====================
foreach my $student (@students) {
    my @scores = @{$student->{scores}};
    $student->{total}   = sum(@scores);
    $student->{average} = $student->{total} / @scores;
    $student->{grade}   = get_grade($student->{average});
    $student->{min}     = min(@scores);
    $student->{max}     = max(@scores);
}

# =====================
# Display report
# =====================

# Header
printf "\n%-15s", "ชื่อ";
printf "%8s", $_ for @subjects;
printf "%8s%8s%8s%5s\n", "รวม", "เฉลี่ย", "เกรด";
print "-" x (15 + 8 * scalar(@subjects) + 24) . "\n";

# Sort by average (desc)
my @sorted = sort { $b->{average} <=> $a->{average} } @students;

foreach my $s (@sorted) {
    printf "%-15s", $s->{name};
    printf "%8d", $_ for @{$s->{scores}};
    printf "%8d%8.1f%8s\n", $s->{total}, $s->{average}, $s->{grade};
}

# =====================
# Subject statistics
# =====================
print "\n" . "=" x 50 . "\n";
print "สถิติรายวิชา\n";
print "=" x 50 . "\n";

for my $i (0..$#subjects) {
    my @subj_scores = map { $_->{scores}[$i] } @students;
    printf "%-12s: เฉลี่ย=%.1f  สูงสุด=%d  ต่ำสุด=%d\n",
        $subjects[$i],
        sum(@subj_scores) / @subj_scores,
        max(@subj_scores),
        min(@subj_scores);
}

# =====================
# Class statistics
# =====================
print "\n" . "=" x 50 . "\n";
print "สถิติทั้งชั้น\n";
print "=" x 50 . "\n";

my %grade_count;
$grade_count{$_->{grade}}++ for @students;

printf "%-10s: %d คน\n", "เกรด A", $grade_count{A} // 0;
printf "%-10s: %d คน\n", "เกรด B", $grade_count{B} // 0;
printf "%-10s: %d คน\n", "เกรด C", $grade_count{C} // 0;
printf "%-10s: %d คน\n", "เกรด D", $grade_count{D} // 0;
printf "%-10s: %d คน\n", "เกรด F", $grade_count{F} // 0;

my @all_avgs = map { $_->{average} } @students;
printf "\nเฉลี่ยทั้งชั้น: %.2f\n", sum(@all_avgs) / @all_avgs;
printf "คะแนนสูงสุด:  %.1f (คุณ%s)\n", 
    $sorted[0]{average}, $sorted[0]{name};
printf "คะแนนต่ำสุด:  %.1f (คุณ%s)\n", 
    $sorted[-1]{average}, $sorted[-1]{name};
```

---

## แบบฝึกหัด Part 05

### แบบฝึกหัดที่ 1: Array Operations
สร้างโปรแกรมที่:
1. รับตัวเลขจาก user จนกว่าจะกด Enter เปล่า
2. แสดงผลรวม, เฉลี่ย, ค่าสูงสุด/ต่ำสุด
3. Sort และแสดง
4. แสดงว่าตัวเลขไหนซ้ำกัน

### แบบฝึกหัดที่ 2: Matrix Calculator
สร้างโปรแกรมรับ matrix 2x2 สองตัว และคำนวณบวก, ลบ, คูณ

### แบบฝึกหัดที่ 3: Shopping Cart
```perl
# สร้างระบบตะกร้าสินค้าพื้นฐาน
# - เพิ่มสินค้า (ชื่อ, ราคา, จำนวน)
# - ลบสินค้า
# - แสดงรายการ
# - คำนวณยอดรวม
```

---

## สรุป Part 05

ใน Part นี้คุณได้เรียนรู้:
- ✅ การสร้างและเข้าถึง Array
- ✅ push, pop, shift, unshift, splice
- ✅ sort, reverse, grep, map
- ✅ Array slices
- ✅ Multi-dimensional arrays
- ✅ Sorting algorithms
- ✅ Array context และ interpolation
- ✅ โปรแกรม Grade System

**ถัดไป: [Part 06 — Hashes](part_06.md)**
