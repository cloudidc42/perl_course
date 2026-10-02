# Part 07: Operators และ Expressions
## Steps 61-70: ตัวดำเนินการและนิพจน์ใน Perl

---

## Step 61: Arithmetic Operators

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# พื้นฐาน
# =====================

my $a = 15;
my $b = 4;

print "Addition:       ", $a + $b,  "\n";   # 19
print "Subtraction:    ", $a - $b,  "\n";   # 11
print "Multiplication: ", $a * $b,  "\n";   # 60
print "Division:       ", $a / $b,  "\n";   # 3.75
print "Modulo:         ", $a % $b,  "\n";   # 3
print "Exponentiation: ", $a ** $b, "\n";   # 50625

# =====================
# Integer vs Float
# =====================

print 7 / 2, "\n";      # 3.5 (float division)
print int(7/2), "\n";   # 3 (truncate)

use POSIX qw(floor ceil);
print floor(7/2), "\n"; # 3 (floor)
print ceil(7/2), "\n";  # 4 (ceil)

# =====================
# Augmented Assignment
# =====================

my $n = 100;
$n += 10;  print "$n\n";  # 110
$n -= 5;   print "$n\n";  # 105
$n *= 2;   print "$n\n";  # 210
$n /= 3;   print "$n\n";  # 70
$n **= 2;  print "$n\n";  # 4900
$n %= 100; print "$n\n";  # 0

# =====================
# Auto-increment string
# =====================

my $s = "aa";
print ++$s, "\n"; # ab
print ++$s, "\n"; # ac

$s = "Az";
print ++$s, "\n"; # Ba

$s = "Zz9";
print ++$s, "\n"; # AAa0

# =====================
# Infinity and NaN
# =====================

my $inf  = 9**9**9;        # Infinity
my $nan  = $inf - $inf;    # NaN
my $ninf = -(9**9**9);     # -Infinity

print "Inf: $inf\n";
print "NaN: $nan\n";
print "-Inf: $ninf\n";

# Test
use POSIX qw(HUGE_VAL);
print "is inf: ", ($inf == HUGE_VAL ? "yes" : "no"), "\n";
```

---

## Step 62: String Operators

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# String Operators
# =====================

my $s1 = "Hello";
my $s2 = "World";

# Concatenation
my $concat = $s1 . ", " . $s2 . "!";
print "$concat\n";  # Hello, World!

# Concat assignment
my $str = "Perl ";
$str .= "is ";
$str .= "awesome!";
print "$str\n";  # Perl is awesome!

# Repetition
my $line = "-" x 40;
print "$line\n";

my @arr = ("ha") x 3;
print "@arr\n";  # ha ha ha

# =====================
# Comparison Operators
# =====================

# Numeric
print "Numeric comparisons:\n";
print "3 == 3.0: ", (3 == 3.0 ? "true" : "false"), "\n";
print "3 != 4:   ", (3 != 4   ? "true" : "false"), "\n";
print "3 < 4:    ", (3 < 4    ? "true" : "false"), "\n";
print "3 > 4:    ", (3 > 4    ? "true" : "false"), "\n";
print "3 <= 3:   ", (3 <= 3   ? "true" : "false"), "\n";
print "3 >= 4:   ", (3 >= 4   ? "true" : "false"), "\n";

# String
print "\nString comparisons:\n";
print "abc eq abc: ", ("abc" eq "abc" ? "true" : "false"), "\n";
print "abc ne def: ", ("abc" ne "def" ? "true" : "false"), "\n";
print "abc lt def: ", ("abc" lt "def" ? "true" : "false"), "\n";
print "abc gt def: ", ("abc" gt "def" ? "true" : "false"), "\n";
print "abc le abc: ", ("abc" le "abc" ? "true" : "false"), "\n";

# Spaceship (numeric)
print "\nSpaceship: ", (5 <=> 10), "\n";  # -1
print "Spaceship: ", (10 <=> 5), "\n";   # 1
print "Spaceship: ", (5 <=> 5), "\n";    # 0

# cmp (string)
print "cmp: ", ("abc" cmp "abd"), "\n";  # -1
print "cmp: ", ("abd" cmp "abc"), "\n";  # 1
print "cmp: ", ("abc" cmp "abc"), "\n";  # 0

# =====================
# Sort using spaceship and cmp
# =====================
my @nums = (5, 2, 8, 1, 9, 3);
my @sorted = sort { $a <=> $b } @nums;
print "sorted: @sorted\n";

my @words = qw(banana apple cherry);
my @str_sorted = sort { $a cmp $b } @words;
print "sorted: @str_sorted\n";
```

---

## Step 63: Logical Operators

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Basic Logical
# =====================

my $t = 1;  # true
my $f = 0;  # false

# && / and
print "true  && true:  ", ($t && $t ? "true" : "false"), "\n";  # true
print "true  && false: ", ($t && $f ? "true" : "false"), "\n";  # false
print "false && true:  ", ($f && $t ? "true" : "false"), "\n";  # false
print "false && false: ", ($f && $f ? "true" : "false"), "\n";  # false

# || / or
print "\ntrue  || true:  ", ($t || $t ? "true" : "false"), "\n";  # true
print "true  || false: ", ($t || $f ? "true" : "false"), "\n";  # true
print "false || false: ", ($f || $f ? "true" : "false"), "\n";  # false

# ! / not
print "\n!true:  ", (!$t ? "true" : "false"), "\n";  # false
print "!false: ", (!$f ? "true" : "false"), "\n";  # true

# =====================
# Short-circuit evaluation
# =====================

# || returns first true value (or last value)
my $name = "" || "default";
print "name: $name\n";  # default

my $val = 0 || "" || undef || "found";
print "val: $val\n";  # found

# && returns first false value (or last value)
my $result = 1 && 2 && 3;
print "result: $result\n";  # 3

$result = 1 && 0 && 3;
print "result: $result\n";  # 0

# =====================
# // Defined-or (Perl 5.10+)
# =====================

my $x = undef // "default";
print "x: $x\n";  # default

$x = 0 // "default";
print "x: $x\n";  # 0 (0 is defined!)

$x = "" // "default";
print "x: $x\n";  # "" (empty string is defined)

# //= assignment
my $count;
$count //= 0;  # ถ้า undef ให้เป็น 0
$count++;
print "count: $count\n";  # 1

# =====================
# Chained || for defaults
# =====================

sub get_config {
    my $key = shift;
    my %config = (debug => 0, timeout => 30);
    return $config{$key};
}

my $debug   = get_config("debug")   // 0;
my $timeout = get_config("timeout") // 60;
my $host    = get_config("host")    // "localhost";

print "debug: $debug, timeout: $timeout, host: $host\n";

# =====================
# Ternary operator
# =====================

my $age = 20;
my $status = $age >= 18 ? "adult" : "minor";
print "status: $status\n";

# Nested ternary (ไม่แนะนำ แต่ใช้ได้)
my $grade = 85;
my $letter = $grade >= 90 ? "A" :
             $grade >= 80 ? "B" :
             $grade >= 70 ? "C" :
             $grade >= 60 ? "D" : "F";
print "grade: $letter\n";

# =====================
# Operator precedence
# =====================

print "\nPrecedence examples:\n";
print 2 + 3 * 4, "\n";     # 14 (not 20)
print (2 + 3) * 4, "\n";   # 20
print 2 ** 3 ** 2, "\n";   # 512 (right-associative: 2^(3^2)=2^9)
print (2 ** 3) ** 2, "\n"; # 64
```

---

## Step 64: Bitwise Operators

```perl
#!/usr/bin/perl
use strict;
use warnings;

my $a = 0b1010;  # 10
my $b = 0b1100;  # 12

printf "a  = %08b (%d)\n", $a, $a;  # 00001010
printf "b  = %08b (%d)\n", $b, $b;  # 00001100

# AND
printf "a & b = %08b (%d)\n", $a & $b, $a & $b;   # 00001000 (8)

# OR
printf "a | b = %08b (%d)\n", $a | $b, $a | $b;   # 00001110 (14)

# XOR
printf "a ^ b = %08b (%d)\n", $a ^ $b, $a ^ $b;   # 00000110 (6)

# NOT (bitwise complement)
printf "~a    = %08b (%d)\n", ~$a & 0xFF, ~$a & 0xFF;  # 11110101 (245)

# Left shift
printf "a << 2 = %08b (%d)\n", $a << 2, $a << 2;  # 00101000 (40)

# Right shift
printf "a >> 1 = %08b (%d)\n", $a >> 1, $a >> 1;  # 00000101 (5)

# =====================
# Bitwise use cases
# =====================

# Flags / Permissions
use constant {
    PERM_READ    => 0b001,  # 1
    PERM_WRITE   => 0b010,  # 2
    PERM_EXECUTE => 0b100,  # 4
};

my $perms = PERM_READ | PERM_WRITE;  # 3 = 011

# Check permission
if ($perms & PERM_READ)    { print "can read\n" }
if ($perms & PERM_WRITE)   { print "can write\n" }
if ($perms & PERM_EXECUTE) { print "can execute\n" }  # ไม่พิมพ์

# Add permission
$perms |= PERM_EXECUTE;
print "perms after adding execute: $perms\n";  # 7

# Remove permission
$perms &= ~PERM_WRITE;
print "perms after removing write: $perms\n";  # 5

# Toggle permission
$perms ^= PERM_READ;
print "perms after toggling read: $perms\n";  # 4

# =====================
# Fast multiply/divide by powers of 2
# =====================

my $n = 16;
print "n = $n\n";
print "n * 2 = ", $n << 1, "\n";    # 32
print "n * 4 = ", $n << 2, "\n";    # 64
print "n / 2 = ", $n >> 1, "\n";    # 8
print "n / 4 = ", $n >> 2, "\n";    # 4

# =====================
# Parity check
# =====================
sub is_even {
    return !($_[0] & 1);  # ถ้า bit0 เป็น 0 = even
}

for my $n (1..10) {
    printf "%2d is %s\n", $n, is_even($n) ? "even" : "odd";
}
```

---

## Step 65: Regular Expression Operators

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Match operator m//
# =====================

my $text = "Hello, World! 123";

# Basic match
if ($text =~ /World/) {
    print "Found 'World'\n";
}

# Negated match
if ($text !~ /xyz/) {
    print "Not found 'xyz'\n";
}

# Capture groups
if ($text =~ /(\w+), (\w+)/) {
    print "Captured: $1 and $2\n";
}

# Named captures (Perl 5.10+)
if ($text =~ /(?<greeting>\w+), (?<subject>\w+)/) {
    print "greeting: $+{greeting}\n";
    print "subject: $+{subject}\n";
}

# =====================
# Modifiers
# =====================

my $str = "Hello HELLO hello HeLLo";

# i — case insensitive
my @matches = ($str =~ /hello/gi);
print "count: ", scalar @matches, "\n";  # 4

# g — global
while ($str =~ /(\w+)/g) {
    print "word: $1\n";
}

# m — multiline (^ and $ match each line)
my $multi = "line1\nline2\nline3";
my @lines = ($multi =~ /^(\w+)$/mg);
print "lines: @lines\n";  # line1 line2 line3

# s — single-line (. matches \n)
my $dot_test = "Hello\nWorld";
print "s flag: ", ($dot_test =~ /Hello.World/s ? "match" : "no match"), "\n";

# x — extended (allow whitespace and comments)
if ($text =~ /
    (\d+)   # capture digits
    /x) {
    print "digits: $1\n";  # 123
}

# =====================
# Substitution s///
# =====================

my $s = "Hello World";
(my $new = $s) =~ s/World/Perl/;
print "$new\n";  # Hello Perl

# Global
$s = "aababab";
(my $g = $s) =~ s/a/X/g;
print "$g\n";  # XXbXbXb

# With function
my $text2 = "hello world";
(my $titled = $text2) =~ s/(\w+)/\u$1/g;
print "$titled\n";  # Hello World

# Eval in replacement
my $expr = "2 + 3 = ";
$expr =~ s/(\d+) \+ (\d+)/$1 + $2/e;  # /e evaluates replacement
print "$expr\n";

# =====================
# tr/// Transliteration
# =====================

my $tr_str = "Hello World";

# Count characters
my $h_count = ($tr_str =~ tr/Hh//);
print "H count: $h_count\n";  # 1

# Delete characters
(my $no_vowels = $tr_str) =~ tr/aeiouAEIOU//d;
print "$no_vowels\n";  # Hll Wrld

# Squeeze
my $spaces = "Hello    World";
$spaces =~ tr/ //s;  # squeeze spaces
print "$spaces\n";  # Hello World

# Uppercase
(my $upper = $tr_str) =~ tr/a-z/A-Z/;
print "$upper\n";  # HELLO WORLD
```

---

## Step 66: Range Operator

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Range ใน list context
# =====================

# Numeric range
my @digits = 1..10;
print "@digits\n";

# Step ไม่มีใน Perl แต่ทำได้ด้วย grep
my @evens = grep { $_ % 2 == 0 } 1..20;
print "evens: @evens\n";

# String range
my @letters = 'a'..'z';
print "@letters\n";

my @upper = 'A'..'Z';
print "@upper\n";

my @alnum = ('A'..'Z', 'a'..'z', '0'..'9');
print scalar @alnum, "\n";  # 62

# =====================
# Range ใน scalar context (flip-flop)
# =====================

# Range operator ใน while/for เป็น flip-flop
my @lines = (
    "before",
    "START",
    "line 1",
    "line 2", 
    "END",
    "after",
);

foreach my $line (@lines) {
    if ($line =~ /START/ .. $line =~ /END/) {
        print "In range: $line\n";
    }
}

# =====================
# Range กับ array operations
# =====================

my @arr = 'a'..'j';  # a b c d e f g h i j
print "full: @arr\n";

# Slice with range
print "2-5: @arr[2..5]\n";  # c d e f

# Reverse range
my @rev_range = reverse 1..5;
print "reverse: @rev_range\n";  # 5 4 3 2 1

# =====================
# Step sequences
# =====================

# Manual step
my @step2 = map { $_ * 2 } 1..10;
print "step 2: @step2\n";

# Floating point range
my @fracs = map { $_ / 10 } 0..10;
printf "%.1f ", $_ for @fracs;
print "\n";

# POSIX step
use List::Util;
# Every 3rd from 0 to 30
my @thirds = grep { $_ % 3 == 0 } 0..30;
print "every 3rd: @thirds\n";
```

---

## Step 67: Miscellaneous Operators

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Ternary ?:
# =====================

my $age = 20;
my $label = $age >= 18 ? "Adult" : "Minor";
print "$label\n";

# Nested ternary
my $score = 75;
my $grade = $score >= 90 ? "A" :
            $score >= 80 ? "B" :
            $score >= 70 ? "C" :
            $score >= 60 ? "D" : "F";
print "Grade: $grade\n";

# =====================
# String repetition
# =====================

print "-" x 40, "\n";
print "=" x 40, "\n";

# =====================
# qw// operator
# =====================

my @fruits = qw(apple banana cherry date elderberry);
my @months = qw(
    January February March
    April   May      June
    July    August   September
    October November December
);

# =====================
# // defined-or
# =====================

my %config;
my $host    = $config{host}    // "localhost";
my $port    = $config{port}    // 8080;
my $timeout = $config{timeout} // 30;

print "host: $host, port: $port, timeout: $timeout\n";

# //= assignment
$config{retries} //= 3;
$config{retries}++;
print "retries: $config{retries}\n";

# =====================
# Chained operators
# =====================

my $n = 5;
my $in_range = 1 <= $n && $n <= 10;  # ต้องเขียนแบบนี้ใน Perl
print "in range: $in_range\n";

# ========================
# String vs Number context
# ========================

my $str_num = "42abc";
my $plus = $str_num + 1;     # 43 (converts "42abc" to 42)
my $cat  = $str_num . "!";   # "42abc!" (string concat)

print "plus: $plus\n";
print "cat: $cat\n";

# =====================
# wantarray
# =====================

sub flexible {
    if (wantarray) {
        return (1, 2, 3);
    } else {
        return 42;
    }
}

my @list = flexible();
my $scalar = flexible();
print "list: @list\n";       # 1 2 3
print "scalar: $scalar\n";   # 42

# =====================
# local
# =====================

our $sep = "|";

sub print_with_sep {
    print join($sep, @_), "\n";
}

print_with_sep("a", "b", "c");  # a|b|c

{
    local $sep = ", ";
    print_with_sep("a", "b", "c");  # a, b, c
}

print_with_sep("a", "b", "c");  # a|b|c (restored)
```

---

## Step 68: Operator Precedence

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Perl operator precedence (highest to lowest):
# 1.  ->
# 2.  ++ --
# 3.  **
# 4.  \ ! ~ + - (unary)
# 5.  =~ !~
# 6.  * / % x
# 7.  + - .
# 8.  << >>
# 9.  named unary (abs, chr, etc.)
# 10. < > <= >= lt gt le ge
# 11. == != <=> eq ne cmp ~~
# 12. &
# 13. | ^
# 14. &&
# 15. ||  //
# 16. .. ...
# 17. ?:
# 18. = += -= etc.
# 19. , =>
# 20. list operators
# 21. not
# 22. and
# 23. or xor

# =====================
# Examples
# =====================

# ** is right-associative
print 2 ** 3 ** 2, "\n";    # 512 = 2^(3^2) = 2^9, NOT (2^3)^2=64
print (2 ** 3) ** 2, "\n";  # 64

# && has higher precedence than ||
my $a = 1;
my $b = 0;
my $c = 1;

my $r = $a || $b && $c;  # parsed as: $a || ($b && $c)
print "r = $r\n";          # 1 (because $a is true)

# and/or have VERY low precedence
my $x = 1 or die "won't die";   # OK
my $y = (1 or die "won't die"); # Same thing

# Dangerous:
# my $x = 1 || die;  # OK
# my $x = 1 or die;  # x = 1, then evaluate "or die" separately!
# Actually: (my $x = 1) or die; -- and x=1 is true, so no die

# Comma operator
my @arr = 1, 2, 3;  # WARNING: only 1 goes to @arr!
print "arr: @arr\n";  # 1 (2,3 are separate expressions)

my @arr2 = (1, 2, 3);  # Correct
print "arr2: @arr2\n";

# =====================
# Use parentheses to be clear
# =====================

# Confusing
my $result = 3 + 4 * 2 ** 2 - 1;
print "Without parens: $result\n";  # 3 + (4 * (2**2)) - 1 = 18

# Clear
my $result2 = 3 + (4 * (2 ** 2)) - 1;
print "With parens: $result2\n";    # Same: 18

# =====================
# Short-circuit side effects
# =====================

my $checked = 0;

sub check_expensive {
    $checked++;
    return 1;
}

my $fast = 0;

# If $fast is false, check_expensive is NOT called
my $r1 = $fast && check_expensive();
print "checked: $checked\n";  # 0 (not called due to short-circuit)

$fast = 1;
my $r2 = $fast && check_expensive();
print "checked: $checked\n";  # 1 (called)
```

---

## Step 69: Regular Expressions ใน Operations

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Using regex in conditions
# =====================

my @lines = (
    "2024-01-15: Purchase $1,200",
    "2024-02-20: Refund $-300",
    "invalid line",
    "2024-03-10: Transfer $500",
);

foreach my $line (@lines) {
    if ($line =~ /^(\d{4}-\d{2}-\d{2}):\s+(\w+)\s+\$(-?\d[\d,]*)$/) {
        printf "Date: %s, Type: %s, Amount: %s\n", $1, $2, $3;
    }
}

# =====================
# Using regex in sort
# =====================

my @files = qw(
    file10.txt file2.txt file1.txt
    file20.txt file3.txt file100.txt
);

# String sort (wrong order)
print "String sort: ", join(", ", sort @files), "\n";

# Natural sort
my @natural = map  { $_->[0] }
              sort { $a->[1] <=> $b->[1] }
              map  { 
                  my $n = /(\d+)/; 
                  [$_, $1 // 0] 
              } @files;
print "Natural sort: ", join(", ", @natural), "\n";

# =====================
# Regex in map/grep
# =====================

my @urls = (
    "http://www.example.com",
    "https://secure.example.com",
    "ftp://files.example.com",
    "https://api.example.com",
);

# Filter HTTPS only
my @https = grep { /^https:/ } @urls;
print "HTTPS URLs:\n";
print "  $_\n" for @https;

# Extract domains
my @domains = map { /^https?:\/\/(.+)$/ ? $1 : () } @urls;
print "Domains: @domains\n";

# =====================
# Global match in list context
# =====================

my $text = "Phone: 081-234-5678, Alt: 02-345-6789";
my @phones = ($text =~ /(\d[\d-]+\d)/g);
print "Phones: @phones\n";

# =====================
# Named captures
# =====================

my $date_str = "Born: 2000-05-15, Registered: 2024-01-01";
while ($date_str =~ /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/g) {
    printf "Date: %s/%s/%s\n", $+{day}, $+{month}, $+{year};
}
```

---

## Step 70: โปรแกรมสรุป — Expression Evaluator

```perl
#!/usr/bin/perl
#
# โปรแกรม: expr_eval.pl
# ตัวประเมินนิพจน์ทางคณิตศาสตร์
#

use strict;
use warnings;
use POSIX qw(floor ceil);
use List::Util qw(min max sum);
use Scalar::Util qw(looks_like_number);

# =====================
# Safe expression evaluator
# =====================

sub safe_eval_expr {
    my $expr = shift;
    
    # ตรวจสอบว่ามีแค่ chars ที่ปลอดภัย
    unless ($expr =~ /^[\d\s\+\-\*\/\%\(\)\.\,\^sqrt()\[\]]+$/i) {
        die "Invalid characters in expression: $expr\n";
    }
    
    # แปลง ^ เป็น **
    $expr =~ s/\^/**/g;
    
    # ใช้ eval ในขอบเขตที่จำกัด
    my $result = eval $expr;
    die "Eval error: $@\n" if $@;
    return $result;
}

# =====================
# Statistics calculator
# =====================

sub calc_stats {
    my @nums = @_;
    return {} unless @nums;
    
    my @sorted = sort { $a <=> $b } @nums;
    my $n      = scalar @nums;
    my $total  = sum(@nums);
    my $mean   = $total / $n;
    
    # Variance
    my $var = sum(map { ($_ - $mean) ** 2 } @nums) / $n;
    
    # Median
    my $median = $n % 2
        ? $sorted[$n/2]
        : ($sorted[$n/2-1] + $sorted[$n/2]) / 2;
    
    # Mode
    my %freq;
    $freq{$_}++ for @nums;
    my $max_freq = max(values %freq);
    my @modes = grep { $freq{$_} == $max_freq } sort { $a <=> $b } keys %freq;
    
    return {
        count    => $n,
        sum      => $total,
        mean     => $mean,
        median   => $median,
        mode     => \@modes,
        variance => $var,
        stddev   => sqrt($var),
        min      => $sorted[0],
        max      => $sorted[-1],
        range    => $sorted[-1] - $sorted[0],
    };
}

# =====================
# Bit operations demo
# =====================

sub show_bits {
    my ($label, $n) = @_;
    printf "%-15s = %08b = %3d = 0x%02X\n", $label, $n & 0xFF, $n, $n & 0xFF;
}

print "=" x 60 . "\n";
print "BITWISE OPERATIONS\n";
print "=" x 60 . "\n";

my $x = 0b10110100;  # 180
my $y = 0b01101110;  # 110

show_bits("x", $x);
show_bits("y", $y);
show_bits("x AND y", $x & $y);
show_bits("x OR y",  $x | $y);
show_bits("x XOR y", $x ^ $y);
show_bits("NOT x",   ~$x);
show_bits("x << 1",  $x << 1);
show_bits("x >> 2",  $x >> 2);

# =====================
# Statistics Demo
# =====================

print "\n" . "=" x 60 . "\n";
print "STATISTICS\n";
print "=" x 60 . "\n";

my @data = map { int(rand(100)) + 1 } 1..20;
srand(42);  # consistent seed
@data = map { int(rand(100)) + 1 } 1..20;

print "Data: ", join(", ", @data), "\n\n";

my $stats = calc_stats(@data);

printf "%-12s: %d\n",   "Count",    $stats->{count};
printf "%-12s: %d\n",   "Sum",      $stats->{sum};
printf "%-12s: %.2f\n", "Mean",     $stats->{mean};
printf "%-12s: %.2f\n", "Median",   $stats->{median};
printf "%-12s: %s\n",   "Mode",     join(", ", @{$stats->{mode}});
printf "%-12s: %.4f\n", "Std Dev",  $stats->{stddev};
printf "%-12s: %d\n",   "Min",      $stats->{min};
printf "%-12s: %d\n",   "Max",      $stats->{max};
printf "%-12s: %d\n",   "Range",    $stats->{range};
```

---

## แบบฝึกหัด Part 07

### แบบฝึกหัดที่ 1: Priority Queue
ใช้ bitwise operators เพื่อสร้าง flags system

### แบบฝึกหัดที่ 2: Expression Parser
รับนิพจน์ทางคณิตศาสตร์ และประเมินผล

### แบบฝึกหัดที่ 3: Operator Precedence Table
สร้างโปรแกรมแสดง operator precedence ด้วยตัวอย่าง

---

## สรุป Part 07

ใน Part นี้คุณได้เรียนรู้:
- ✅ Arithmetic operators และ augmented assignment
- ✅ String operators (., x, cmp, eq, ne, lt, gt)
- ✅ Logical operators (&&, ||, !, //)
- ✅ Bitwise operators (&, |, ^, ~, <<, >>)
- ✅ Regular expression operators (=~, !~, s///, tr///)
- ✅ Range operator (..)
- ✅ Ternary operator (?:)
- ✅ Operator precedence

**ถัดไป: [Part 08 — String Operations](part_08.md)**
