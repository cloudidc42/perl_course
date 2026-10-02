# Part 01: แนะนำ Perl และการติดตั้ง
## Steps 1-10: เริ่มต้นกับโลกของ Perl

---

## Step 1: Perl คืออะไร?

**Perl** (Practical Extraction and Report Language) เป็นภาษาโปรแกรมที่สร้างโดย **Larry Wall** ในปี 1987 มีจุดเด่นที่:

- **ยืดหยุ่นสูง** — "There's more than one way to do it" (TMTOWTDI)
- **เหมาะกับ Text Processing** — การประมวลผลข้อความและไฟล์
- **CGI Programming** — ใช้สร้าง Dynamic Web Pages
- **System Administration** — จัดการระบบ Unix/Linux
- **Bioinformatics** — ประมวลผลข้อมูลทางชีววิทยา

### ประวัติศาสตร์ Perl

```
1987 — Larry Wall สร้าง Perl 1.0
1988 — Perl 2.0 (regex engine ปรับปรุง)
1989 — Perl 3.0 (binary data support)
1991 — Perl 4.0 (Camel Book ฉบับแรก)
1994 — Perl 5.0 (OOP, references, modules)
2000 — Perl 6 เริ่มพัฒนา (ภายหลังกลายเป็น Raku)
2021 — Perl 5.34
2022 — Perl 5.36
2023 — Perl 5.38
```

### Perl ใช้ทำอะไรได้บ้าง?

```perl
# 1. Text Processing
while (<>) {
    s/foo/bar/g;  # แทนที่คำ foo ด้วย bar
    print;
}

# 2. Web CGI
use CGI;
my $q = CGI->new;
print $q->header, $q->start_html("Hello"), 
      $q->h1("สวัสดี Perl!"), $q->end_html;

# 3. File Operations
open(my $fh, '<', 'data.txt') or die "ไม่สามารถเปิดไฟล์: $!";
while (<$fh>) {
    print if /pattern/;
}
close $fh;

# 4. System Admin
use File::Find;
find(sub { print "$File::Find::name\n" if -f }, '/home');

# 5. Database
use DBI;
my $dbh = DBI->connect("dbi:mysql:mydb", "user", "pass");
my $sth = $dbh->prepare("SELECT * FROM users");
$sth->execute;
```

---

## Step 2: ทำไมต้องเรียน Perl?

### จุดแข็งของ Perl

| ด้าน | รายละเอียด |
|------|-----------|
| **Regular Expressions** | regex ที่ทรงพลังที่สุดในบรรดาภาษา scripting |
| **Text Manipulation** | ประมวลผลข้อความได้ยอดเยี่ยม |
| **CPAN** | คลังโมดูลมากกว่า 25,000 โมดูล |
| **Cross-platform** | ทำงานได้บน Unix, Windows, Mac |
| **Web Development** | CGI, Mojolicious, Catalyst, Dancer2 |
| **Legacy Systems** | ระบบเก่าจำนวนมากเขียนด้วย Perl |

### Perl กับภาษาอื่น

```
Perl vs Python:
- Perl: เก่ากว่า, regex ดีกว่า, TMTOWTDI
- Python: อ่านง่ายกว่า, ML/AI ecosystem ดีกว่า

Perl vs Ruby:
- Perl: เร็วกว่า, regex ดีกว่า
- Ruby: OOP สะอาดกว่า, Rails framework

Perl vs PHP:
- Perl: ทรงพลังกว่า, regex ดีกว่า
- PHP: ติดตั้งง่ายกว่าสำหรับ web hosting
```

---

## Step 3: การติดตั้ง Perl

### ตรวจสอบว่ามี Perl อยู่แล้วหรือไม่

```bash
# ตรวจสอบ version
perl --version

# ผลลัพธ์ที่ได้
# This is perl 5, version 38, subversion 2 (v5.38.2)
# ...
```

### Linux (Ubuntu/Debian)

```bash
# ติดตั้ง Perl
sudo apt-get update
sudo apt-get install perl

# ติดตั้ง build tools สำหรับ CPAN modules
sudo apt-get install build-essential

# ติดตั้ง cpanminus (เครื่องมือจัดการ modules)
sudo apt-get install cpanminus

# หรือ install cpanm ผ่าน CPAN
curl -L https://cpanmin.us | perl - --sudo App::cpanminus
```

### Linux (CentOS/RHEL/Fedora)

```bash
# CentOS/RHEL
sudo yum install perl perl-devel perl-CPAN

# Fedora
sudo dnf install perl perl-devel perl-App-cpanminus
```

### macOS

```bash
# macOS มี Perl ติดตั้งมาแล้ว แต่แนะนำให้ติดตั้งใหม่ผ่าน Homebrew
brew install perl

# เพิ่ม Homebrew Perl ใน PATH
echo 'export PATH="/usr/local/opt/perl/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# ติดตั้ง cpanm
curl -L https://cpanmin.us | perl - App::cpanminus
```

### Windows

```bash
# ดาวน์โหลด Strawberry Perl จาก http://strawberryperl.com/
# หรือ ActivePerl จาก https://www.activestate.com/

# หลังติดตั้ง ตรวจสอบ
perl --version

# Strawberry Perl มี cpanm ติดตั้งมาแล้ว
cpanm --version
```

### การใช้ perlbrew (แนะนำ)

```bash
# ติดตั้ง perlbrew — จัดการหลาย version ของ Perl
curl -L https://install.perlbrew.pl | bash

# เพิ่มใน ~/.bashrc หรือ ~/.zshrc
source ~/perl5/perlbrew/etc/bashrc

# ดู Perl versions ที่มี
perlbrew available

# ติดตั้ง Perl version ล่าสุด
perlbrew install perl-5.38.2

# ใช้ Perl version ที่ติดตั้ง
perlbrew switch perl-5.38.2

# ตรวจสอบ
perl --version
```

---

## Step 4: เครื่องมือที่จำเป็น

### Text Editors

```
VS Code       — แนะนำ, มี extension Perl language support
Vim/Neovim    — มี syntax highlighting ในตัว
Emacs         — มี cperl-mode
Sublime Text  — มี plugin สำหรับ Perl
Notepad++     — Windows, รองรับ Perl syntax
```

### VS Code สำหรับ Perl

```bash
# ติดตั้ง extension ใน VS Code
# 1. "Perl" by Gerald Richter
# 2. "Perl Navigator" (แนะนำ)
# 3. "Perl Toolbox"

# ติดตั้ง Perl::LanguageServer
cpanm Perl::LanguageServer

# หรือ PLS (Perl Language Server)
cpanm PLS
```

### Perl Debugger

```bash
# ใช้ debugger ในตัว
perl -d script.pl

# คำสั่งใน debugger
# n    — next line
# s    — step into
# c    — continue
# p    — print variable
# q    — quit
# h    — help
```

### CPAN และ cpanm

```bash
# ติดตั้ง module ด้วย cpanm
cpanm Module::Name

# ตัวอย่าง
cpanm CGI
cpanm DBI
cpanm LWP::UserAgent
cpanm Mojolicious

# ดู modules ที่ติดตั้งแล้ว
perl -e "use POSIX; print join '\n', sort @INC"

# หา module ใน CPAN
# เข้าเว็บ https://metacpan.org
```

---

## Step 5: โครงสร้างไฟล์ Perl

### ส่วนประกอบหลักของไฟล์ Perl

```perl
#!/usr/bin/perl
# บรรทัดแรก: shebang — บอก OS ว่าใช้ interpreter ไหน

use strict;     # บังคับให้ declare variables
use warnings;   # แสดง warning messages

# โค้ดของเรา
my $message = "สวัสดี Perl!";
print "$message\n";
```

### การตั้งชื่อไฟล์

```
script.pl       — Perl script ทั่วไป
MyModule.pm     — Perl module
cgi-script.cgi  — CGI script
```

### สิทธิ์ไฟล์ (Unix/Linux)

```bash
# ทำให้ script สามารถรันได้
chmod +x script.pl

# รัน script
./script.pl

# หรือรันผ่าน perl
perl script.pl
```

---

## Step 6: Hello World — โปรแกรมแรก

### สร้างไฟล์ hello.pl

```perl
#!/usr/bin/perl
use strict;
use warnings;

# แสดงข้อความ
print "Hello, World!\n";
print "สวัสดี โลก!\n";
print "สวัสดี Perl!\n";
```

### รันโปรแกรม

```bash
# วิธีที่ 1: ผ่าน perl command
perl hello.pl

# วิธีที่ 2: ทำให้ executable แล้วรันตรง
chmod +x hello.pl
./hello.pl

# วิธีที่ 3: One-liner (รัน code สั้นๆ โดยไม่ต้องสร้างไฟล์)
perl -e 'print "Hello, World!\n"'
```

### ผลลัพธ์

```
Hello, World!
สวัสดี โลก!
สวัสดี Perl!
```

---

## Step 7: Perl One-liners

Perl มีความสามารถพิเศษในการรัน code สั้นๆ จาก command line

```bash
# พิมพ์ข้อความ
perl -e 'print "Hello!\n"'

# ประมวลผลทุกบรรทัดของไฟล์
perl -p -e 's/foo/bar/g' input.txt

# แสดงบรรทัดที่ตรงกับ pattern
perl -n -e 'print if /error/i' log.txt

# นับบรรทัด
perl -n -e 'END{print "$.\n"}' file.txt

# แสดงบรรทัดเฉพาะ (เช่น บรรทัดที่ 5-10)
perl -n -e 'print if 5..10' file.txt

# คำนวณ
perl -e 'print 2 ** 10, "\n"'  # 1024

# แปลง CSV เป็น format อื่น
perl -F, -ane 'print "$F[0]: $F[2]\n"' data.csv
```

### Flags ที่ใช้บ่อย

| Flag | ความหมาย |
|------|----------|
| `-e` | รัน code ที่ระบุ |
| `-p` | loop ผ่านทุกบรรทัด, print ออกมา |
| `-n` | loop ผ่านทุกบรรทัด, ไม่ print |
| `-i` | แก้ไขไฟล์ in-place |
| `-F` | กำหนด field separator |
| `-a` | แยก fields อัตโนมัติ (เก็บใน @F) |
| `-l` | จัดการ newlines อัตโนมัติ |
| `-w` | เปิด warnings |

---

## Step 8: Comments ใน Perl

### Single-line Comments

```perl
#!/usr/bin/perl
use strict;
use warnings;

# นี่คือ comment บรรทัดเดียว
print "Hello\n";  # comment ท้ายบรรทัด

# ตัวแปรสำหรับเก็บชื่อ
my $name = "Somchai";  # ชื่อผู้ใช้
```

### Multi-line Comments (Pod Documentation)

```perl
#!/usr/bin/perl
use strict;
use warnings;

=pod

นี่คือ multi-line comment ใน Perl
เรียกว่า POD (Plain Old Documentation)
สามารถเขียนได้หลายบรรทัด

ใช้สำหรับ:
1. อธิบายโปรแกรม
2. สร้าง documentation
3. ชั่วคราว comment out โค้ด

=cut

print "โค้ดปกติ\n";

=begin comment

โค้ดด้านล่างนี้ถูก comment out ชั่วคราว
my $x = 10;
my $y = 20;
print $x + $y;

=end comment

=cut
```

### ใช้ __END__ สำหรับ inline data

```perl
#!/usr/bin/perl
use strict;
use warnings;

while (<DATA>) {
    chomp;
    print "ชื่อ: $_\n";
}

__END__
สมชาย
สมหญิง
วิชัย
สุดา
```

---

## Step 9: การทำงานของ Perl Interpreter

### กระบวนการ Compile และ Run

```
Source Code (.pl)
       ↓
  Compilation Phase
  (ตรวจสอบ syntax, สร้าง bytecode)
       ↓
  Runtime Phase
  (รัน bytecode)
       ↓
    Output
```

### Compile-time vs Runtime Errors

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Compile-time error: syntax error (จะ error ก่อนรัน)
# print "hello"  # ขาด semicolon

# Runtime error: เกิดขึ้นขณะรัน
my $x = 10;
my $y = 0;
# my $z = $x / $y;  # Division by zero — runtime error

# Logical error: รันได้แต่ผลผิด
my $sum = 5 + 3;
print "ผลรวม: $sum\n";  # แสดง 8 (ถูกต้อง)

my $avg = 5 + 3 / 2;   # ผิด! ควรเป็น (5+3)/2
print "เฉลี่ย: $avg\n";  # แสดง 6.5 แทน 4
```

### ตรวจสอบ Syntax โดยไม่รัน

```bash
# ตรวจสอบ syntax เท่านั้น
perl -c script.pl

# ผลลัพธ์เมื่อถูกต้อง
# script.pl syntax OK

# ผลลัพธ์เมื่อมี error
# syntax error at script.pl line 5, near "print"
```

---

## Step 10: โปรแกรมแรก — ครบถ้วน

### สรุปสิ่งที่เรียนใน Step 1-9 ผ่านโปรแกรมตัวอย่าง

```perl
#!/usr/bin/perl
#
# โปรแกรม: my_first_program.pl
# วัตถุประสงค์: แสดงการใช้งาน Perl พื้นฐาน
# ผู้สร้าง: ผู้เรียน Perl
# วันที่: 2024
#

use strict;
use warnings;

# ========================================
# ส่วนที่ 1: แสดงข้อความ
# ========================================

print "=" x 50 . "\n";
print "ยินดีต้อนรับสู่ Perl Programming!\n";
print "=" x 50 . "\n\n";

# ========================================
# ส่วนที่ 2: ตัวแปรพื้นฐาน
# ========================================

my $language = "Perl";
my $version  = 5.38;
my $year     = 1987;

print "ภาษา: $language\n";
print "Version: $version\n";
print "สร้างในปี: $year\n\n";

# ========================================
# ส่วนที่ 3: การคำนวณ
# ========================================

my $age = 2024 - $year;
print "Perl มีอายุ $age ปีแล้ว\n";

# ========================================
# ส่วนที่ 4: Array พื้นฐาน
# ========================================

my @features = ("Text Processing", "CGI", "Regular Expressions", "CPAN");

print "\nคุณสมบัติเด่นของ Perl:\n";
foreach my $feature (@features) {
    print "  - $feature\n";
}

# ========================================
# ส่วนที่ 5: Hash พื้นฐาน
# ========================================

my %creator = (
    name    => "Larry Wall",
    born    => 1954,
    country => "USA",
);

print "\nผู้สร้าง Perl:\n";
print "  ชื่อ: $creator{name}\n";
print "  เกิด: $creator{born}\n";
print "  ประเทศ: $creator{country}\n";

# ========================================
# ส่วนที่ 6: สรุป
# ========================================

print "\n" . "=" x 50 . "\n";
print "พร้อมเรียน Perl แล้ว! เริ่มต้นเลย!\n";
print "=" x 50 . "\n";
```

### รันโปรแกรม

```bash
perl my_first_program.pl
```

### ผลลัพธ์

```
==================================================
ยินดีต้อนรับสู่ Perl Programming!
==================================================

ภาษา: Perl
Version: 5.38
สร้างในปี: 1987

Perl มีอายุ 37 ปีแล้ว

คุณสมบัติเด่นของ Perl:
  - Text Processing
  - CGI
  - Regular Expressions
  - CPAN

ผู้สร้าง Perl:
  ชื่อ: Larry Wall
  เกิด: 1954
  ประเทศ: USA

==================================================
พร้อมเรียน Perl แล้ว! เริ่มต้นเลย!
==================================================
```

---

## แบบฝึกหัด Part 01

### แบบฝึกหัดที่ 1: Hello World หลายภาษา

สร้างไฟล์ `hello_languages.pl` ที่แสดงข้อความสวัสดีใน 5 ภาษา:

```perl
#!/usr/bin/perl
use strict;
use warnings;

# เติมโค้ดของคุณที่นี่
# แสดง:
# ไทย: สวัสดี
# อังกฤษ: Hello
# ญี่ปุ่น: こんにちは
# จีน: 你好
# เกาหลี: 안녕하세요
```

### แบบฝึกหัดที่ 2: ข้อมูลส่วนตัว

สร้างโปรแกรมที่แสดงข้อมูลส่วนตัวของคุณ:

```perl
#!/usr/bin/perl
use strict;
use warnings;

# เติมข้อมูลของคุณ
my $name    = "";  # ชื่อ
my $age     = 0;   # อายุ
my $city    = "";  # เมือง
my $country = "";  # ประเทศ

# แสดงข้อมูล
print "ชื่อ: $name\n";
print "อายุ: $age ปี\n";
print "เมือง: $city\n";
print "ประเทศ: $country\n";
```

### แบบฝึกหัดที่ 3: One-liner ฝึกหัด

ลองรัน one-liner ต่อไปนี้และอธิบายผลลัพธ์:

```bash
# 1. แสดงตัวเลข 1-10
perl -e 'print "$_\n" for 1..10'

# 2. คำนวณผลรวม 1-100
perl -e 'my $sum = 0; $sum += $_ for 1..100; print "$sum\n"'

# 3. แปลง uppercase
perl -e 'print uc("hello world"), "\n"'

# 4. วันและเวลาปัจจุบัน
perl -e 'print scalar(localtime), "\n"'
```

---

## สรุป Part 01

ใน Part นี้คุณได้เรียนรู้:
- ✅ Perl คืออะไรและประวัติความเป็นมา
- ✅ จุดเด่นและการใช้งาน Perl
- ✅ การติดตั้ง Perl บนระบบต่างๆ
- ✅ เครื่องมือที่จำเป็น (Editor, CPAN, cpanm)
- ✅ โครงสร้างไฟล์ Perl พื้นฐาน
- ✅ โปรแกรม Hello World แรก
- ✅ Perl One-liners
- ✅ Comments ใน Perl
- ✅ การทำงานของ Perl Interpreter
- ✅ โปรแกรมสรุปครบถ้วน

**ถัดไป: [Part 02 — Hello World และโครงสร้างโปรแกรม](part_02.md)**
