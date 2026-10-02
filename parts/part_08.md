# Part 08: String Operations
## Steps 71-80: การประมวลผลข้อความอย่างสมบูรณ์

---

## Step 71: String Functions สรุปครบ

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# ฟังก์ชัน string ทั้งหมด
# =====================

my $str = "Hello, World!";

# length
print "length: ", length($str), "\n";  # 13

# uc / lc / ucfirst / lcfirst
print "uc:      ", uc($str), "\n";           # HELLO, WORLD!
print "lc:      ", lc($str), "\n";           # hello, world!
print "ucfirst: ", ucfirst(lc($str)), "\n";  # Hello, world!
print "lcfirst: ", lcfirst($str), "\n";      # hELLO, WORLD!

# substr
print "substr(0,5):  '", substr($str, 0, 5), "'\n";   # Hello
print "substr(7,5):  '", substr($str, 7, 5), "'\n";   # World
print "substr(-6):   '", substr($str, -6),   "'\n";   # World!
print "substr(-6,5): '", substr($str, -6, 5), "'\n";  # World

# index / rindex
print "index 'l':  ", index($str, "l"), "\n";     # 2
print "index 'l',5:", index($str, "l", 5), "\n";  # 10
print "rindex 'l': ", rindex($str, "l"), "\n";    # 10

# split / join
my @words = split(/\s+/, $str);
print "split: @words\n";

my $joined = join(" | ", @words);
print "join: $joined\n";

# reverse
print "reverse: ", scalar reverse($str), "\n";  # !dlroW ,olleH

# chomp / chop
my $with_newline = "test\n";
chomp $with_newline;
print "chomp: '$with_newline'\n";  # 'test'

my $word = "Hello!";
my $last = chop $word;
print "chop: '$word', removed: '$last'\n";  # 'Hello', '!'

# sprintf
my $formatted = sprintf("Name: %-10s Age: %3d", "Alice", 25);
print "$formatted\n";

# index as search
my $hay = "the quick brown fox";
my $needle = "quick";
my $pos = index($hay, $needle);
if ($pos != -1) {
    print "Found '$needle' at position $pos\n";
}
```

---

## Step 72: Regular Expression Basics

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Regex Metacharacters
# =====================

# . — any character (except newline by default)
print "test" =~ /t.st/ ? "match\n" : "no match\n";   # match (e = any char)
print "tXst" =~ /t.st/ ? "match\n" : "no match\n";   # match
print "tst"  =~ /t.st/ ? "match\n" : "no match\n";   # no match (need 1 char)

# ^ $ — anchors
print "hello" =~ /^hell/  ? "match\n" : "no\n";  # match (start)
print "hello" =~ /ello$/  ? "match\n" : "no\n";  # match (end)
print "hello" =~ /^hello$/ ? "match\n" : "no\n"; # exact match

# * + ? — quantifiers
print "color" =~ /colou?r/   ? "match\n" : "no\n"; # u? = 0 or 1 u
print "colour" =~ /colou?r/  ? "match\n" : "no\n"; # match
print "ab"   =~ /a+b/  ? "match\n" : "no\n";  # + = 1 or more
print "aab"  =~ /a+b/  ? "match\n" : "no\n";  # match
print "b"    =~ /a*b/  ? "match\n" : "no\n";  # * = 0 or more (match)

# {n} {n,} {n,m} — specific counts
print "aaa"  =~ /a{3}/   ? "match\n" : "no\n";  # exactly 3
print "aaaa" =~ /a{3}/   ? "match\n" : "no\n";  # match (contains 3)
print "aa"   =~ /a{3}/   ? "match\n" : "no\n";  # no match
print "aaaa" =~ /^a{3}$/ ? "match\n" : "no\n";  # no (not exactly 3)
print "aaaa" =~ /a{2,4}/ ? "match\n" : "no\n";  # 2 to 4: match

# =====================
# Character Classes
# =====================

# [abc] — any of a, b, or c
print "cat" =~ /[abc]at/ ? "match\n" : "no\n";  # match (c)
print "bat" =~ /[abc]at/ ? "match\n" : "no\n";  # match (b)
print "dat" =~ /[abc]at/ ? "match\n" : "no\n";  # no match

# [a-z] — range
print "hello" =~ /^[a-z]+$/ ? "match\n" : "no\n";  # match (all lowercase)
print "Hello" =~ /^[a-z]+$/ ? "match\n" : "no\n";  # no (H is uppercase)

# [^abc] — negation
print "dog" =~ /[^abc]at/ ? "match\n" : "no\n";  # no (there's no 'at')
print "dat" =~ /[^abc]/   ? "match\n" : "no\n";  # match (d is not abc)

# =====================
# Shorthand Classes
# =====================

# \d = digit [0-9]
# \D = non-digit [^0-9]
# \w = word char [a-zA-Z0-9_]
# \W = non-word char
# \s = whitespace [ \t\n\r\f]
# \S = non-whitespace

print "123"   =~ /^\d+$/ ? "all digits\n" : "not all digits\n";
print "abc"   =~ /^\w+$/ ? "word chars\n" : "not word\n";
print " \t\n" =~ /^\s+$/ ? "whitespace\n" : "not whitespace\n";

# =====================
# Groups and Alternation
# =====================

# (abc) — grouping
print "abcabc" =~ /(abc){2}/ ? "match\n" : "no\n";  # match

# a|b — alternation
print "cat" =~ /cat|dog/ ? "match\n" : "no\n";  # match
print "dog" =~ /cat|dog/ ? "match\n" : "no\n";  # match
print "bird" =~ /cat|dog/ ? "match\n" : "no\n"; # no match

# Captures
my $date = "2024-01-15";
if ($date =~ /(\d{4})-(\d{2})-(\d{2})/) {
    print "year=$1, month=$2, day=$3\n";
}
```

---

## Step 73: Advanced Regular Expressions

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Lookahead / Lookbehind
# =====================

my $text = "price: $100 and $200";

# Positive lookahead (?=...)
my @before_dollar = ($text =~ /\d+(?= dollars)/g);
# ไม่เจออะไรเพราะไม่มีคำว่า dollars

# Lookbehind (?<=...)
my @prices = ($text =~ /(?<=\$)\d+/g);
print "prices: @prices\n";  # 100 200

# Negative lookahead (?!...)
my $words = "foo foobar foobaz";
my @foo_only = ($words =~ /foo(?!bar|baz)\b/g);
print "foo only: @foo_only\n";  # foo

# =====================
# Non-greedy
# =====================

my $html = "<b>bold</b> and <i>italic</i>";

# Greedy (default)
my ($greedy) = ($html =~ /<(.+)>/);
print "greedy: $greedy\n";  # b>bold</b> and <i>italic</i

# Non-greedy (??)
my @tags;
@tags = ($html =~ /<(.+?)>/g);
print "non-greedy: @tags\n";  # b /b i /i

# =====================
# Named captures (?<name>...)
# =====================

my $record = "Alice Smith, 30, alice@example.com";
if ($record =~ /(?<first>\w+)\s+(?<last>\w+),\s*(?<age>\d+),\s*(?<email>\S+)/) {
    print "First: $+{first}\n";
    print "Last:  $+{last}\n";
    print "Age:   $+{age}\n";
    print "Email: $+{email}\n";
}

# =====================
# Non-capturing groups (?:...)
# =====================

my $str = "catdog catcat dogdog";
my @pairs = ($str =~ /(?:cat|dog)(?:cat|dog)/g);
print "pairs: @pairs\n";

# =====================
# Global match with captures
# =====================

my $data = "name:Alice age:30 city:Bangkok";
my %parsed;
while ($data =~ /(\w+):(\w+)/g) {
    $parsed{$1} = $2;
}
foreach my $k (sort keys %parsed) {
    print "$k = $parsed{$k}\n";
}

# =====================
# Modifiers
# =====================

# x — extended mode (whitespace and comments ignored)
my $phone_re = qr/
    ^           # start
    (\+\d{1,3})?  # optional country code
    [-. ]?      # optional separator
    \(?         # optional open paren
    (\d{3})     # area code
    \)?         # optional close paren
    [-. ]?      # separator
    (\d{3})     # prefix
    [-. ]?      # separator
    (\d{4})     # number
    $           # end
/x;

my @test_phones = ("555-1234", "617-555-1234", "1-617-555-1234", "not a phone");
for my $phone (@test_phones) {
    if ($phone =~ $phone_re) {
        print "$phone: valid\n";
    } else {
        print "$phone: invalid\n";
    }
}
```

---

## Step 74: Text Processing

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Line-by-line processing
# =====================

my $text = <<'END';
Alice Smith: alice@example.com : 555-1234
Bob Jones: bob@example.com : 555-5678
Charlie Brown: charlie@example.com : 555-9012
END

my @records;
for my $line (split /\n/, $text) {
    next unless $line =~ /\S/;  # skip blank lines
    
    my ($name, $email, $phone) = split /\s*:\s*/, $line;
    push @records, { name => $name, email => $email, phone => $phone };
}

for my $r (@records) {
    printf "%-20s %-30s %s\n", $r->{name}, $r->{email}, $r->{phone};
}

# =====================
# CSV parsing
# =====================

sub parse_csv {
    my $line = shift;
    my @fields;
    
    while ($line =~ /("(?:[^""]|"")*"|[^,]*),?/g) {
        my $field = $1;
        $field =~ s/^"|"$//g;  # remove quotes
        $field =~ s/""/"/g;    # unescape quotes
        push @fields, $field;
        last if pos($line) >= length($line);
    }
    return @fields;
}

my @csv_lines = (
    'Alice,30,"Bangkok, Thailand",alice@example.com',
    'Bob,25,"New York, USA",bob@example.com',
    '"Charlie ""Chuck"" Brown",35,London,chuck@example.com',
);

print "\nCSV Parsing:\n";
for my $line (@csv_lines) {
    my @fields = split /,/, $line;  # simple (won't handle quoted commas)
    printf "  [%s]\n", join("] [", @fields);
}

# =====================
# Template substitution
# =====================

sub render_template {
    my ($template, %vars) = @_;
    $template =~ s/\{\{(\w+)\}\}/$vars{$1} \/\/ ''/ge;
    return $template;
}

my $email_template = <<'END';
Dear {{name}},

Thank you for your order #{{order_id}}.
Your order total is ${{total}}.
Expected delivery: {{delivery_date}}

Best regards,
{{company}}
END

my $email = render_template($email_template,
    name          => "Alice Smith",
    order_id      => "ORD-12345",
    total         => "150.00",
    delivery_date => "2024-02-01",
    company       => "ACME Corp",
);

print $email;

# =====================
# Word wrap
# =====================

sub word_wrap {
    my ($text, $width) = @_;
    $width //= 70;
    
    my @words = split /\s+/, $text;
    my @lines;
    my $current = '';
    
    for my $word (@words) {
        if (length($current) + length($word) + 1 <= $width) {
            $current .= ($current ? " " : "") . $word;
        } else {
            push @lines, $current if $current;
            $current = $word;
        }
    }
    push @lines, $current if $current;
    
    return join("\n", @lines);
}

my $long_text = "The quick brown fox jumps over the lazy dog. " x 5;
print "\nWrapped at 60:\n";
print word_wrap($long_text, 60), "\n";
```

---

## Step 75: String Parsing

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# INI file parser
# =====================

sub parse_ini {
    my $content = shift;
    my %config;
    my $section = 'default';
    
    for my $line (split /\n/, $content) {
        $line =~ s/\s*[;#].*$//;  # remove comments
        $line =~ s/^\s+|\s+$//g;  # trim
        next unless length $line;
        
        if ($line =~ /^\[(.+)\]$/) {
            $section = $1;
        } elsif ($line =~ /^(\w+)\s*=\s*(.*)$/) {
            $config{$section}{$1} = $2;
        }
    }
    return %config;
}

my $ini = <<'INI';
; Application settings
[database]
host = localhost
port = 3306
name = mydb
user = admin

[app]
debug = true
max_connections = 100
timeout = 30  ; seconds

[logging]
level = info
file = /var/log/app.log
INI

my %config = parse_ini($ini);

foreach my $section (sort keys %config) {
    print "\n[$section]\n";
    foreach my $key (sort keys %{$config{$section}}) {
        printf "  %-20s = %s\n", $key, $config{$section}{$key};
    }
}

# =====================
# URL parser
# =====================

sub parse_url {
    my $url = shift;
    my %parts;
    
    if ($url =~ m{
        ^(?<scheme>[a-z]+)://    # scheme
        (?:(?<user>[^:@]+)(?::(?<pass>[^@]*))?@)?  # optional user:pass
        (?<host>[^:/]+)          # host
        (?::(?<port>\d+))?       # optional port
        (?<path>/[^?#]*)?        # optional path
        (?:\?(?<query>[^#]*))?   # optional query
        (?:\#(?<fragment>.*))?   # optional fragment
    }xi) {
        %parts = %+;
    }
    return %parts;
}

my @urls = (
    "http://www.example.com/path/to/page?key=value&foo=bar#section",
    "https://user:pass\@api.example.com:8080/api/v1/users",
    "ftp://files.example.com/pub/downloads/file.txt",
);

for my $url (@urls) {
    my %p = parse_url($url);
    print "\nURL: $url\n";
    for my $part (qw(scheme user pass host port path query fragment)) {
        printf "  %-10s: %s\n", $part, $p{$part} // "(none)"
            if defined $p{$part};
    }
}

# =====================
# Log parser
# =====================

my $log = <<'LOG';
2024-01-15 09:15:23 INFO  User 'alice' logged in from 192.168.1.10
2024-01-15 09:15:45 DEBUG Processing request #12345
2024-01-15 09:16:01 WARN  Slow query: 5.3s (threshold: 2s)
2024-01-15 09:16:15 ERROR Connection to DB failed: timeout
2024-01-15 09:16:20 INFO  Retrying connection (attempt 1/3)
LOG

my %log_stats;
my @errors;

for my $line (split /\n/, $log) {
    next unless $line =~ /\S/;
    
    if ($line =~ /^(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\s+(\w+)\s+(.+)$/) {
        my ($timestamp, $level, $message) = ($1, $2, $3);
        $log_stats{$level}++;
        
        push @errors, { time => $timestamp, msg => $message } 
            if $level eq 'ERROR';
    }
}

print "\nLog Statistics:\n";
for my $level (sort keys %log_stats) {
    printf "  %-8s: %d\n", $level, $log_stats{$level};
}

if (@errors) {
    print "\nErrors:\n";
    for my $e (@errors) {
        print "  $e->{time}: $e->{msg}\n";
    }
}
```

---

## Step 76: String Encoding

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Encode;

# =====================
# UTF-8 handling
# =====================

# บอก Perl ว่า source code เป็น UTF-8
use utf8;

# Output UTF-8
binmode(STDOUT, ":utf8");

my $thai = "สวัสดีครับ";
my $japanese = "こんにちは";
my $mixed = "Hello สวัสดี World";

print "$thai\n";
print "$japanese\n";
print "$mixed\n";
print "length: ", length($mixed), "\n";  # จำนวน characters (ไม่ใช่ bytes)

# =====================
# Encode/Decode
# =====================

# เข้ารหัสเป็น UTF-8 bytes
my $utf8_bytes = encode('UTF-8', $thai);
print "byte length: ", length($utf8_bytes), "\n";  # มากกว่า char length

# ถอดรหัสจาก bytes
my $decoded = decode('UTF-8', $utf8_bytes);
print "decoded: $decoded\n";

# =====================
# Base64
# =====================

use MIME::Base64;

my $plain = "Hello, World! สวัสดี";
my $encoded = encode_base64($plain);
my $decoded2 = decode_base64($encoded);

print "\nBase64:\n";
print "Original: $plain\n";
print "Encoded:  $encoded";
print "Decoded:  $decoded2\n";

# =====================
# URL encoding
# =====================

use URI::Escape;

my $url_param = "Hello World สวัสดี & = ?";
my $url_encoded = uri_escape($url_param);
my $url_decoded = uri_unescape($url_encoded);

print "\nURL Encoding:\n";
print "Original: $url_param\n";
print "Encoded:  $url_encoded\n";
print "Decoded:  $url_decoded\n";

# =====================
# HTML encoding
# =====================

use HTML::Entities;

my $html_str = '<script>alert("XSS")</script> & "hello"';
my $html_encoded = encode_entities($html_str);
my $html_decoded = decode_entities($html_encoded);

print "\nHTML Encoding:\n";
print "Original: $html_str\n";
print "Encoded:  $html_encoded\n";
print "Decoded:  $html_decoded\n";
```

---

## Step 77: String Generation

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Password generator
# =====================

sub generate_password {
    my (%opts) = @_;
    my $length  = $opts{length}  // 12;
    my $upper   = $opts{upper}   // 1;
    my $lower   = $opts{lower}   // 1;
    my $digits  = $opts{digits}  // 1;
    my $special = $opts{special} // 1;
    
    my $chars = '';
    $chars .= 'ABCDEFGHIJKLMNOPQRSTUVWXYZ' if $upper;
    $chars .= 'abcdefghijklmnopqrstuvwxyz' if $lower;
    $chars .= '0123456789'                  if $digits;
    $chars .= '!@#$%^&*()_+-=[]{}|;:,.<>?' if $special;
    
    my @char_array = split //, $chars;
    my $password = join '', map { $char_array[rand @char_array] } 1..$length;
    
    return $password;
}

# สร้าง passwords
for (1..5) {
    print generate_password(length => 16), "\n";
}

# =====================
# Lorem ipsum generator
# =====================

my @lorem_words = qw(
    lorem ipsum dolor sit amet consectetur adipiscing elit
    sed do eiusmod tempor incididunt labore dolore magna
    aliqua enim minim veniam quis nostrud exercitation
    ullamco laboris nisi aliquip commodo consequat duis
    aute irure reprehenderit voluptate velit esse cillum
);

sub lorem_ipsum {
    my $word_count = shift // 50;
    
    my @words;
    for my $i (1..$word_count) {
        my $word = $lorem_words[int rand @lorem_words];
        $word = ucfirst $word if $i == 1 || ($i > 1 && $words[-1] =~ /[.!?]$/);
        push @words, $word;
        
        if ($i % 15 == 0 && $i < $word_count) {
            $words[-1] .= ".";
        }
    }
    $words[-1] =~ s/[.,]?$/./;
    
    return join " ", @words;
}

print "\nLorem ipsum:\n";
print word_wrap(lorem_ipsum(60), 70), "\n";

sub word_wrap {
    my ($text, $width) = @_;
    $width //= 70;
    my @words = split /\s+/, $text;
    my @lines;
    my $line = '';
    for my $w (@words) {
        if (length($line) + length($w) + 1 <= $width) {
            $line .= ($line ? ' ' : '') . $w;
        } else {
            push @lines, $line;
            $line = $w;
        }
    }
    push @lines, $line if $line;
    return join "\n", @lines;
}

# =====================
# String Templates
# =====================

sub sprintf_template {
    my ($template, %vars) = @_;
    $template =~ s/\{(\w+)\}/exists $vars{$1} ? $vars{$1} : "{$1}"/ge;
    return $template;
}

my $tmpl = "Hello, {name}! You have {count} messages.";
print sprintf_template($tmpl, name => "Alice", count => 5), "\n";
print sprintf_template($tmpl, name => "Bob"), "\n";  # {count} stays

# =====================
# Table generator
# =====================

sub generate_table {
    my ($headers, $rows, %opts) = @_;
    
    my $col_sep = $opts{col_sep} // " | ";
    my $row_sep = $opts{row_sep} // "-";
    
    # Calculate column widths
    my @widths = map { length $_ } @$headers;
    for my $row (@$rows) {
        for my $i (0..$#$row) {
            my $len = length($row->[$i] // '');
            $widths[$i] = $len if $len > $widths[$i];
        }
    }
    
    # Build format
    my $fmt = join($col_sep, map { "%-${_}s" } @widths) . "\n";
    my $sep = join("-+-", map { $row_sep x $_ } @widths) . "\n";
    
    # Build table
    my $table = '';
    $table .= sprintf $fmt, @$headers;
    $table .= $sep;
    for my $row (@$rows) {
        $table .= sprintf $fmt, map { $row->[$_] // '' } 0..$#widths;
    }
    
    return $table;
}

print "\n";
print generate_table(
    [qw(Name Age City)],
    [
        ["Alice",   25, "Bangkok"],
        ["Bob",     30, "Chiang Mai"],
        ["Charlie", 28, "Phuket"],
    ]
);
```

---

## Step 78: String Comparison และ Search

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Fuzzy matching
# =====================

# Levenshtein distance
sub levenshtein {
    my ($s, $t) = @_;
    my @d;
    
    for my $i (0..length($s)) { $d[$i][0] = $i }
    for my $j (0..length($t)) { $d[0][$j] = $j }
    
    for my $i (1..length($s)) {
        for my $j (1..length($t)) {
            my $cost = substr($s, $i-1, 1) eq substr($t, $j-1, 1) ? 0 : 1;
            $d[$i][$j] = min(
                $d[$i-1][$j] + 1,
                $d[$i][$j-1] + 1,
                $d[$i-1][$j-1] + $cost
            );
        }
    }
    return $d[length($s)][length($t)];
}

use List::Util qw(min);

my @dict = qw(apple apricot banana cherry blueberry);
my $query = "appel";  # typo

print "Fuzzy search for '$query':\n";
my @ranked = sort { 
    levenshtein($a, $query) <=> levenshtein($b, $query) 
} @dict;

for my $word (@ranked) {
    printf "  %s (distance: %d)\n", $word, levenshtein($word, $query);
}

# =====================
# String similarity
# =====================

sub similarity {
    my ($s, $t) = @_;
    my $max_len = max(length($s), length($t));
    return 0 unless $max_len;
    return 1 - levenshtein($s, $t) / $max_len;
}

use List::Util qw(max);

printf "similarity('hello', 'helo'): %.2f\n", similarity("hello", "helo");
printf "similarity('abc', 'xyz'):    %.2f\n", similarity("abc", "xyz");
printf "similarity('perl', 'perl'):  %.2f\n", similarity("perl", "perl");

# =====================
# Pattern matching
# =====================

sub match_pattern {
    my ($pattern, @strings) = @_;
    # Convert glob-like pattern to regex
    my $re = $pattern;
    $re =~ s/\./\\./g;
    $re =~ s/\*/.*?/g;
    $re =~ s/\?/./g;
    $re = "^$re\$";
    
    return grep { /^$re$/i } @strings;
}

my @files = qw(main.pl test.pl README.md config.ini data.csv script.sh);

print "\nPattern '*.pl':\n";
print "  $_\n" for match_pattern("*.pl", @files);

print "Pattern '*a*':\n";
print "  $_\n" for match_pattern("*a*", @files);
```

---

## Step 79: Text Formatting

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Number formatting
# =====================

sub format_number {
    my ($num, $decimals, $thousands_sep, $decimal_sep) = @_;
    $decimals      //= 0;
    $thousands_sep //= ',';
    $decimal_sep   //= '.';
    
    my $formatted = sprintf("%.${decimals}f", $num);
    my ($int_part, $dec_part) = split /\./, $formatted;
    
    # Add thousands separator
    $int_part =~ s/(\d)(?=(\d{3})+$)/$1$thousands_sep/g;
    
    return $dec_part ? "$int_part$decimal_sep$dec_part" : $int_part;
}

printf "%-15s: %s\n", "1234567.89", format_number(1234567.89, 2);
printf "%-15s: %s\n", "1000000",    format_number(1000000);
printf "%-15s: %s\n", "0.123",      format_number(0.123, 3);

# Thai style (Thai comma-like: .)
printf "%-15s: %s\n", "Thai format",
    format_number(1234567.50, 2, ',', '.');

# =====================
# Table formatting
# =====================

sub right_align {
    my ($str, $width) = @_;
    return sprintf "%${width}s", $str;
}

sub format_currency {
    my ($amount, $currency) = @_;
    $currency //= '$';
    return $currency . format_number($amount, 2);
}

# =====================
# Progress bar
# =====================

sub progress_bar {
    my ($current, $total, $width) = @_;
    $width //= 40;
    
    my $pct = $current / $total;
    my $filled = int($pct * $width);
    my $empty  = $width - $filled;
    
    return sprintf("[%s%s] %3.0f%%", 
        '#' x $filled, 
        '-' x $empty,
        $pct * 100);
}

for my $i (0, 10, 25, 50, 75, 90, 100) {
    printf "%3d/%d %s\n", $i, 100, progress_bar($i, 100);
}

# =====================
# Text alignment
# =====================

sub box {
    my ($text, $width, $style) = @_;
    $style //= 'single';
    
    my %borders = (
        single => { tl => '+', tr => '+', bl => '+', br => '+',
                   h => '-', v => '|' },
        double => { tl => '╔', tr => '╗', bl => '╚', br => '╝',
                   h => '═', v => '║' },
        round  => { tl => '╭', tr => '╮', bl => '╰', br => '╯',
                   h => '─', v => '│' },
    );
    
    my $b = $borders{$style} // $borders{single};
    my @lines = split /\n/, $text;
    my $max_len = max(map { length $_ } @lines, $width - 4);
    my $inner = $max_len;
    
    my $result = '';
    $result .= $b->{tl} . ($b->{h} x ($inner + 2)) . $b->{tr} . "\n";
    for my $line (@lines) {
        $result .= $b->{v} . " " . sprintf("%-${inner}s", $line) . " " . $b->{v} . "\n";
    }
    $result .= $b->{bl} . ($b->{h} x ($inner + 2)) . $b->{br} . "\n";
    
    return $result;
}

use List::Util qw(max);

print box("Hello\nWorld\nPerl!", 20, 'single');
print box("Important Message", 25, 'double');
```

---

## Step 80: โปรแกรมสรุป — Text Processing Tool

```perl
#!/usr/bin/perl
#
# โปรแกรม: text_tool.pl
# เครื่องมือประมวลผลข้อความ
#

use strict;
use warnings;
use List::Util qw(max min sum);
use POSIX qw(floor);

# =====================
# Text analysis functions
# =====================

sub count_words       { scalar split /\s+/, shift }
sub count_chars       { length shift }
sub count_chars_no_ws { my $s = shift; $s =~ s/\s+//g; length $s }
sub count_lines       { scalar split /\n/, shift }
sub count_sentences   { my @s = split /[.!?]+/, shift; scalar grep { /\S/ } @s }

sub word_frequency {
    my $text  = lc shift;
    my @words = ($text =~ /\b\w+\b/g);
    my %freq;
    $freq{$_}++ for @words;
    return %freq;
}

sub avg_word_length {
    my @words = split /\s+/, shift;
    return 0 unless @words;
    return sum(map { length } @words) / @words;
}

sub reading_time {
    my $text = shift;
    my $wpm  = 200;  # average words per minute
    my $words = count_words($text);
    return int($words / $wpm) . " min " . int(($words % $wpm) / ($wpm/60)) . " sec";
}

# =====================
# Test text
# =====================

my $sample = <<'TEXT';
Perl is a high-level, general-purpose, interpreted, dynamic programming
language. Perl was originally developed by Larry Wall in 1987 as a
Unix scripting language to make report processing easier. Since then,
it has undergone many changes and revisions and become widely popular
amongst programmers. Larry Wall continues to oversee development of
the core language, and its upcoming version, Perl 7, is currently
in development.

The Perl languages borrow features from other programming languages
including C, shell script, AWK, and sed. They provide powerful
text processing facilities without the arbitrary data length limits
of many contemporary Unix commandline tools, facilitating easy
manipulation of text files.
TEXT

# =====================
# Display analysis
# =====================

print "=" x 60 . "\n";
print "TEXT ANALYSIS TOOL\n";
print "=" x 60 . "\n\n";

printf "%-25s: %d\n",   "Characters (total)",  count_chars($sample);
printf "%-25s: %d\n",   "Characters (no space)",count_chars_no_ws($sample);
printf "%-25s: %d\n",   "Words",               count_words($sample);
printf "%-25s: %d\n",   "Lines",               count_lines($sample);
printf "%-25s: %d\n",   "Sentences",           count_sentences($sample);
printf "%-25s: %.2f\n", "Avg word length",     avg_word_length($sample);
printf "%-25s: %s\n",   "Est. reading time",   reading_time($sample);

# Word frequency
my %freq = word_frequency($sample);
my @top10 = (sort { $freq{$b} <=> $freq{$a} || $a cmp $b } keys %freq)[0..9];

print "\nTop 10 words:\n";
printf "  %-15s: %d\n", $_, $freq{$_} for @top10;

# Character frequency (only letters)
my %char_freq;
my @chars = ($sample =~ /([a-z])/gi);
$char_freq{lc $_}++ for @chars;

print "\nLetter frequency:\n";
for my $letter ('a'..'z') {
    next unless $char_freq{$letter};
    my $bar = '#' x int($char_freq{$letter} / 5);
    printf "  %s: %-20s (%d)\n", $letter, $bar, $char_freq{$letter};
}
```

---

## แบบฝึกหัด Part 08

### แบบฝึกหัดที่ 1: String Processor
สร้างโปรแกรมที่รับ string และทำ:
- Reverse
- Count vowels/consonants
- Check palindrome
- Title case

### แบบฝึกหัดที่ 2: Log Parser
เขียน parser สำหรับ Apache access log format

### แบบฝึกหัดที่ 3: Template Engine
สร้าง template engine ที่รองรับ `{{var}}`, `{{#if}}`, `{{#each}}`

---

## สรุป Part 08

ใน Part นี้คุณได้เรียนรู้:
- ✅ String functions ครบถ้วน
- ✅ Regular expressions ขั้นพื้นฐานและขั้นสูง
- ✅ Text processing และ parsing
- ✅ String encoding (UTF-8, Base64, URL)
- ✅ String generation (passwords, templates)
- ✅ Fuzzy matching และ similarity
- ✅ Text formatting (tables, boxes, progress bars)
- ✅ Text analysis tool

**ถัดไป: [Part 09 — Input/Output พื้นฐาน](part_09.md)**
