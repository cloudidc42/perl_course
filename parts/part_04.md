# Part 04: Scalars — ตัวแปรพื้นฐาน
## Steps 31-40: เจาะลึก Scalar Variables

---

## Step 31: Scalar Context

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Scalar context บังคับให้ expression คืนค่าเดียว

my @array = (1, 2, 3, 4, 5);

# Array ใน scalar context = จำนวนสมาชิก
my $count = @array;
print "count: $count\n";  # 5

# บังคับ scalar context ด้วย scalar()
print "length: ", scalar(@array), "\n";  # 5

# Conditional ใช้ scalar context
if (@array) {
    print "array ไม่ว่าง\n";
}

# ตัวอย่างที่สับสนได้
my @a = (3, 1, 4);
my @b = (1, 5, 9);

# Array ใน numeric context
my $sum = @a + @b;
print "sum of lengths: $sum\n";  # 3 + 3 = 6 (ไม่ใช่ผลรวมของสมาชิก!)

# ถ้าต้องการผลรวมจริงๆ
my $real_sum = 0;
$real_sum += $_ for (@a, @b);
print "real sum: $real_sum\n";  # 3+1+4+1+5+9 = 23
```

---

## Step 32: String Functions ละเอียด

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# length
# =====================
my $str = "Hello, สวัสดี!";
print "length: ", length($str), "\n";

# =====================
# substr — substring
# =====================
my $text = "Hello, World!";

# substr($string, $offset, $length)
print substr($text, 0, 5), "\n";    # Hello
print substr($text, 7, 5), "\n";    # World
print substr($text, -6), "\n";       # orld! (จากท้าย)
print substr($text, -6, 5), "\n";    # orld

# แก้ไขด้วย substr
my $s = "Hello, World!";
substr($s, 7, 5) = "Perl";
print "$s\n";  # Hello, Perl!!

# 4-argument substr
my $s2 = "Hello, World!";
my $old = substr($s2, 7, 5, "Universe");
print "replaced: $old\n";  # World
print "result: $s2\n";     # Hello, Universe!

# =====================
# index / rindex
# =====================
my $str2 = "Hello World Hello";

my $pos = index($str2, "Hello");
print "first 'Hello' at: $pos\n";   # 0

$pos = index($str2, "Hello", 1);    # ค้นหาจาก position 1
print "next 'Hello' at: $pos\n";    # 12

$pos = rindex($str2, "Hello");      # ค้นหาจากท้าย
print "last 'Hello' at: $pos\n";    # 12

$pos = index($str2, "xyz");
print "not found: $pos\n";          # -1

# =====================
# uc, lc, ucfirst, lcfirst
# =====================
print uc("hello world"), "\n";         # HELLO WORLD
print lc("HELLO WORLD"), "\n";         # hello world
print ucfirst("hello world"), "\n";    # Hello world
print lcfirst("HELLO WORLD"), "\n";    # hELLO WORLD

# Title case (ทุก word)
my $title = join(" ", map { ucfirst(lc($_)) } split(/\s+/, "hello world foo"));
print "$title\n";  # Hello World Foo

# =====================
# reverse
# =====================
my $rev = reverse("Hello");
print "$rev\n";  # olleH

my @rev_arr = reverse(1, 2, 3, 4, 5);
print "@rev_arr\n";  # 5 4 3 2 1

# =====================
# split
# =====================
my $csv  = "apple,banana,cherry,date";
my @fruits = split(/,/, $csv);
print "@fruits\n";          # apple banana cherry date
print scalar @fruits, "\n"; # 4

# Split ด้วย whitespace
my $words = "  hello   world   foo  ";
my @w = split(/\s+/, $words);
print "@w\n";  # "" hello world foo (มี empty string ตรงหน้า)

# ตัด empty string ตรงหน้า
@w = split(' ', $words);  # ใช้ ' ' (space literal) ตัด whitespace ด้านหน้า
print "@w\n";  # hello world foo

# จำกัดจำนวน parts
my @parts = split(/,/, "a,b,c,d,e", 3);
print "@parts\n";  # a b c,d,e (แค่ 3 parts)

# =====================
# join
# =====================
my @items = ("apple", "banana", "cherry");
my $str3 = join(", ", @items);
print "$str3\n";  # apple, banana, cherry

my $path = join("/", "usr", "local", "bin");
print "$path\n";  # usr/local/bin

# =====================
# chomp / chop
# =====================
my $line = "Hello\n";
my $removed = chomp $line;  # ลบ newline, คืนจำนวน chars ที่ลบ
print "'$line' (removed $removed chars)\n";

my $str4 = "Hello!";
my $last = chop $str4;
print "'$str4' (removed '$last')\n";

# =====================
# trim (ไม่มีในตัว Perl แต่ทำได้ง่าย)
# =====================
sub trim {
    my $s = shift;
    $s =~ s/^\s+|\s+$//g;
    return $s;
}

sub ltrim {
    my $s = shift;
    $s =~ s/^\s+//;
    return $s;
}

sub rtrim {
    my $s = shift;
    $s =~ s/\s+$//;
    return $s;
}

my $padded = "   Hello, World!   ";
print "'", trim($padded), "'\n";   # 'Hello, World!'
print "'", ltrim($padded), "'\n";  # 'Hello, World!   '
print "'", rtrim($padded), "'\n";  # '   Hello, World!'

# Perl 5.36+ มี built-in trim
# use builtin 'trim';
```

---

## Step 33: Number Functions

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(floor ceil fmod);
use List::Util qw(min max sum);

# =====================
# abs — absolute value
# =====================
print abs(-42), "\n";   # 42
print abs(42), "\n";    # 42
print abs(-3.14), "\n"; # 3.14

# =====================
# int — truncate to integer
# =====================
print int(3.9), "\n";    # 3 (ไม่ปัดขึ้น!)
print int(-3.9), "\n";   # -3 (truncate toward zero)
print int(3.14159), "\n"; # 3

# =====================
# POSIX functions
# =====================
print floor(3.9), "\n";   # 3 (ปัดลง)
print floor(-3.9), "\n";  # -4
print ceil(3.1), "\n";    # 4 (ปัดขึ้น)
print ceil(-3.1), "\n";   # -3

# =====================
# sqrt — square root
# =====================
print sqrt(16), "\n";    # 4
print sqrt(2), "\n";     # 1.4142...

# =====================
# log, exp
# =====================
print log(1), "\n";     # 0 (natural log)
print log(exp(1)), "\n"; # 1
use POSIX qw(log10);
print log10(100), "\n";  # 2

# =====================
# Power
# =====================
print 2 ** 10, "\n";   # 1024
print 9 ** 0.5, "\n";  # 3 (square root)

# =====================
# Random numbers
# =====================
srand(42);  # seed

# random float [0, 1)
my $rand = rand();
printf "%.4f\n", $rand;

# random int [0, 10)
my $randint = int(rand(10));
print "$randint\n";

# random int [1, 10]
my $dice = int(rand(6)) + 1;
print "dice: $dice\n";

# =====================
# Number formatting
# =====================
use POSIX qw(floor);

sub format_number {
    my ($num, $decimal) = @_;
    $decimal //= 0;
    
    # format ทศนิยม
    my $formatted = sprintf("%.${decimal}f", $num);
    
    # เพิ่ม comma separator
    $formatted =~ s/\d{1,3}(?=(\d{3})+(?!\d))/$&,/g;
    
    return $formatted;
}

print format_number(1234567.89, 2), "\n";  # 1,234,567.89
print format_number(1000000), "\n";         # 1,000,000

# =====================
# Math functions ผ่าน POSIX
# =====================
use POSIX qw(sin cos tan asin acos atan2 pow fabs);

my $pi = 4 * atan2(1, 1);  # คำนวณ pi
printf "pi = %.10f\n", $pi;

# Trigonometry
printf "sin(90°) = %.4f\n", sin($pi/2);
printf "cos(0°)  = %.4f\n", cos(0);

# =====================
# List::Util
# =====================
my @nums = (3, 1, 4, 1, 5, 9, 2, 6, 5, 3);

print "min: ", min(@nums), "\n";      # 1
print "max: ", max(@nums), "\n";      # 9
print "sum: ", sum(@nums), "\n";      # 39

use List::Util qw(reduce any all none first);

my $product = reduce { $a * $b } @nums;
print "product: $product\n";

my $has_zero = any { $_ == 0 } @nums;
print "has zero: ", $has_zero ? "yes" : "no", "\n";  # no

my $all_positive = all { $_ > 0 } @nums;
print "all positive: ", $all_positive ? "yes" : "no", "\n";  # yes

my $first_even = first { $_ % 2 == 0 } @nums;
print "first even: $first_even\n";  # 4
```

---

## Step 34: String ขั้นสูง — Regular Expression พื้นฐาน

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Match operator m//
# =====================
my $text = "Hello, World! This is Perl.";

if ($text =~ /World/) {
    print "Found 'World'\n";
}

if ($text =~ /perl/i) {  # i = case-insensitive
    print "Found 'perl' (case insensitive)\n";
}

# Match ที่มีการ capture
if ($text =~ /(\w+), (\w+)/) {
    print "First: $1\n";   # Hello
    print "Second: $2\n";  # World
}

# =====================
# Substitution operator s///
# =====================
my $str = "Hello, World!";
(my $new = $str) =~ s/World/Perl/;
print "$new\n";  # Hello, Perl!

# Global substitution
my $s = "aababab";
$s =~ s/a/X/g;  # g = global
print "$s\n";   # XXbXbXb

# Case-insensitive substitution
$s = "Hello HELLO hello";
$s =~ s/hello/Hi/gi;
print "$s\n";  # Hi Hi Hi

# =====================
# Transliteration tr///
# =====================
my $tr_str = "Hello, World!";

(my $tr = $tr_str) =~ tr/a-z/A-Z/;
print "$tr\n";  # HELLO, WORLD!

my $count = ($tr_str =~ tr/l//);  # นับจำนวน 'l'
print "l count: $count\n";  # 3

# ROT13
my $encoded = $tr_str;
$encoded =~ tr/A-Za-z/N-ZA-Mn-za-m/;
print "ROT13: $encoded\n";

# =====================
# String repetition patterns
# =====================

# สร้าง separator
sub separator {
    my ($char, $width) = @_;
    $char //= '-';
    $width //= 40;
    return $char x $width;
}

print separator(), "\n";
print separator('=', 50), "\n";
print separator('*'), "\n";

# Center text
sub center {
    my ($text, $width, $fill) = @_;
    $width //= 40;
    $fill  //= ' ';
    my $padding = $width - length($text);
    return $padding <= 0 ? $text : 
           $fill x int($padding/2) . $text . $fill x ($padding - int($padding/2));
}

my $title = " HELLO WORLD ";
print center($title, 40, "="), "\n";
# ===========HELLO WORLD ============
```

---

## Step 35: Heredoc ขั้นสูง

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Basic heredoc
# =====================
my $text = <<END;
This is a
multi-line string
END
print $text;

# =====================
# Indented heredoc (Perl 5.26+)
# =====================
my $indented = <<~END;
    This heredoc
    has indentation
    removed automatically
    END
print $indented;

# =====================
# Heredoc with interpolation
# =====================
my $name    = "Somchai";
my $country = "Thailand";

my $greeting = <<END;
สวัสดีคุณ $name
คุณมาจาก $country ใช่ไหม?
ยินดีต้อนรับ!
END
print $greeting;

# =====================
# Heredoc ไม่ interpolate (single quote)
# =====================
my $no_interp = <<'END';
No $interpolation here
$name and @array stay as-is
Backslash \n is literal
END
print $no_interp;

# =====================
# Heredoc กับ indented delimiter
# =====================
my $html = <<~HTML;
    <html>
    <body>
        <h1>Hello</h1>
    </body>
    </html>
    HTML
print $html;

# =====================
# Heredoc ใน expressions
# =====================
my @lines = split /\n/, <<END;
line one
line two
line three
END

foreach my $line (@lines) {
    print "- $line\n";
}

# =====================
# Heredoc กับ functions
# =====================
sub process_text {
    my $text = shift;
    $text =~ s/^\s+//mg;  # ลบ whitespace นำหน้าทุกบรรทัด
    return $text;
}

my $processed = process_text(<<END);
    This text
    has leading
    whitespace
    END
print $processed;
```

---

## Step 36: sprintf และ Format Strings

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Number formatting
# =====================

# ทศนิยม
printf "%.2f\n", 3.14159;      # 3.14
printf "%.0f\n", 3.14159;      # 3
printf "%10.2f\n", 3.14159;    # "      3.14" (width 10)
printf "%-10.2f|\n", 3.14159;  # "3.14      |" (left-align)

# Integer
printf "%d\n", 42;             # 42
printf "%10d\n", 42;           # "        42"
printf "%-10d|\n", 42;         # "42        |"
printf "%010d\n", 42;          # "0000000042"
printf "%+d\n", 42;            # "+42"
printf "%+d\n", -42;           # "-42"

# Hex, Oct, Binary
printf "%x\n", 255;            # ff
printf "%X\n", 255;            # FF
printf "%#x\n", 255;           # 0xff
printf "%o\n", 8;              # 10
printf "%b\n", 10;             # 1010
printf "%#b\n", 10;            # 0b1010

# String
printf "%s\n", "hello";        # hello
printf "%10s\n", "hello";      # "     hello"
printf "%-10s|\n", "hello";    # "hello     |"
printf "%.3s\n", "hello";      # hel (truncate)

# =====================
# สร้าง Table
# =====================
my @data = (
    ["สมชาย",    25, "กรุงเทพ",     50000],
    ["สมหญิง",   30, "เชียงใหม่",    60000],
    ["วิชัย",    28, "ภูเก็ต",      45000],
    ["สุดา",     35, "ขอนแก่น",     55000],
    ["ประยุทธ์", 40, "นครราชสีมา", 70000],
);

my $fmt = "%-15s %5s %-15s %12s\n";
printf $fmt, "ชื่อ", "อายุ", "เมือง", "เงินเดือน";
print "-" x 52 . "\n";

my $total = 0;
foreach my $row (@data) {
    printf "%-15s %5d %-15s %,12.0f\n", @$row;
    $total += $row->[3];
}
print "-" x 52 . "\n";
printf "%-15s %5s %-15s %12.0f\n", "รวม", "", "", $total;

# =====================
# Dynamic format strings
# =====================
sub table_row {
    my ($widths, @values) = @_;
    my $fmt = join(" ", map { "%-${_}s" } @$widths) . "\n";
    printf $fmt, @values;
}

my @widths = (15, 5, 15, 12);
table_row(\@widths, "Name", "Age", "City", "Salary");
table_row(\@widths, "Somchai", 25, "Bangkok", 50000);
```

---

## Step 37: String Manipulation Advanced

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# String เป็น Character Array
# =====================
my $str = "Hello";
my @chars = split //, $str;
print "@chars\n";   # H e l l o

# กลับด้าน
my @reversed = reverse @chars;
print "@reversed\n";  # o l l e H

# Join กลับ
print join("", @reversed), "\n";  # olleH

# =====================
# Encode / Decode
# =====================
# URL encoding
sub url_encode {
    my $str = shift;
    $str =~ s/([^A-Za-z0-9\-_.~])/sprintf("%%%02X", ord($1))/ge;
    return $str;
}

sub url_decode {
    my $str = shift;
    $str =~ s/%([0-9A-Fa-f]{2})/chr(hex($1))/ge;
    return $str;
}

my $url = "Hello, World! สวัสดี";
my $encoded = url_encode($url);
print "encoded: $encoded\n";
print "decoded: ", url_decode($encoded), "\n";

# HTML encoding
sub html_encode {
    my $str = shift;
    $str =~ s/&/&amp;/g;
    $str =~ s/</&lt;/g;
    $str =~ s/>/&gt;/g;
    $str =~ s/"/&quot;/g;
    $str =~ s/'/&#39;/g;
    return $str;
}

my $html = '<script>alert("XSS")</script>';
print html_encode($html), "\n";

# =====================
# String comparison
# =====================
my @words = ("banana", "apple", "cherry", "date", "elderberry");
my @sorted = sort @words;
print "@sorted\n";  # alphabetical order

my @by_length = sort { length($a) <=> length($b) } @words;
print "@by_length\n";  # sorted by length

my @by_length_alpha = sort { length($a) <=> length($b) || $a cmp $b } @words;
print "@by_length_alpha\n";

# =====================
# String padding functions
# =====================
sub pad_left {
    my ($str, $width, $char) = @_;
    $char //= ' ';
    my $pad_len = $width - length($str);
    return $pad_len > 0 ? ($char x $pad_len) . $str : $str;
}

sub pad_right {
    my ($str, $width, $char) = @_;
    $char //= ' ';
    my $pad_len = $width - length($str);
    return $pad_len > 0 ? $str . ($char x $pad_len) : $str;
}

sub pad_center {
    my ($str, $width, $char) = @_;
    $char //= ' ';
    my $pad_len = $width - length($str);
    return $str if $pad_len <= 0;
    my $left  = int($pad_len / 2);
    my $right = $pad_len - $left;
    return ($char x $left) . $str . ($char x $right);
}

print "|", pad_left("Hello", 10), "|\n";      # |     Hello|
print "|", pad_right("Hello", 10), "|\n";     # |Hello     |
print "|", pad_center("Hello", 10), "|\n";    # |  Hello   |
print "|", pad_left("Hello", 10, "0"), "|\n"; # |00000Hello|
```

---

## Step 38: Numbers ขั้นสูง

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(floor ceil);
use List::Util qw(sum min max reduce);
use Scalar::Util qw(looks_like_number);

# =====================
# ตรวจสอบว่าเป็นตัวเลข
# =====================
my @tests = (42, "42", "3.14", "0xFF", "1e5", "hello", undef, "");

foreach my $val (@tests) {
    my $display = defined($val) ? qq("$val") : "undef";
    printf "%-10s is %s\n", $display,
        looks_like_number($val) ? "a number" : "NOT a number";
}

# =====================
# Number base conversion
# =====================
sub to_binary {
    return sprintf "%b", shift;
}

sub to_hex {
    return sprintf "%x", shift;
}

sub to_octal {
    return sprintf "%o", shift;
}

sub from_binary {
    return oct("0b" . shift);
}

sub from_hex {
    return hex(shift);
}

print "255 in binary: ", to_binary(255), "\n";   # 11111111
print "255 in hex: ", to_hex(255), "\n";         # ff
print "255 in octal: ", to_octal(255), "\n";     # 377

print "from binary 11111111: ", from_binary("11111111"), "\n";  # 255
print "from hex ff: ", from_hex("ff"), "\n";                    # 255

# =====================
# Statistics functions
# =====================
sub mean {
    my @nums = @_;
    return 0 unless @nums;
    return sum(@nums) / @nums;
}

sub median {
    my @sorted = sort { $a <=> $b } @_;
    my $n = @sorted;
    if ($n % 2) {
        return $sorted[$n/2];
    } else {
        return ($sorted[$n/2-1] + $sorted[$n/2]) / 2;
    }
}

sub mode {
    my @nums = @_;
    my %freq;
    $freq{$_}++ for @nums;
    my $max_freq = max(values %freq);
    return grep { $freq{$_} == $max_freq } sort keys %freq;
}

sub variance {
    my @nums = @_;
    my $mean = mean(@nums);
    my $sum_sq = sum(map { ($_ - $mean) ** 2 } @nums);
    return $sum_sq / @nums;
}

sub stddev {
    return sqrt(variance(@_));
}

my @data = (4, 8, 6, 5, 3, 2, 8, 9, 2, 5);

printf "Mean:   %.2f\n", mean(@data);
printf "Median: %.2f\n", median(@data);
printf "Mode:   %s\n", join(", ", mode(@data));
printf "Stddev: %.4f\n", stddev(@data);
printf "Min:    %d\n", min(@data);
printf "Max:    %d\n", max(@data);
printf "Range:  %d\n", max(@data) - min(@data);
```

---

## Step 39: Context ขั้นสูง

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# List vs Scalar context
# =====================
my @array = (1, 2, 3, 4, 5);

# list context — คืน list
my @copy  = @array;
my ($first, @rest) = @array;

# scalar context — คืนจำนวนสมาชิก
my $count = @array;  # 5
my $last = $array[-1];  # 5

# =====================
# Forcing context
# =====================

# Force scalar
print scalar(@array), "\n";  # 5

# Force list (ใช้ wantarray ใน sub)
sub context_test {
    if (wantarray) {
        return (1, 2, 3);    # ถ้าถูกเรียกใน list context
    } else {
        return "scalar";     # ถ้าถูกเรียกใน scalar context
    }
}

my @list_result   = context_test();
my $scalar_result = context_test();

print "list: @list_result\n";      # 1 2 3
print "scalar: $scalar_result\n";  # scalar

# =====================
# localtime — บอก context
# =====================

# Scalar context
my $time_str = localtime();
print "string: $time_str\n";

# List context
my ($sec, $min, $hour, $mday, $mon, $year, $wday, $yday, $isdst) = localtime();
$year += 1900;
$mon  += 1;
printf "datetime: %04d-%02d-%02d %02d:%02d:%02d\n",
    $year, $mon, $mday, $hour, $min, $sec;

# =====================
# Hash ใน scalar context
# =====================
my %hash = (a => 1, b => 2, c => 3);

# scalar context = "X/Y" (used/total buckets)
# Perl 5.26+: = number of keys
my $h_count = scalar %hash;  
print "hash scalar: $h_count\n";

# more reliable:
my $key_count = scalar keys %hash;
print "key count: $key_count\n";  # 3
```

---

## Step 40: โปรแกรมสรุป — String และ Number Processing

```perl
#!/usr/bin/perl
#
# โปรแกรม: text_processor.pl
# ระบบประมวลผลข้อความและตัวเลข
#

use strict;
use warnings;
use POSIX qw(floor);
use List::Util qw(sum min max);

# =====================
# Text analysis
# =====================

sub analyze_text {
    my ($text) = @_;
    
    my %stats;
    
    # Character count
    $stats{chars}       = length($text);
    $stats{chars_no_ws} = () = $text =~ /\S/g;
    
    # Word count
    my @words           = split /\s+/, $text;
    @words              = grep { length($_) } @words;
    $stats{words}       = scalar @words;
    
    # Line count
    my @lines           = split /\n/, $text;
    $stats{lines}       = scalar @lines;
    
    # Sentence count (approximate)
    my @sentences       = split /[.!?]+/, $text;
    @sentences          = grep { /\S/ } @sentences;
    $stats{sentences}   = scalar @sentences;
    
    # Unique words
    my %unique;
    $unique{lc($_)}++ for @words;
    $stats{unique_words} = scalar keys %unique;
    
    # Avg word length
    if (@words) {
        my $total_len = sum(map { length($_) } @words);
        $stats{avg_word_len} = $total_len / @words;
    }
    
    # Most common words
    my @sorted_words = sort { $unique{$b} <=> $unique{$a} } keys %unique;
    $stats{top_words} = [map { [$_, $unique{$_}] } @sorted_words[0..4]];
    
    return %stats;
}

# =====================
# Number cruncher
# =====================

sub crunch_numbers {
    my @nums = @_;
    
    return unless @nums;
    
    my %stats;
    $stats{count}  = scalar @nums;
    $stats{sum}    = sum(@nums);
    $stats{min}    = min(@nums);
    $stats{max}    = max(@nums);
    $stats{range}  = $stats{max} - $stats{min};
    $stats{mean}   = $stats{sum} / $stats{count};
    
    # Variance and stddev
    my $sq_sum = sum(map { ($_ - $stats{mean}) ** 2 } @nums);
    $stats{variance} = $sq_sum / $stats{count};
    $stats{stddev}   = sqrt($stats{variance});
    
    # Median
    my @sorted = sort { $a <=> $b } @nums;
    my $n = $stats{count};
    $stats{median} = $n % 2 
        ? $sorted[$n/2] 
        : ($sorted[$n/2-1] + $sorted[$n/2]) / 2;
    
    return %stats;
}

# =====================
# Main program
# =====================

my $sample_text = <<'TEXT';
The quick brown fox jumps over the lazy dog.
The dog was not amused.
The fox was quick and clever.
The dog was lazy but happy.
TEXT

print "=" x 50 . "\n";
print " TEXT ANALYSIS\n";
print "=" x 50 . "\n\n";

my %text_stats = analyze_text($sample_text);

printf "%-20s: %d\n", "Total characters",  $text_stats{chars};
printf "%-20s: %d\n", "Non-whitespace",    $text_stats{chars_no_ws};
printf "%-20s: %d\n", "Words",             $text_stats{words};
printf "%-20s: %d\n", "Unique words",      $text_stats{unique_words};
printf "%-20s: %d\n", "Lines",             $text_stats{lines};
printf "%-20s: %d\n", "Sentences",         $text_stats{sentences};
printf "%-20s: %.2f\n", "Avg word length", $text_stats{avg_word_len};

print "\nTop 5 words:\n";
foreach my $word_pair (@{$text_stats{top_words}}) {
    printf "  %-15s: %d\n", $word_pair->[0], $word_pair->[1];
}

print "\n" . "=" x 50 . "\n";
print " NUMBER ANALYSIS\n";
print "=" x 50 . "\n\n";

my @numbers = (23, 45, 12, 67, 34, 89, 56, 78, 11, 90, 45, 23);
my %num_stats = crunch_numbers(@numbers);

printf "Numbers: %s\n\n", join(", ", @numbers);
printf "%-12s: %d\n",   "Count",    $num_stats{count};
printf "%-12s: %d\n",   "Sum",      $num_stats{sum};
printf "%-12s: %d\n",   "Min",      $num_stats{min};
printf "%-12s: %d\n",   "Max",      $num_stats{max};
printf "%-12s: %d\n",   "Range",    $num_stats{range};
printf "%-12s: %.2f\n", "Mean",     $num_stats{mean};
printf "%-12s: %.2f\n", "Median",   $num_stats{median};
printf "%-12s: %.4f\n", "Std Dev",  $num_stats{stddev};
```

---

## แบบฝึกหัด Part 04

### แบบฝึกหัดที่ 1: String Functions

ทดลองฟังก์ชัน string ทั้งหมดที่เรียนมา:
- `length`, `substr`, `index`, `rindex`
- `uc`, `lc`, `ucfirst`
- `split`, `join`
- `reverse`

### แบบฝึกหัดที่ 2: Palindrome Checker

```perl
#!/usr/bin/perl
use strict;
use warnings;

sub is_palindrome {
    my $str = lc(shift);
    $str =~ s/[^a-z0-9]//g;  # ลบ non-alphanumeric
    return $str eq reverse($str);
}

my @tests = ("racecar", "hello", "A man a plan a canal Panama", "Was it a car or a cat I saw");
foreach my $test (@tests) {
    printf "'%s' %s a palindrome\n", $test, is_palindrome($test) ? "IS" : "is NOT";
}
```

### แบบฝึกหัดที่ 3: Number Base Converter

สร้างโปรแกรมแปลงเลขฐาน 10 เป็นฐาน 2, 8, 16

---

## สรุป Part 04

ใน Part นี้คุณได้เรียนรู้:
- ✅ Scalar context อย่างละเอียด
- ✅ String functions: length, substr, index, split, join
- ✅ String case functions: uc, lc, ucfirst
- ✅ Number functions: abs, int, floor, ceil, sqrt
- ✅ sprintf และการ format strings
- ✅ Heredoc ขั้นสูง
- ✅ Advanced string manipulation
- ✅ Number utilities และ statistics

**ถัดไป: [Part 05 — Arrays](part_05.md)**
