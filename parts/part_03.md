# Part 03: Variables และ Data Types
## Steps 21-30: ทำความเข้าใจตัวแปรและชนิดข้อมูลใน Perl

---

## Step 21: ประเภทของตัวแปรใน Perl

Perl มีตัวแปร 3 ประเภทหลัก โดยใช้ **sigil** (สัญลักษณ์นำหน้า) เพื่อบอกประเภท:

```perl
#!/usr/bin/perl
use strict;
use warnings;

# 1. SCALAR ($) — เก็บค่าเดียว
my $name   = "Somchai";        # string
my $age    = 25;                # integer
my $height = 1.75;             # float
my $flag   = 1;                # boolean (ไม่มี boolean type)

# 2. ARRAY (@) — เก็บ list ของค่า
my @fruits  = ("apple", "banana", "cherry");
my @numbers = (1, 2, 3, 4, 5);
my @mixed   = ("hello", 42, 3.14, undef);

# 3. HASH (%) — เก็บคู่ key-value
my %person = (
    name => "Somchai",
    age  => 25,
    city => "Bangkok",
);
```

### สัญลักษณ์และความหมาย

```
$  (scalar sigil)   — ตัวแปรเดี่ยว
@  (array sigil)    — อาร์เรย์
%  (hash sigil)     — แฮช
&  (code sigil)     — subroutine reference
*  (typeglob)       — typeglob (ขั้นสูง)
\  (reference)      — สร้าง reference
```

---

## Step 22: Scalar Variables ละเอียด

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Numeric Scalars
# =====================

my $int     = 42;           # integer
my $neg     = -15;          # negative integer
my $float   = 3.14;         # floating point
my $sci     = 1.5e10;       # scientific: 15000000000
my $hex_num = 0xFF;         # hexadecimal: 255
my $oct_num = 0755;         # octal: 493
my $bin_num = 0b10101010;   # binary: 170

# ตัวช่วยอ่านตัวเลขยาวๆ (Perl 5.8+)
my $million = 1_000_000;    # 1000000
my $pi_val  = 3.141_592_653_589_793;

print "$million\n";         # 1000000
print "$pi_val\n";          # 3.14159265358979

# =====================
# String Scalars
# =====================

my $single   = 'no $interpolation';  # single quote
my $double   = "has $interpolation"; # double quote (interpolates)
my $heredoc  = <<END;
Multi-line
string here
END

# =====================
# Special Values
# =====================

my $undef_var = undef;      # undefined value
my $empty     = "";         # empty string
my $zero      = 0;          # numeric zero
my $zero_str  = "0";        # string "0"

# ตรวจสอบ undef
print defined($undef_var) ? "defined\n" : "undefined\n";  # undefined
print defined($empty)     ? "defined\n" : "undefined\n";  # defined (empty string IS defined)

# =====================
# Context — Scalar ใช้แตกต่างกันตาม context
# =====================

my @array = (1, 2, 3, 4, 5);

# Numeric context
my $count = @array;  # จำนวนสมาชิก = 5
print "มี $count ตัว\n";

# String context
my $arr_str = "@array";  # "1 2 3 4 5"
print "Array: $arr_str\n";
```

---

## Step 23: Variable Declaration

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# my — Lexical scope
# =====================

{
    my $local = "ตัวแปรท้องถิ่น";
    print "$local\n";  # ใช้ได้
}
# print "$local\n";  # ERROR: $local ไม่มีแล้วนอก block

# my ใน for loop
for my $i (1..5) {
    print "$i ";
}
print "\n";
# print $i;  # ERROR: $i ไม่มีนอก loop

# =====================
# our — Package variable
# =====================

our $global = "ตัวแปร global";
print "$global\n";

# =====================
# local — Temporary global
# =====================

our $separator = ", ";

sub print_items {
    my @items = @_;
    print join($separator, @items), "\n";
}

print_items("a", "b", "c");  # a, b, c

{
    local $separator = " | ";  # เปลี่ยนชั่วคราว
    print_items("a", "b", "c");  # a | b | c
}

print_items("a", "b", "c");  # a, b, c (กลับมาเหมือนเดิม)

# =====================
# Declaring Multiple Variables
# =====================

my ($x, $y, $z) = (1, 2, 3);
print "$x, $y, $z\n";  # 1, 2, 3

my ($first, @rest) = (10, 20, 30, 40);
print "First: $first\n";   # 10
print "Rest: @rest\n";     # 20 30 40

# Swap values
($x, $y) = ($y, $x);
print "x=$x, y=$y\n";  # x=2, y=1
```

---

## Step 24: Type Coercion (การแปลงชนิดอัตโนมัติ)

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Perl แปลงชนิดข้อมูลอัตโนมัติตาม context

# =====================
# String → Number
# =====================

my $str_num = "42 baht";
my $result  = $str_num + 8;
print "$result\n";  # 50 (ใช้ตัวเลขตอนต้น)

my $str_only = "hello";
my $sum = $str_only + 5;
print "$sum\n";  # 5 (string ที่ไม่มีตัวเลขเท่ากับ 0)

my $pi_str  = "3.14 is pi";
my $doubled = $pi_str * 2;
print "$doubled\n";  # 6.28

# =====================
# Number → String
# =====================

my $num = 42;
my $str = "Value is: " . $num;
print "$str\n";  # Value is: 42

# จำนวน format เป็น string
my $big_num = 1234567890;
print length($big_num), "\n";  # 10 (ความยาว string ของตัวเลข)

# =====================
# Boolean Context
# =====================

# ค่าต่อไปนี้เป็น FALSE:
# undef, 0, "", "0"

my @test_values = (undef, 0, 0.0, "", "0", 1, -1, "hello", "0.0", "00");

print "Testing truth values:\n";
foreach my $val (@test_values) {
    my $display = defined($val) ? qq("$val") : "undef";
    printf "  %-10s → %s\n", $display, ($val ? "TRUE" : "FALSE");
}

# =====================
# Numeric String Comparison
# =====================

# == เปรียบเทียบตัวเลข
print "2 == '2.0': ", (2 == "2.0" ? "true" : "false"), "\n";  # true

# eq เปรียบเทียบ string
print "2 eq '2.0': ", (2 eq "2.0" ? "true" : "false"), "\n";  # false
```

---

## Step 25: Special Variables (ตัวแปรพิเศษ)

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# $_ — Default variable
# =====================

my @numbers = (1, 2, 3, 4, 5);

foreach (1..5) {
    print "$_ ";  # $_ คือค่าปัจจุบันใน loop
}
print "\n";

# ใช้กับ functions
@numbers = (3, 1, 4, 1, 5);
foreach (sort @numbers) {
    print "$_ ";  # sort ใช้ $_ แต่ละค่า
}
print "\n";

# =====================
# @_ — Arguments ของ subroutine
# =====================

sub add {
    my ($a, $b) = @_;  # @_ มี arguments ทั้งหมด
    return $a + $b;
}
print add(3, 4), "\n";  # 7

# =====================
# @ARGV — Command line arguments
# =====================

# รันด้วย: perl script.pl arg1 arg2 arg3
# @ARGV = ("arg1", "arg2", "arg3")

if (@ARGV) {
    print "Arguments: @ARGV\n";
    print "จำนวน: ", scalar @ARGV, "\n";
}

# =====================
# $0 — ชื่อ script
# =====================

print "Script: $0\n";

# =====================
# $/ — Input record separator
# =====================

# ค่าเริ่มต้นคือ "\n" (newline)
{
    local $/ = undef;  # Slurp mode — อ่านทั้งไฟล์ในครั้งเดียว
    # my $content = <FILE>;
}

# =====================
# $\ — Output record separator
# =====================

{
    local $\ = "\n";  # เติม newline ทุกครั้งที่ print
    print "บรรทัด 1";
    print "บรรทัด 2";
    print "บรรทัด 3";
}

# =====================
# $, — Output field separator
# =====================

{
    local $, = ", ";
    print "a", "b", "c";  # a, b, c
}
print "\n";

# =====================
# $. — Current line number
# =====================

# เมื่ออ่านไฟล์ $. มีเลขบรรทัดปัจจุบัน

# =====================
# $! — Error message
# =====================

open(my $fh, '<', 'nonexistent.txt') or warn "Error: $!\n";
# $! มี error message เช่น "No such file or directory"

# =====================
# $@ — eval error
# =====================

eval { die "something went wrong" };
if ($@) {
    print "Caught: $@";
}

# =====================
# $$ — Process ID
# =====================

print "PID: $$\n";

# =====================
# $0 vs ${^PROGRAM_NAME}
# =====================

print "Program: $0\n";

# =====================
# %ENV — Environment variables
# =====================

print "HOME: $ENV{HOME}\n";
print "PATH: $ENV{PATH}\n";
print "USER: $ENV{USER}\n";

# ตั้งค่า environment variable
$ENV{MY_VAR} = "hello";
print "MY_VAR: $ENV{MY_VAR}\n";
```

---

## Step 26: Constants (ค่าคงที่)

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# use constant
# =====================

use constant PI         => 3.14159265358979;
use constant MAX_SIZE   => 100;
use constant APP_NAME   => "MyApp";
use constant VERSION    => "1.0.0";

# ใช้ constant — ไม่ต้องใส่ $ นำหน้า
print "PI = ", PI, "\n";
print "Max size: ", MAX_SIZE, "\n";
print "App: ", APP_NAME, " v", VERSION, "\n";

# =====================
# Multiple constants ในครั้งเดียว
# =====================

use constant {
    RED   => 0xFF0000,
    GREEN => 0x00FF00,
    BLUE  => 0x0000FF,
    
    TRUE  => 1,
    FALSE => 0,
    
    DB_HOST => 'localhost',
    DB_PORT => 3306,
    DB_NAME => 'mydb',
};

printf "Red: #%06X\n", RED;
print "DB: ", DB_HOST, ":", DB_PORT, "/", DB_NAME, "\n";

# =====================
# Readonly (CPAN module)
# =====================

# cpanm Readonly
# use Readonly;
# Readonly my $MAX => 100;
# Readonly my @COLORS => ('red', 'green', 'blue');
# Readonly my %CONFIG => (host => 'localhost', port => 3306);

# =====================
# Constants เป็น Hash
# =====================

use constant COLORS => {
    red   => '#FF0000',
    green => '#00FF00',
    blue  => '#0000FF',
};

print "Red: ", COLORS->{red}, "\n";

# =====================
# Constants เป็น Array (ต้องใช้ arrayref)
# =====================

use constant DAYS => [qw(Mon Tue Wed Thu Fri Sat Sun)];

foreach my $day (@{+DAYS}) {
    print "$day ";
}
print "\n";
```

---

## Step 27: undef และการตรวจสอบค่า

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# undef คืออะไร
# =====================

my $x = undef;  # ไม่มีค่า
my $y;          # เหมือนกัน — ค่าเริ่มต้นคือ undef

# =====================
# defined()
# =====================

print defined($x) ? "defined\n" : "undefined\n";  # undefined
print defined(0) ? "defined\n" : "undefined\n";    # defined (0 is defined!)
print defined("") ? "defined\n" : "undefined\n";   # defined
print defined("0") ? "defined\n" : "undefined\n";  # defined

# =====================
# undef ใน context ต่างๆ
# =====================

my $undef = undef;
print "Numeric: ", $undef + 0, "\n";   # 0 (+ warning)
print "String: '", $undef // "", "'\n"; # '' (empty string)
print "Bool: ", ($undef ? "true" : "false"), "\n";  # false

# =====================
# ล้างค่าตัวแปร
# =====================

my $name = "Somchai";
$name = undef;  # ล้างค่า
print defined($name) ? "$name\n" : "ไม่มีค่า\n";  # ไม่มีค่า

# =====================
# undef กับ Arrays
# =====================

my @arr = (1, undef, 3, undef, 5);

# นับค่า defined
my @defined_only = grep { defined $_ } @arr;
print "จำนวน defined: ", scalar @defined_only, "\n";  # 3

# กรอง undef ออก
@arr = grep { defined } @arr;
print "@arr\n";  # 1 3 5

# =====================
# undef กับ Hashes
# =====================

my %hash = (a => 1, b => undef, c => 3);

# ตรวจสอบ key ที่มีค่า
foreach my $key (sort keys %hash) {
    if (defined $hash{$key}) {
        print "$key = $hash{$key}\n";
    } else {
        print "$key = <undef>\n";
    }
}

# =====================
# exists vs defined
# =====================

my %data = (name => "Somchai", score => undef);

# exists — ตรวจว่า key มีอยู่ไหม
print exists $data{name}  ? "name exists\n"  : "no name\n";   # exists
print exists $data{age}   ? "age exists\n"   : "no age\n";    # no age
print exists $data{score} ? "score exists\n" : "no score\n";  # exists (แม้จะเป็น undef)

# defined — ตรวจว่า key มีค่าที่ defined ไหม
print defined $data{name}  ? "name defined\n"  : "name undef\n";   # defined
print defined $data{score} ? "score defined\n" : "score undef\n";  # undef
```

---

## Step 28: Variable Scoping

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Lexical Scope (my)
# =====================

my $global_my = "accessible everywhere in this file";

{
    my $block_var = "only in this block";
    print "$block_var\n";  # OK
    print "$global_my\n";  # OK (จาก outer scope)
}
# print $block_var;  # ERROR!

# =====================
# Nested Scopes
# =====================

my $level = "file";

{
    my $level = "block1";  # shadow ตัวแปรด้านนอก
    print "level: $level\n";  # block1
    
    {
        my $level = "block2";
        print "level: $level\n";  # block2
    }
    
    print "level: $level\n";  # block1 (กลับมา)
}

print "level: $level\n";  # file (กลับมา)

# =====================
# Scope ใน if/while/for
# =====================

if (1) {
    my $if_var = "in if";
    print "$if_var\n";  # OK
}
# print $if_var;  # ERROR

for my $i (1..3) {
    my $square = $i ** 2;
    print "$i^2 = $square\n";
}
# print $i;  # ERROR

# =====================
# Package Variables (our)
# =====================

package MyPackage;

our $package_var = "package variable";  # accessible via $MyPackage::package_var

package main;

print "$MyPackage::package_var\n";  # package variable

# =====================
# Dynamic Scope (local)
# =====================

our $indent = 0;

sub print_indented {
    my ($text) = @_;
    print " " x $indent, $text, "\n";
}

sub print_section {
    my ($title, @items) = @_;
    print_indented($title);
    
    local $indent = $indent + 4;  # เพิ่ม indent ชั่วคราว
    
    foreach my $item (@items) {
        print_indented($item);
    }
    # $indent กลับมาเป็นค่าเดิมอัตโนมัติ
}

print_section("Section 1:", "Item A", "Item B", "Item C");
print_section("Section 2:", "Item X", "Item Y");
```

---

## Step 29: Complex Variable Operations

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Multiple Assignment
# =====================

# List assignment
my ($a, $b, $c) = (10, 20, 30);
print "$a $b $c\n";  # 10 20 30

# ถ้าค่าน้อยกว่าตัวแปร ที่เหลือเป็น undef
my ($x, $y, $z) = (1, 2);
print defined($z) ? $z : "undef", "\n";  # undef

# ถ้าค่ามากกว่าตัวแปร ส่วนเกินถูก ignore
my ($p, $q) = (1, 2, 3, 4);  # 3, 4 ถูก ignore
print "$p $q\n";  # 1 2

# =====================
# Swap ตัวแปร
# =====================

my $first = "Hello";
my $second = "World";
print "ก่อน swap: $first, $second\n";

($first, $second) = ($second, $first);
print "หลัง swap: $first, $second\n";  # World, Hello

# =====================
# Chained Assignment
# =====================

my $n1 = my $n2 = my $n3 = 0;
print "$n1 $n2 $n3\n";  # 0 0 0

# =====================
# Augmented Assignment
# =====================

my $num = 10;
$num += 5;   # $num = $num + 5 = 15
$num -= 3;   # $num = $num - 3 = 12
$num *= 2;   # $num = $num * 2 = 24
$num /= 4;   # $num = $num / 4 = 6
$num **= 2;  # $num = $num ** 2 = 36
$num %= 10;  # $num = $num % 10 = 6

print "num = $num\n";

my $str = "Hello";
$str .= ", World!";  # String concatenation assignment
print "$str\n";  # Hello, World!

# =====================
# Increment / Decrement
# =====================

my $count = 0;
$count++;  # post-increment: ใช้แล้วเพิ่ม
++$count;  # pre-increment: เพิ่มแล้วใช้
$count--;  # post-decrement
--$count;  # pre-decrement

print "count: $count\n";  # 0 (0+1+1-1-1=0)

# String auto-increment!
my $letter = "a";
$letter++;
print "$letter\n";  # b

my $str2 = "Az";
$str2++;
print "$str2\n";  # Ba

my $str3 = "zz";
$str3++;
print "$str3\n";  # aaa
```

---

## Step 30: โปรแกรมสรุป — ระบบจัดการข้อมูลส่วนตัว

```perl
#!/usr/bin/perl
#
# โปรแกรม: personal_info.pl
# ระบบจัดการข้อมูลส่วนตัวพื้นฐาน
#

use strict;
use warnings;
use POSIX qw(floor);

# =====================
# Constants
# =====================
use constant {
    MIN_AGE      => 0,
    MAX_AGE      => 150,
    MIN_HEIGHT   => 50,
    MAX_HEIGHT   => 300,
    APP_TITLE    => "ระบบข้อมูลส่วนตัว",
    BORDER_CHAR  => "=",
    BORDER_WIDTH => 50,
};

# =====================
# Utility functions
# =====================

sub border {
    return BORDER_CHAR x BORDER_WIDTH . "\n";
}

sub center_text {
    my ($text, $width) = @_;
    $width //= BORDER_WIDTH;
    my $padding = int(($width - length($text)) / 2);
    return " " x $padding . $text . "\n";
}

sub get_input {
    my ($prompt, $validator) = @_;
    
    while (1) {
        print $prompt;
        my $input = <STDIN>;
        chomp $input;
        
        if (!defined $validator || $validator->($input)) {
            return $input;
        }
    }
}

# =====================
# Validators
# =====================

my $is_not_empty = sub {
    my $v = shift;
    return length($v) > 0 || do { print "กรุณากรอกข้อมูล!\n"; 0 };
};

my $is_valid_age = sub {
    my $v = shift;
    if ($v =~ /^\d+$/ && $v >= MIN_AGE && $v <= MAX_AGE) {
        return 1;
    }
    print "กรอกอายุ " . MIN_AGE . "-" . MAX_AGE . "\n";
    return 0;
};

my $is_valid_height = sub {
    my $v = shift;
    if ($v =~ /^\d+\.?\d*$/ && $v >= MIN_HEIGHT && $v <= MAX_HEIGHT) {
        return 1;
    }
    print "กรอกส่วนสูง " . MIN_HEIGHT . "-" . MAX_HEIGHT . " ซม.\n";
    return 0;
};

my $is_valid_weight = sub {
    my $v = shift;
    if ($v =~ /^\d+\.?\d*$/ && $v > 0 && $v < 500) {
        return 1;
    }
    print "กรอกน้ำหนัก 1-499 กก.\n";
    return 0;
};

# =====================
# Data collection
# =====================

print border();
print center_text(APP_TITLE);
print border();
print "\n";

my %person;

$person{first_name} = get_input("ชื่อ: ", $is_not_empty);
$person{last_name}  = get_input("นามสกุล: ", $is_not_empty);
$person{age}        = get_input("อายุ: ", $is_valid_age);
$person{height}     = get_input("ส่วนสูง (ซม.): ", $is_valid_height);
$person{weight}     = get_input("น้ำหนัก (กก.): ", $is_valid_weight);
$person{city}       = get_input("เมือง: ", $is_not_empty);

# =====================
# Calculations
# =====================

my $bmi = $person{weight} / ($person{height}/100) ** 2;

my $bmi_category;
if    ($bmi < 18.5) { $bmi_category = "ต่ำกว่าเกณฑ์" }
elsif ($bmi < 25.0) { $bmi_category = "ปกติ" }
elsif ($bmi < 30.0) { $bmi_category = "น้ำหนักเกิน" }
else                { $bmi_category = "อ้วน" }

my $birth_year = 2024 - $person{age};

# =====================
# Display report
# =====================

print "\n";
print border();
print center_text("รายงานข้อมูลส่วนตัว");
print border();

printf "%-20s: %s %s\n", "ชื่อ-นามสกุล", $person{first_name}, $person{last_name};
printf "%-20s: %s ปี\n", "อายุ", $person{age};
printf "%-20s: ประมาณ พ.ศ. %d\n", "ปีเกิด", $birth_year + 543;
printf "%-20s: %.1f ซม.\n", "ส่วนสูง", $person{height};
printf "%-20s: %.1f กก.\n", "น้ำหนัก", $person{weight};
printf "%-20s: %.2f (%s)\n", "ดัชนีมวลกาย (BMI)", $bmi, $bmi_category;
printf "%-20s: %s\n", "เมือง", $person{city};

print border();
print "\n";

# =====================
# Special variables demo
# =====================

print "ข้อมูลเพิ่มเติม:\n";
print "  - รัน script: $0\n";
print "  - Process ID: $$\n";
print "  - เวลา: ", scalar(localtime()), "\n";
```

---

## แบบฝึกหัด Part 03

### แบบฝึกหัดที่ 1: Variable Types

```perl
#!/usr/bin/perl
use strict;
use warnings;

# สร้างตัวแปรแต่ละประเภทและแสดงผล:
# 1. Scalar: ชื่อ, อายุ, คะแนน, ประเทศ
# 2. Array: รายชื่อวิชา 5 วิชา
# 3. Hash: ข้อมูลนักเรียน (ชื่อ, อายุ, คะแนน, ชั้น)

# แสดงผลในรูปแบบที่อ่านง่าย
```

### แบบฝึกหัดที่ 2: Type Conversion

```perl
#!/usr/bin/perl
use strict;
use warnings;

# ทดลองการแปลงชนิดข้อมูล
my @tests = ("42", "3.14", "0", "", "hello", "42abc", undef);

foreach my $val (@tests) {
    # แสดง: ค่า, ชนิดที่คิดว่าเป็น, ผลบวกกับ 1, ผล concat กับ "x"
}
```

### แบบฝึกหัดที่ 3: Special Variables

ลองใช้ special variables ต่างๆ:
- อ่านไฟล์ด้วย `$/`
- เปลี่ยน output separator ด้วย `$,` และ `$\`
- ใช้ `$_` ใน loop

---

## สรุป Part 03

ใน Part นี้คุณได้เรียนรู้:
- ✅ ประเภทตัวแปร 3 แบบ: Scalar, Array, Hash
- ✅ การ declare ตัวแปรด้วย my, our, local
- ✅ Type coercion (แปลงชนิดอัตโนมัติ)
- ✅ Special variables ที่สำคัญ
- ✅ Constants ด้วย use constant
- ✅ undef และการตรวจสอบด้วย defined()
- ✅ Variable scoping
- ✅ Complex variable operations

**ถัดไป: [Part 04 — Scalars ละเอียด](part_04.md)**
