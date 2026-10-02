# Part 02: Hello World และโครงสร้างโปรแกรม
## Steps 11-20: เข้าใจโครงสร้างพื้นฐานของ Perl

---

## Step 11: โครงสร้างโปรแกรม Perl

ทุกโปรแกรม Perl มีโครงสร้างหลักที่ควรมี:

```perl
#!/usr/bin/perl
#
# ส่วนที่ 1: Shebang Line
# บอก OS ว่าไฟล์นี้รันด้วย Perl
#

use strict;    # ส่วนที่ 2: Pragmas
use warnings;

# ส่วนที่ 3: โค้ดหลัก
print "Hello, World!\n";

# ส่วนที่ 4: Subroutines (ฟังก์ชัน)
sub greet {
    my ($name) = @_;
    return "สวัสดี, $name!";
}

print greet("Perl"), "\n";
```

### รายละเอียดแต่ละส่วน

```perl
# 1. SHEBANG LINE
#!/usr/bin/perl
# หรือ
#!/usr/bin/env perl  # ยืดหยุ่นกว่า หา perl ใน PATH

# 2. PRAGMAS — ตัวช่วยควบคุมพฤติกรรม Perl
use strict;     # บังคับ declare variables (ป้องกัน typo)
use warnings;   # แสดง warning เมื่อมีสิ่งผิดปกติ
use utf8;       # รองรับ Unicode ใน source code
use feature 'say';  # เปิดใช้ say() function

# 3. MODULES
use POSIX;
use File::Path;
use CGI;

# 4. GLOBAL VARIABLES (ไม่แนะนำ แต่ใช้ได้)
our $DEBUG = 1;

# 5. MAIN CODE
# ...

# 6. SUBROUTINES
sub function_name { }
```

---

## Step 12: Pragmas ที่สำคัญ

### `use strict`

```perl
#!/usr/bin/perl
use strict;

# ถ้าไม่มี use strict:
$name = "Somchai";  # ใช้ได้ แต่ไม่ดี

# ถ้ามี use strict:
# $name = "Somchai";  # ERROR: Global symbol requires explicit package name
my $name = "Somchai";  # ถูกต้อง — ต้อง declare ด้วย my

# strict บังคับ 3 อย่าง:
# 1. vars   — ต้อง declare variables
# 2. subs   — ต้อง declare subroutines ก่อนใช้
# 3. refs   — ห้ามใช้ symbolic references
```

### `use warnings`

```perl
#!/usr/bin/perl
use warnings;

my $x;
print $x;  # Warning: Use of uninitialized value $x

my @arr = (1, 2, 3);
my $val = $arr[10];  # Warning: array index out of bounds (ไม่ error แต่ warn)

# เปิด/ปิด warnings เฉพาะส่วน
{
    no warnings 'uninitialized';
    my $y;
    print $y;  # ไม่แสดง warning
}
```

### `use utf8`

```perl
#!/usr/bin/perl
use strict;
use warnings;
use utf8;  # บอก Perl ว่า source code เป็น UTF-8

# ตอนนี้สามารถใช้ Thai/Unicode ใน code ได้
my $greeting = "สวัสดีครับ";
my $emoji    = "🎉";

print "$greeting $emoji\n";

# สำหรับ I/O ต้องตั้งค่าเพิ่มเติม
use open ':std', ':encoding(UTF-8)';
```

### `use feature`

```perl
#!/usr/bin/perl
use strict;
use warnings;
use feature 'say';    # เพิ่ม say() — เหมือน print แต่เติม newline
use feature 'state';  # เพิ่ม state variables
use feature ':5.10';  # เปิดทุก features ของ Perl 5.10

say "Hello!";         # เหมือน print "Hello!\n"

sub counter {
    state $count = 0;  # ค่าคงอยู่ระหว่าง calls
    return ++$count;
}

say counter();  # 1
say counter();  # 2
say counter();  # 3
```

### `use v5.36` หรือ newer syntax

```perl
#!/usr/bin/perl
use v5.36;  # เปิดทุก features ของ Perl 5.36 + strict + warnings

# เหมือนกับเขียน:
# use strict;
# use warnings;
# use feature ':5.36';
```

---

## Step 13: การแสดงผล — print และ say

### print

```perl
#!/usr/bin/perl
use strict;
use warnings;

# print พื้นฐาน
print "Hello, World!\n";    # \n = newline
print "สวัสดี\n";

# print หลายอาร์กิวเมนต์ (คั่นด้วย $, หรือไม่มีอะไร)
print "Hello", " ", "World", "\n";

# print กับ list
my @items = ("apple", "banana", "cherry");
print @items;           # applebananacherry (ไม่มี separator)
print "@items\n";       # apple banana cherry (มี space)

# print ไปยัง filehandle
print STDOUT "ไปยัง standard output\n";
print STDERR "ไปยัง standard error\n";

# เปลี่ยน default output separator
$, = ", ";      # $, = Output Field Separator
$\ = "\n";      # $\ = Output Record Separator

print "a", "b", "c";   # a, b, c
                        # (แล้วขึ้นบรรทัดใหม่อัตโนมัติ)

# Reset
$, = undef;
$\ = undef;
```

### say (Perl 5.10+)

```perl
#!/usr/bin/perl
use strict;
use warnings;
use feature 'say';

# say เหมือน print แต่เติม \n อัตโนมัติ
say "Hello!";           # Hello! + newline
say "สวัสดี";          # สวัสดี + newline

# เปรียบเทียบ
print "Hello\n";        # เหมือนกัน
say "Hello";

# say กับ list
say "a", "b", "c";      # abc + newline
```

### printf — Formatted Output

```perl
#!/usr/bin/perl
use strict;
use warnings;

# printf เหมือน C printf
printf "Name: %s\n", "Somchai";
printf "Age: %d\n", 25;
printf "Price: %.2f\n", 199.999;
printf "Hex: %x\n", 255;          # ff
printf "Oct: %o\n", 8;            # 10
printf "Binary: %b\n", 10;        # 1010

# Format specifiers
# %s  — string
# %d  — integer
# %f  — float
# %e  — scientific notation
# %g  — shorter of %f or %e
# %x  — hexadecimal (lowercase)
# %X  — hexadecimal (uppercase)
# %o  — octal
# %b  — binary
# %c  — character
# %%  — literal %

# Width and precision
printf "%10s\n", "Hello";     # "     Hello" (right-aligned, width 10)
printf "%-10s\n", "Hello";    # "Hello     " (left-aligned)
printf "%010d\n", 42;         # "0000000042" (zero-padded)
printf "%.5f\n", 3.14159265;  # "3.14159" (5 decimal places)

# ตัวอย่างการจัดตาราง
printf "%-20s %5s %10s\n", "สินค้า", "จำนวน", "ราคา";
printf "%-20s %5d %10.2f\n", "แอปเปิ้ล", 10, 25.50;
printf "%-20s %5d %10.2f\n", "กล้วยหอม", 5, 15.00;
printf "%-20s %5d %10.2f\n", "มะม่วง", 8, 40.00;
```

### sprintf — Format ไม่แสดงผล

```perl
#!/usr/bin/perl
use strict;
use warnings;

# sprintf ส่งกลับ string ที่ format แล้ว
my $formatted = sprintf("%.2f", 3.14159);
print "ค่า pi ≈ $formatted\n";  # 3.14

my $padded = sprintf("%05d", 42);
print "รหัส: $padded\n";  # 00042

# ใช้สร้าง string ที่ format แล้ว
my $name  = "Somchai";
my $score = 95.5;
my $report = sprintf("นักเรียน: %-20s คะแนน: %6.1f", $name, $score);
print "$report\n";
```

---

## Step 14: การรับ Input จากผู้ใช้

### STDIN — Standard Input

```perl
#!/usr/bin/perl
use strict;
use warnings;

# รับ input จาก keyboard
print "กรอกชื่อของคุณ: ";
my $name = <STDIN>;     # อ่านบรรทัดจาก STDIN
chomp $name;            # ลบ newline ออก

print "สวัสดี, $name!\n";
```

### chomp และ chop

```perl
#!/usr/bin/perl
use strict;
use warnings;

# chomp — ลบ newline ที่ท้าย string
my $line = "Hello\n";
chomp $line;
print length($line), "\n";  # 5 (ลบ \n ออก)

# chomp กับ array
my @lines = ("line1\n", "line2\n", "line3\n");
chomp @lines;
print "@lines\n";  # line1 line2 line3

# chop — ลบ character สุดท้าย (ไม่ว่าจะเป็นอะไร)
my $str = "Hello!";
my $removed = chop $str;
print "$str\n";      # Hello
print "$removed\n";  # !
```

### การรับ Input หลายค่า

```perl
#!/usr/bin/perl
use strict;
use warnings;

print "กรอกชื่อ: ";
my $name = <STDIN>;
chomp $name;

print "กรอกอายุ: ";
my $age = <STDIN>;
chomp $age;

print "กรอกเมือง: ";
my $city = <STDIN>;
chomp $city;

print "\n--- ข้อมูลของคุณ ---\n";
print "ชื่อ: $name\n";
print "อายุ: $age ปี\n";
print "เมือง: $city\n";
```

### การตรวจสอบ Input

```perl
#!/usr/bin/perl
use strict;
use warnings;

sub get_number {
    my ($prompt) = @_;
    
    while (1) {
        print $prompt;
        my $input = <STDIN>;
        chomp $input;
        
        # ตรวจสอบว่าเป็นตัวเลข
        if ($input =~ /^\d+$/) {
            return $input;
        }
        print "กรุณากรอกตัวเลขเท่านั้น!\n";
    }
}

my $num1 = get_number("กรอกตัวเลขแรก: ");
my $num2 = get_number("กรอกตัวเลขที่สอง: ");
print "ผลรวม: ", $num1 + $num2, "\n";
```

---

## Step 15: Escape Sequences

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Escape sequences ใน double-quoted strings
print "บรรทัดที่ 1\nบรรทัดที่ 2\n";  # \n = newline
print "Tab:\tตาราง\n";                 # \t = tab
print "Backslash: \\\n";               # \\ = literal \
print "Double quote: \"\n";            # \" = literal "
print "Bell: \a";                      # \a = bell (beep)
print "Carriage return: test\r\n";     # \r = carriage return
print "Null: \0\n";                    # \0 = null character
print "Hex: \x41\n";                   # \x41 = 'A'
print "Unicode: \x{263A}\n";           # ☺ emoji

# Single-quoted strings — ไม่ interpret escape sequences
print 'ไม่มี \n interpolation', "\n";   # แสดง \n ตามตัว
print 'ไม่มี $variable', "\n";          # แสดง $variable ตามตัว

# heredoc
my $text = <<END;
นี่คือ heredoc
สามารถเขียนหลายบรรทัด
รองรับ variable interpolation: Perl
END

print $text;

# heredoc แบบ indented (Perl 5.26+)
my $indented = <<~END;
    บรรทัดนี้มี indent
    แต่จะถูกลบออก
    END

print $indented;
```

---

## Step 16: String Interpolation

```perl
#!/usr/bin/perl
use strict;
use warnings;

my $name  = "สมชาย";
my $age   = 25;
my @fruits = ("แอปเปิ้ล", "กล้วย", "ส้ม");
my %info   = (city => "กรุงเทพ");

# Double-quoted string — interpolate variables
print "ชื่อ: $name\n";               # ชื่อ: สมชาย
print "อายุ: $age ปี\n";             # อายุ: 25 ปี
print "ผลไม้แรก: $fruits[0]\n";      # ผลไม้แรก: แอปเปิ้ล
print "เมือง: $info{city}\n";         # เมือง: กรุงเทพ

# Single-quoted — ไม่ interpolate
print 'ชื่อ: $name\n';               # ชื่อ: $name\n (literal)

# การ interpolate ที่ซับซ้อน
my $item = "apple";
print "I like ${item}s\n";           # I like apples (ใช้ {} ป้องกัน ambiguity)

# Array ใน string
print "fruits: @fruits\n";           # fruits: แอปเปิ้ล กล้วย ส้ม

# Expression ใน string — ต้องใช้ @{[...]}
print "จำนวน: @{[scalar @fruits]} ชนิด\n";    # จำนวน: 3 ชนิด
print "1+1 = @{[1+1]}\n";           # 1+1 = 2

# ใช้ . สำหรับ concatenation
my $greeting = "สวัสดี " . $name . "!";
print "$greeting\n";
```

---

## Step 17: ตัวดำเนินการ String

```perl
#!/usr/bin/perl
use strict;
use warnings;

# String Concatenation ด้วย .
my $first = "Hello";
my $last  = "World";
my $full  = $first . ", " . $last . "!";
print "$full\n";  # Hello, World!

# String Repetition ด้วย x
my $line = "-" x 50;
print "$line\n";  # --------------------------------------------------

my $repeated = "abc" x 3;
print "$repeated\n";  # abcabcabc

# Array repetition ด้วย x
my @arr = (1, 2, 3) x 3;
print "@arr\n";  # 1 2 3 1 2 3 1 2 3

# String operators
my $a = "Hello";
my $b = "World";

# เปรียบเทียบ strings
if ($a lt $b) { print "$a < $b\n" }  # lt = less than
if ($a le $b) { print "$a <= $b\n" } # le = less or equal
if ($a gt $b) { print "$a > $b\n" }  # gt = greater than
if ($a ge $b) { print "$a >= $b\n" } # ge = greater or equal
if ($a eq $b) { print "$a = $b\n" }  # eq = equal
if ($a ne $b) { print "$a != $b\n" } # ne = not equal

# cmp — string comparison (-1, 0, 1)
my $result = $a cmp $b;
print "cmp result: $result\n";

# String functions
print length("Hello"), "\n";         # 5
print uc("hello"), "\n";             # HELLO
print lc("HELLO"), "\n";             # hello
print ucfirst("hello world"), "\n";  # Hello world
print lcfirst("HELLO WORLD"), "\n";  # hELLO WORLD
print reverse("Hello"), "\n";        # olleH
```

---

## Step 18: Numbers และการคำนวณ

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Integer operations
my $a = 10;
my $b = 3;

print "บวก: ",   $a + $b,  "\n";   # 13
print "ลบ: ",    $a - $b,  "\n";   # 7
print "คูณ: ",   $a * $b,  "\n";   # 30
print "หาร: ",   $a / $b,  "\n";   # 3.33333...
print "หารเอาเศษ: ", $a % $b, "\n"; # 1
print "ยกกำลัง: ",   $a ** $b, "\n"; # 1000

# Integer division
use POSIX qw(floor);
print "หารปัด: ", floor($a / $b), "\n";  # 3

# Floating point
my $pi = 3.14159265358979;
my $r  = 5.0;
my $area = $pi * $r ** 2;
printf "พื้นที่วงกลม r=%g: %.4f\n", $r, $area;

# Scientific notation
my $big   = 1.5e10;   # 15,000,000,000
my $small = 2.5e-3;   # 0.0025
print "ใหญ่: $big\n";
print "เล็ก: $small\n";

# Number formats
my $hex  = 0xFF;    # 255 (hexadecimal)
my $oct  = 0377;    # 255 (octal)
my $bin  = 0b11111111;  # 255 (binary)
print "hex: $hex\n";
print "oct: $oct\n";
print "bin: $bin\n";

# Numeric comparison
print "เปรียบเทียบ:\n";
print "10 == 10: ", (10 == 10 ? "จริง" : "เท็จ"), "\n";
print "10 != 5: ",  (10 != 5  ? "จริง" : "เท็จ"), "\n";
print "10 > 5: ",   (10 > 5   ? "จริง" : "เท็จ"), "\n";
print "10 < 5: ",   (10 < 5   ? "จริง" : "เท็จ"), "\n";
print "10 >= 10: ", (10 >= 10 ? "จริง" : "เท็จ"), "\n";
print "10 <= 10: ", (10 <= 10 ? "จริง" : "เท็จ"), "\n";

# <=> spaceship operator
my $cmp = 10 <=> 20;
print "10 <=> 20 = $cmp\n";   # -1
$cmp = 20 <=> 10;
print "20 <=> 10 = $cmp\n";   # 1
$cmp = 10 <=> 10;
print "10 <=> 10 = $cmp\n";   # 0
```

---

## Step 19: Boolean Logic

```perl
#!/usr/bin/perl
use strict;
use warnings;

# ใน Perl ไม่มี boolean type แต่ทุกค่ามี "truthiness"

# FALSE values (ค่าที่เป็น false)
my @false_values = (
    0,        # ตัวเลขศูนย์
    "",       # string ว่าง
    "0",      # string "0"
    undef,    # undefined
    # ()      # empty list (ใน scalar context)
);

# TRUE values (ทุกอย่างที่ไม่ใช่ false)
my @true_values = (
    1,          # ตัวเลขที่ไม่ใช่ศูนย์
    "hello",    # string ที่ไม่ว่าง (และไม่ใช่ "0")
    "00",       # string "00" (ไม่ใช่ "0")
    " ",        # space (ไม่ใช่ string ว่าง)
    0.1,        # ทศนิยม
);

# ทดสอบ
foreach my $val (0, 1, "", "0", "hello", undef) {
    my $truth = $val ? "true" : "false";
    my $display = defined($val) ? "\"$val\"" : "undef";
    print "$display เป็น: $truth\n";
}

# Logical Operators
my $x = 1;
my $y = 0;

print "AND: ", ($x && $y) ? "true" : "false", "\n";   # false
print "OR:  ", ($x || $y) ? "true" : "false", "\n";   # true
print "NOT: ", (!$x) ? "true" : "false", "\n";        # false

# and, or, not (ลำดับความสำคัญต่ำกว่า)
if ($x and $y) { print "both true\n" }
if ($x or $y)  { print "at least one true\n" }
if (not $y)    { print "y is false\n" }

# Short-circuit evaluation
my $name = undef;
my $default = $name || "ไม่ระบุ";   # ถ้า $name เป็น false ใช้ "ไม่ระบุ"
print "ชื่อ: $default\n";

# Defined-or operator (//) — Perl 5.10+
my $val = undef // "ค่าเริ่มต้น";
print "ค่า: $val\n";  # ค่าเริ่มต้น

$val = 0 // "ค่าเริ่มต้น";
print "ค่า: $val\n";  # 0 (0 เป็น defined แม้จะเป็น false)
```

---

## Step 20: โปรแกรมรวม — ระบบคำนวณง่ายๆ

```perl
#!/usr/bin/perl
#
# โปรแกรม: simple_calculator.pl
# ระบบคิดเลขพื้นฐาน
#

use strict;
use warnings;

# ========================================
# แสดง header
# ========================================
sub show_header {
    print "\n";
    print "=" x 40 . "\n";
    print "   เครื่องคิดเลข Perl\n";
    print "=" x 40 . "\n\n";
}

# ========================================
# แสดง menu
# ========================================
sub show_menu {
    print "เลือกการคำนวณ:\n";
    print "1. บวก (+)\n";
    print "2. ลบ (-)\n";
    print "3. คูณ (*)\n";
    print "4. หาร (/)\n";
    print "5. ยกกำลัง (**)\n";
    print "6. หารเอาเศษ (%)\n";
    print "0. ออก\n";
    print "\nเลือก: ";
}

# ========================================
# รับตัวเลข
# ========================================
sub get_number {
    my ($prompt) = @_;
    while (1) {
        print $prompt;
        my $input = <STDIN>;
        chomp $input;
        if ($input =~ /^-?\d+\.?\d*$/) {
            return $input;
        }
        print "กรุณากรอกตัวเลข!\n";
    }
}

# ========================================
# คำนวณ
# ========================================
sub calculate {
    my ($op, $a, $b) = @_;
    
    if ($op == 1) { return ($a + $b, "+") }
    if ($op == 2) { return ($a - $b, "-") }
    if ($op == 3) { return ($a * $b, "*") }
    if ($op == 4) {
        if ($b == 0) { return ("หารด้วยศูนย์ไม่ได้!", "/") }
        return ($a / $b, "/");
    }
    if ($op == 5) { return ($a ** $b, "**") }
    if ($op == 6) {
        if ($b == 0) { return ("หารด้วยศูนย์ไม่ได้!", "%") }
        return ($a % $b, "%");
    }
    return ("ไม่รู้จักการคำนวณ", "?");
}

# ========================================
# โปรแกรมหลัก
# ========================================
show_header();

while (1) {
    show_menu();
    
    my $choice = <STDIN>;
    chomp $choice;
    
    last if $choice == 0;
    
    unless ($choice =~ /^[1-6]$/) {
        print "กรุณาเลือก 0-6\n";
        next;
    }
    
    print "\n";
    my $num1 = get_number("กรอกตัวเลขแรก: ");
    my $num2 = get_number("กรอกตัวเลขที่สอง: ");
    
    my ($result, $operator) = calculate($choice, $num1, $num2);
    
    print "\n" . "-" x 30 . "\n";
    printf "  %s %s %s = %s\n", $num1, $operator, $num2, $result;
    print "-" x 30 . "\n\n";
}

print "\nขอบคุณที่ใช้งาน!\n\n";
```

### รันโปรแกรม

```bash
perl simple_calculator.pl
```

### ผลลัพธ์ตัวอย่าง

```
========================================
   เครื่องคิดเลข Perl
========================================

เลือกการคำนวณ:
1. บวก (+)
2. ลบ (-)
3. คูณ (*)
4. หาร (/)
5. ยกกำลัง (**)
6. หารเอาเศษ (%)
0. ออก

เลือก: 1

กรอกตัวเลขแรก: 15
กรอกตัวเลขที่สอง: 27

------------------------------
  15 + 27 = 42
------------------------------
```

---

## แบบฝึกหัด Part 02

### แบบฝึกหัดที่ 1: Formatted Output

สร้างโปรแกรมแสดงตารางคูณแบบสวยงาม:

```perl
#!/usr/bin/perl
use strict;
use warnings;

# แสดงตารางคูณ 1-12
print "ตารางคูณ\n";
print "=" x 60 . "\n";

# เติมโค้ดของคุณที่นี่
# ควรแสดงเป็น:
#      1   2   3  ...  12
# 1 |  1   2   3  ...  12
# 2 |  2   4   6  ...  24
# ...
```

### แบบฝึกหัดที่ 2: User Profile

สร้างโปรแกรมรับข้อมูลและแสดงผล:
- ชื่อ, นามสกุล, อายุ, อาชีพ, เงินเดือน
- แสดงผลเป็น formatted report

### แบบฝึกหัดที่ 3: String Manipulation

```perl
#!/usr/bin/perl
use strict;
use warnings;

my $sentence = "the quick brown fox jumps over the lazy dog";

# ทำสิ่งต่อไปนี้:
# 1. แปลงเป็น uppercase
# 2. แปลง title case (ทุก word ขึ้นต้นด้วย capital)
# 3. นับจำนวน word
# 4. หาคำที่ยาวที่สุด
# 5. นับจำนวนตัวอักษรทั้งหมด (ไม่รวม space)
```

---

## สรุป Part 02

ใน Part นี้คุณได้เรียนรู้:
- ✅ โครงสร้างโปรแกรม Perl อย่างละเอียด
- ✅ Pragmas: strict, warnings, utf8, feature
- ✅ การแสดงผล: print, say, printf, sprintf
- ✅ การรับ Input จากผู้ใช้
- ✅ Escape sequences
- ✅ String interpolation
- ✅ String operators
- ✅ Numbers และการคำนวณ
- ✅ Boolean logic
- ✅ โปรแกรมเครื่องคิดเลข

**ถัดไป: [Part 03 — Variables และ Data Types](part_03.md)**
