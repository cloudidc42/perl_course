# Part 12: Regular Expressions ขั้นสูง
## Steps 111-120: การใช้ Regex อย่างมืออาชีพ

---

## Step 111: Regex Basics Review และ Modifiers

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Modifiers ทั้งหมด
# =====================

my $text = "Hello World\nThis is Line 2\nAnother LINE";

# /i — case insensitive
print "found\n" if $text =~ /hello/i;

# /g — global (find all)
my @words = ($text =~ /\b\w+/g);
printf "Words: %d\n", scalar @words;

# /m — multiline (^ and $ match line boundaries)
my @lines = ($text =~ /^(\w+)/mg);
print "Line starts: @lines\n";

# /s — single-line (. matches \n)
print "dot matches nl\n" if $text =~ /Hello.World/s;

# /x — extended (allow whitespace and comments)
my $email_re = qr/
    ^           # start
    [^\@]+      # local part
    \@          # @
    [^\@]+      # domain
    \.          # dot
    [^\@]+      # TLD
    $           # end
/x;

print "valid\n"   if "user\@example.com" =~ $email_re;
print "invalid\n" if "bad-email"         =~ $email_re;

# /e — evaluate replacement as code
my $str = "1 + 2 = ANSWER";
(my $evaluated = $str) =~ s/(\d+) \+ (\d+)/($1+$2)/e;
print "$evaluated\n";   # 3 = ANSWER

# /r — non-destructive (Perl 5.14+)
my $original = "Hello World";
my $modified = $original =~ s/World/Perl/r;
print "original: $original\n";   # unchanged
print "modified: $modified\n";

# Combined: /gi
my @all_words = ($text =~ /\bline\b/gi);
printf "Found 'line': %d times\n", scalar @all_words;
```

---

## Step 112: Character Classes และ Assertions

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# POSIX character classes
# =====================

my $str = "Hello World 123!";

# [:alpha:] — letters
(my $letters_only = $str) =~ s/[^[:alpha:]]//g;
print "Letters: $letters_only\n";

# [:digit:] — digits
my @digits = ($str =~ /[[:digit:]]/g);
print "Digits: @digits\n";

# [:alnum:] — letters and digits
my @alnum = ($str =~ /[[:alnum:]]+/g);
print "Alnum words: @alnum\n";

# [:space:] — whitespace
my @parts = split /[[:space:]]+/, $str;
print "Parts: @parts\n";

# [:punct:] — punctuation
my @punct = ($str =~ /[[:punct:]]/g);
print "Punct: @punct\n";

# =====================
# Anchors
# =====================

my @lines = ("start middle end", "start_only", "middle end", " padded ");

for my $line (@lines) {
    print "'$line': ";
    print "starts-with-start " if $line =~ /^start/;
    print "ends-with-end "     if $line =~ /end$/;
    print "word-boundary "     if $line =~ /\bstart\b/;
    print "\n";
}

# =====================
# Lookahead and Lookbehind
# =====================

my $text = "100px 200em 300px 50% 75rem";

# Positive lookahead: digits followed by px
my @px_values = ($text =~ /(\d+)(?=px)/g);
print "px values: @px_values\n";   # 100 300

# Negative lookahead: digits NOT followed by px
my @non_px = ($text =~ /(\d+)(?!px)(?:\.\d+)?\s*(?:%|em|rem)/g);
# Alternative: find numbers not in px units
my @units;
while ($text =~ /(\d+)(px|em|rem|%)/g) {
    push @units, "$1$2" if $2 ne 'px';
}
print "non-px: @units\n";

# Positive lookbehind: number after $
my $prices = "Cost: \$100, Tax: \$15, Total: \$115";
my @amounts = ($prices =~ /(?<=\$)(\d+)/g);
print "Amounts: @amounts\n";

# Negative lookbehind
my $code = "func() method() _private()";
my @public_funcs = ($code =~ /(?<![_])(\w+)\(\)/g);
print "Public funcs: @public_funcs\n";

# =====================
# Atomic groups and possessive
# =====================

# Non-backtracking group (?> ... )
my $html = "<div>content</div>";
if ($html =~ /(<(?>[^>]+)>)/) {
    print "Tag: $1\n";
}
```

---

## Step 113: Captures และ Backreferences

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Numbered captures
# =====================

my $date = "2024-03-15";
if ($date =~ /^(\d{4})-(\d{2})-(\d{2})$/) {
    printf "Year: %s, Month: %s, Day: %s\n", $1, $2, $3;
}

# =====================
# Named captures (?<name>...)
# =====================

if ($date =~ /^(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})$/) {
    printf "Year: %s, Month: %s, Day: %s\n",
        $+{year}, $+{month}, $+{day};
}

# =====================
# Non-capturing group (?:...)
# =====================

my $str = "foobar foobaz";
my @matches = ($str =~ /foo(?:bar|baz)/g);
print "Matches: @matches\n";   # foobar foobaz

# =====================
# Backreferences in pattern
# =====================

# Find repeated words
my $text = "the the quick brown brown fox";
while ($text =~ /\b(\w+)\s+\1\b/g) {
    print "Repeated: '$1'\n";
}

# Find matching tags
my $html = "<b>bold</b> and <i>italic</i>";
while ($html =~ /<(\w+)>(.+?)<\/\1>/g) {
    print "Tag: $1, Content: $2\n";
}

# =====================
# Captures in replacement
# =====================

# Swap first and last name
my $name = "Smith, John";
(my $reversed = $name) =~ s/^(\w+),\s*(\w+)/$2 $1/;
print "Reversed: $reversed\n";

# Format date
my $iso_date = "2024-03-15";
(my $us_date = $iso_date) =~ s/^(\d{4})-(\d{2})-(\d{2})$/$2\/$3\/$1/;
print "US date: $us_date\n";

# Camel to snake_case
sub camel_to_snake {
    my $str = shift;
    $str =~ s/([A-Z])/'_' . lc($1)/ge;
    $str =~ s/^_//;
    return $str;
}

for my $name (qw(camelCase myVariableName HTMLParser)) {
    printf "%-20s => %s\n", $name, camel_to_snake($name);
}

# =====================
# Global match with captures
# =====================

my $csv_line = 'Alice,"New York",28,"Perl, Python"';
my @fields;
while ($csv_line =~ /(?:"([^"]*?)"|([^,]*))/g) {
    push @fields, defined $1 ? $1 : $2;
    last if pos($csv_line) >= length($csv_line);
}
print "CSV fields: ", join(" | ", @fields), "\n";
```

---

## Step 114: tr/// และ String Transliteration

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# tr/// (y///) basics
# =====================

my $str = "Hello, World!";

# Count occurrences
my $vowels = ($str =~ tr/aeiouAEIOU//);
print "Vowels: $vowels\n";

# Replace
(my $no_vowels = $str) =~ tr/aeiouAEIOU/*/;
print "No vowels: $no_vowels\n";

# Case conversion
my $upper = $str;
$upper =~ tr/a-z/A-Z/;
print "Upper: $upper\n";

my $lower = $str;
$lower =~ tr/A-Z/a-z/;
print "Lower: $lower\n";

# =====================
# tr modifiers
# =====================

# /d — delete characters
my $text = "He110 W0r1d";
(my $digits_removed = $text) =~ tr/0-9//d;
print "Digits removed: $digits_removed\n";

# /s — squeeze repeated chars
my $squeezed = "heeellllooo   wwwooorrrlddd";
$squeezed =~ tr/a-z//s;
print "Squeezed: $squeezed\n";   # helo world

# /c — complement
my $only_letters = "Hello 123 World!";
$only_letters =~ tr/a-zA-Z//cd;   # delete non-letters
print "Only letters: $only_letters\n";

# =====================
# ROT13
# =====================

sub rot13 {
    my $str = shift;
    $str =~ tr/A-Za-z/N-ZA-Mn-za-m/;
    return $str;
}

my $msg = "Hello, World!";
my $encoded = rot13($msg);
my $decoded = rot13($encoded);
print "Original: $msg\n";
print "Encoded:  $encoded\n";
print "Decoded:  $decoded\n";

# =====================
# Caesar cipher
# =====================

sub caesar {
    my ($str, $shift) = @_;
    $shift %= 26;
    my $from = join('', 'A'..'Z', 'a'..'z');
    my $to   = join('', (map { chr((ord($_) - ord('A') + $shift) % 26 + ord('A')) } 'A'..'Z'),
                        (map { chr((ord($_) - ord('a') + $shift) % 26 + ord('a')) } 'a'..'z'));
    eval "\$str =~ tr/$from/$to/";
    return $str;
}

my $secret = caesar("Hello, World!", 13);
print "Caesar(13): $secret\n";
print "Decoded:    ", caesar($secret, -13), "\n";

# =====================
# Count char frequency
# =====================

my $sample = "the quick brown fox jumps over the lazy dog";
my %freq;
for my $c (split //, $sample) {
    $freq{$c}++ if $c =~ /[a-z]/;
}

print "\nChar frequency:\n";
printf "  %s: %d\n", $_, $freq{$_}
    for sort { $freq{$b} <=> $freq{$a} } keys %freq;
```

---

## Step 115: Advanced Substitution

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# /e flag — evaluate replacement
# =====================

my $template = "Result is 2 + 3 and 10 * 5";
(my $computed = $template) =~ s/(\d+)\s*([+\-*\/])\s*(\d+)/eval("$1 $2 $3")/ge;
print "$computed\n";

# =====================
# Variable interpolation in replacement
# =====================

my %translations = (
    hello  => "สวัสดี",
    world  => "โลก",
    perl   => "เพิร์ล",
);

my $str = "hello world perl";
$str =~ s/(\w+)/$translations{$1} \/\/ $1/ge;
print "$str\n";

# =====================
# Template engine
# =====================

sub render_template {
    my ($template, $vars) = @_;
    $template =~ s/\{\{(\w+)\}\}/$vars->{$1} \/\/ ""/ge;
    return $template;
}

my $tmpl = "Dear {{name}},\nYour order {{order_id}} has been {{status}}.\n";
my $rendered = render_template($tmpl, {
    name     => "Alice",
    order_id => "ORD-12345",
    status   => "shipped",
});
print $rendered;

# =====================
# HTML entities
# =====================

sub html_encode {
    my $str = shift;
    $str =~ s/&/&amp;/g;
    $str =~ s/</&lt;/g;
    $str =~ s/>/&gt;/g;
    $str =~ s/"/&quot;/g;
    $str =~ s/'/&#39;/g;
    return $str;
}

sub html_decode {
    my $str = shift;
    $str =~ s/&amp;/&/g;
    $str =~ s/&lt;/</g;
    $str =~ s/&gt;/>/g;
    $str =~ s/&quot;/"/g;
    $str =~ s/&#39;/'/g;
    return $str;
}

my $html = "<script>alert('XSS')</script>";
my $safe = html_encode($html);
print "Encoded: $safe\n";
print "Decoded: ", html_decode($safe), "\n";

# =====================
# Word wrap
# =====================

sub word_wrap {
    my ($text, $width) = @_;
    $width //= 72;
    $text =~ s/(.{1,$width})(?:\s|$)/$1\n/g;
    $text =~ s/\n+$/\n/;
    return $text;
}

my $paragraph = "This is a very long paragraph that needs to be wrapped at a specific column width for better readability in terminal output and other text-based interfaces.";
print word_wrap($paragraph, 50);

# =====================
# Pluralize
# =====================

sub pluralize {
    my ($word, $count) = @_;
    return "$count $word" if $count == 1;
    
    # Basic English pluralization rules
    return "$count ${word}ies" if $word =~ s/y$/y/  && $word =~ /[^aeiou]y$/;
    return "$count ${word}es"  if $word =~ /(?:s|sh|ch|x|z)$/;
    return "$count ${word}s";
}

for my $pair ([1, "cat"], [3, "dog"], [2, "box"], [5, "city"], [0, "fish"]) {
    print pluralize($pair->[1], $pair->[0]), "\n";
}
```

---

## Step 116: Regex Optimization

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Benchmark qw(:all);
use Time::HiRes qw(time);

# =====================
# Compiled regex: qr//
# =====================

# แบบไม่ compile ทุกครั้ง (ช้า)
sub match_slow {
    my ($str, $pattern) = @_;
    return $str =~ /$pattern/;
}

# แบบ compile ครั้งเดียว (เร็วกว่า)
my $compiled = qr/\d{4}-\d{2}-\d{2}/;
sub match_fast {
    my $str = shift;
    return $str =~ $compiled;
}

# =====================
# Anchoring สำคัญมาก
# =====================

my $long_str = "a" x 10000 . "xyz";

my $t1 = time();
for (1..10000) {
    $long_str =~ /xyz/;
}
printf "Without anchor: %.4fs\n", time() - $t1;

$t1 = time();
for (1..10000) {
    $long_str =~ /xyz$/;   # anchor ช่วยลดพื้นที่ค้นหา
}
printf "With anchor: %.4fs\n", time() - $t1;

# =====================
# Catastrophic backtracking (หลีกเลี่ยง!)
# =====================

# ระวัง! นี่คือตัวอย่าง regex ที่อาจทำให้ช้ามาก
# BAD:  /^(a+)+$/    ← exponential backtracking
# GOOD: /^a+$/       ← linear

# =====================
# Specific > General
# =====================

my $emails = ["alice\@example.com", "bob\@test.org", "invalid"];

# General (slower)
my $general_re = qr/.+\@.+/;

# Specific (faster and more correct)
my $specific_re = qr/^[a-zA-Z0-9._%+\-]+\@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$/;

for my $email (@$emails) {
    printf "%-25s general:%-5s specific:%-5s\n",
        $email,
        $email =~ $general_re  ? "match" : "no",
        $email =~ $specific_re ? "match" : "no";
}

# =====================
# Use index() when possible
# =====================

my $haystack = "the quick brown fox jumps over the lazy dog";
my $needle = "fox";

# index is faster than regex for literal strings
if (index($haystack, $needle) >= 0) {
    print "Found '$needle' with index\n";
}

if ($haystack =~ /\Q$needle\E/) {   # \Q...\E for literal matching
    print "Found '$needle' with regex\n";
}

# =====================
# \Q...\E (quote metacharacters)
# =====================

my $user_input = "file.txt (version 2.0)";
my $escaped = quotemeta($user_input);  # same as \Q...\E

my $text = "Please open file.txt (version 2.0) now";
if ($text =~ /\Q$user_input\E/) {
    print "Found literal string\n";
}
```

---

## Step 117: Real-World Regex Patterns

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Email validation
# =====================

my $email_re = qr/^[a-zA-Z0-9.!#$%&'*+\/=?^_`{|}~\-]+\@[a-zA-Z0-9](?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?(?:\.[a-zA-Z0-9](?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?)*$/;

for my $e ("user\@example.com", "bad", "a\@b.c", "x.y+z\@domain.co.th") {
    printf "%-30s %s\n", $e, $e =~ $email_re ? "valid" : "invalid";
}

# =====================
# URL parsing
# =====================

my $url_re = qr{
    ^
    (https?)://           # scheme
    ([^/:]+)              # host
    (?::(\d+))?           # optional port
    (/[^?#]*)?            # path
    (?:\?([^#]*))?        # query
    (?:\#(.*))?           # fragment
    $
}x;

my @urls = (
    "https://www.example.com/path/to/page?q=hello&lang=en#section1",
    "http://localhost:8080/api/v1/users",
    "https://example.com",
);

for my $url (@urls) {
    if ($url =~ $url_re) {
        printf "URL: %s\n  scheme=%s host=%s port=%s path=%s query=%s\n",
            $url, $1, $2, $3//"", $4//"", $5//"";
    }
}

# =====================
# IP address
# =====================

sub is_valid_ip {
    my $ip = shift;
    return 0 unless $ip =~ /^(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})$/;
    return 0 if grep { $_ > 255 } ($1, $2, $3, $4);
    return 1;
}

for my $ip ("192.168.1.1", "256.0.0.1", "10.0.0.0", "999.999.999.999") {
    printf "%-20s %s\n", $ip, is_valid_ip($ip) ? "valid" : "invalid";
}

# =====================
# Credit card (Luhn algorithm)
# =====================

sub luhn_check {
    my $num = shift;
    $num =~ s/\D//g;
    my @digits = reverse split //, $num;
    my $sum = 0;
    for my $i (0..$#digits) {
        my $d = $digits[$i];
        $d *= 2 if $i % 2 == 1;
        $d -= 9 if $d > 9;
        $sum += $d;
    }
    return $sum % 10 == 0;
}

for my $card ("4532015112830366", "1234567890123456", "79927398713") {
    printf "%-20s %s\n", $card, luhn_check($card) ? "valid" : "invalid";
}

# =====================
# Log parsing
# =====================

my $log_re = qr/
    ^
    \[(?<date>\d{4}-\d{2}-\d{2})\s+(?<time>\d{2}:\d{2}:\d{2})\]  # timestamp
    \s+
    (?<level>DEBUG|INFO|WARN|ERROR|FATAL)   # level
    \s+
    (?<source>\w+):                          # source module
    \s+
    (?<message>.+)                           # message
    $
/x;

my @logs = (
    "[2024-03-15 14:30:25] INFO  app: User Alice logged in",
    "[2024-03-15 14:31:02] ERROR database: Connection failed: timeout",
    "[2024-03-15 14:31:05] WARN  auth: Invalid token from 192.168.1.100",
);

for my $log (@logs) {
    if ($log =~ $log_re) {
        printf "%-5s [%s] %s: %s\n",
            $+{level}, $+{time}, $+{source}, $+{message};
    }
}
```

---

## Step 118: Parsing กับ Regex

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# CSV Parser
# =====================

sub parse_csv_line {
    my $line = shift;
    my @fields;
    
    while ($line =~ /
        (?:^|,)                    # start or comma
        (?:
            "([^"]*(?:""[^"]*)*)"  # quoted field
            |
            ([^,]*)                 # unquoted field
        )
    /gx) {
        my $field = defined $1 ? $1 : $2;
        $field =~ s/""/"/g if defined $1;  # unescape ""
        push @fields, $field;
    }
    return @fields;
}

my @csv_lines = (
    'Alice,28,"New York",Engineer',
    '"Bob Smith",35,"Los Angeles, CA","Manager, Sales"',
    '"Carol ""The Expert"" Jones",42,Boston,CTO',
);

for my $line (@csv_lines) {
    my @fields = parse_csv_line($line);
    printf "  [%s]\n", join("] [", @fields);
}

# =====================
# INI file parser
# =====================

sub parse_ini {
    my $content = shift;
    my %config;
    my $section = '_default';
    
    for my $line (split /\n/, $content) {
        $line =~ s/#.*$//;         # remove comments
        $line =~ s/^\s+|\s+$//g;   # trim
        next unless length $line;
        
        if ($line =~ /^\[(.+)\]$/) {
            $section = $1;
        } elsif ($line =~ /^(\w+)\s*=\s*(.*)$/) {
            $config{$section}{$1} = $2;
        }
    }
    
    return %config;
}

my $ini_text = <<'INI';
# Main config
[database]
host = localhost
port = 5432
name = myapp

[server]
host = 0.0.0.0
port = 8080
debug = true
INI

my %ini = parse_ini($ini_text);
for my $section (sort keys %ini) {
    print "[$section]\n";
    printf "  %s = %s\n", $_, $ini{$section}{$_}
        for sort keys %{$ini{$section}};
}

# =====================
# HTTP header parser
# =====================

sub parse_http_headers {
    my $raw = shift;
    my %headers;
    
    for my $line (split /\r?\n/, $raw) {
        if ($line =~ /^(\S+):\s+(.+)$/) {
            my ($name, $value) = ($1, $2);
            $name = lc $name;
            # Multi-value headers
            if (exists $headers{$name}) {
                $headers{$name} = [$headers{$name}] unless ref $headers{$name};
                push @{$headers{$name}}, $value;
            } else {
                $headers{$name} = $value;
            }
        }
    }
    
    return %headers;
}

my $raw_headers = "Content-Type: application/json\r\nContent-Length: 42\r\nX-Custom: value1\r\nX-Custom: value2";
my %headers = parse_http_headers($raw_headers);

for my $key (sort keys %headers) {
    my $val = ref $headers{$key} ? join(", ", @{$headers{$key}}) : $headers{$key};
    print "$key: $val\n";
}
```

---

## Step 119: Regex สำหรับ Text Analysis

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Word frequency analysis
# =====================

my $text = <<'TEXT';
To be or not to be that is the question
Whether tis nobler in the mind to suffer
The slings and arrows of outrageous fortune
Or to take arms against a sea of troubles
TEXT

my %word_count;
my $total_words = 0;
my $total_unique;

while ($text =~ /\b([a-z]+)\b/gi) {
    $word_count{lc $1}++;
    $total_words++;
}
$total_unique = scalar keys %word_count;

printf "Total words: %d, Unique: %d\n\n", $total_words, $total_unique;
printf "Top 10 words:\n";

my $rank = 0;
for my $word (sort { $word_count{$b} <=> $word_count{$a} } keys %word_count) {
    last if ++$rank > 10;
    printf "  %2d. %-15s %d\n", $rank, $word, $word_count{$word};
}

# =====================
# Sentence detection
# =====================

sub split_sentences {
    my $text = shift;
    my @sentences;
    
    while ($text =~ /([^.!?]+[.!?]+(?:\s|$))/g) {
        my $s = $1;
        $s =~ s/^\s+|\s+$//g;
        push @sentences, $s if length $s;
    }
    return @sentences;
}

my $para = "Hello! How are you? I'm doing well. This is Perl. It's great!";
my @sentences = split_sentences($para);
printf "Sentence %d: %s\n", $_, $sentences[$_-1] for 1..@sentences;

# =====================
# Keyword extraction
# =====================

sub extract_keywords {
    my ($text, $min_length, $top_n) = @_;
    $min_length //= 4;
    $top_n //= 10;
    
    my %stopwords = map { $_ => 1 } 
        qw(the and or but in on at to for of with this that is are was were);
    
    my %freq;
    while ($text =~ /\b([a-zA-Z]{$min_length,})\b/g) {
        my $word = lc $1;
        next if $stopwords{$word};
        $freq{$word}++;
    }
    
    return (sort { $freq{$b} <=> $freq{$a} } keys %freq)[0..$top_n-1];
}

my @keywords = extract_keywords($text, 3, 5);
print "\nKeywords: @keywords\n";

# =====================
# HTML text extraction
# =====================

sub strip_html {
    my $html = shift;
    $html =~ s/<[^>]+>//g;    # remove tags
    $html =~ s/&nbsp;/ /g;
    $html =~ s/&lt;/</g;
    $html =~ s/&gt;/>/g;
    $html =~ s/&amp;/&/g;
    $html =~ s/\s+/ /g;
    $html =~ s/^\s+|\s+$//g;
    return $html;
}

my $html = "<h1>Title</h1><p>This is a <b>bold</b> paragraph with <a href='#'>link</a>.</p>";
print "Stripped: ", strip_html($html), "\n";
```

---

## Step 120: โปรแกรมสรุป — Text Processor

```perl
#!/usr/bin/perl
#
# text_processor.pl — โปรแกรมประมวลผลข้อความ
#

use strict;
use warnings;
use POSIX qw(floor);

# =====================
# Utility subs
# =====================

sub trim { my $s = shift; $s =~ s/^\s+|\s+$//g; $s }
sub max_val { my $m = shift; $m = $_ > $m ? $_ : $m for @_; $m }

# =====================
# Analysis functions
# =====================

sub analyze_text {
    my $text = shift;
    
    my %stats;
    
    # Word count
    my @words = ($text =~ /\b[a-zA-Z'-]+\b/g);
    $stats{word_count}  = scalar @words;
    
    # Sentence count
    my @sentences = ($text =~ /[^.!?]+[.!?]+/g);
    $stats{sentence_count} = scalar @sentences;
    
    # Character count
    $stats{char_count}       = length $text;
    $stats{char_no_space}    = () = $text =~ /\S/g;
    
    # Average word length
    my $total_len = 0;
    $total_len += length $_ for @words;
    $stats{avg_word_len} = @words ? $total_len / @words : 0;
    
    # Readability (Flesch-Kincaid approximation)
    my $syllables = 0;
    for my $word (@words) {
        # Count vowel groups as syllables
        my @v = ($word =~ /[aeiouAEIOU]+/g);
        $syllables += scalar @v || 1;
    }
    
    if ($stats{sentence_count} > 0 && $stats{word_count} > 0) {
        my $fk = 206.835
            - 1.015  * ($stats{word_count} / $stats{sentence_count})
            - 84.6   * ($syllables / $stats{word_count});
        $stats{readability} = $fk;
    }
    
    # Word frequency
    my %freq;
    $freq{lc $_}++ for @words;
    $stats{word_freq} = \%freq;
    $stats{unique_words} = scalar keys %freq;
    
    # Vocabulary richness (TTR)
    $stats{ttr} = $stats{word_count} > 0
        ? $stats{unique_words} / $stats{word_count}
        : 0;
    
    return %stats;
}

sub print_report {
    my ($title, %stats) = @_;
    
    print "\n" . "=" x 60 . "\n";
    print "TEXT ANALYSIS: $title\n";
    print "=" x 60 . "\n";
    
    printf "  Characters (with spaces):   %d\n", $stats{char_count};
    printf "  Characters (no spaces):     %d\n", $stats{char_no_space};
    printf "  Words:                      %d\n", $stats{word_count};
    printf "  Unique words:               %d\n", $stats{unique_words};
    printf "  Sentences:                  %d\n", $stats{sentence_count};
    printf "  Avg words/sentence:         %.1f\n",
        $stats{sentence_count} > 0
        ? $stats{word_count} / $stats{sentence_count}
        : 0;
    printf "  Avg word length:            %.1f chars\n", $stats{avg_word_len};
    printf "  Vocabulary richness (TTR):  %.2f\n", $stats{ttr};
    
    if (exists $stats{readability}) {
        printf "  Readability score:          %.1f", $stats{readability};
        my $level = $stats{readability} >= 90 ? "Very Easy"
                  : $stats{readability} >= 70 ? "Easy"
                  : $stats{readability} >= 50 ? "Standard"
                  : $stats{readability} >= 30 ? "Difficult"
                  :                             "Very Difficult";
        print " ($level)\n";
    }
    
    # Top words
    print "\n  Top 10 words:\n";
    my $rank = 0;
    for my $word (sort { $stats{word_freq}{$b} <=> $stats{word_freq}{$a} }
                       keys %{$stats{word_freq}}) {
        last if ++$rank > 10;
        printf "    %2d. %-20s %d\n", $rank, $word, $stats{word_freq}{$word};
    }
}

# =====================
# Sample texts
# =====================

my $simple_text = "The cat sat on the mat. The cat is fat. The mat is flat.";

my $complex_text = <<'TEXT';
Perl is a highly capable, feature-rich programming language with over 30 years
of development. Originally developed by Larry Wall in 1987 as a general-purpose
Unix scripting language to make report processing easier, it has since grown
into a versatile, general-purpose programming language.

Perl borrows features from other programming languages including C, shell
scripting, AWK, and sed. The language provides powerful text processing
facilities without the arbitrary data-length limits of many contemporary
Unix commandline tools, facilitating easy manipulation of text files.
TEXT

# =====================
# Run analysis
# =====================

my %s1 = analyze_text($simple_text);
my %s2 = analyze_text($complex_text);

print_report("Simple Text", %s1);
print_report("Complex Text", %s2);

# =====================
# Pattern search
# =====================

print "\n" . "=" x 60 . "\n";
print "PATTERN SEARCH\n";
print "=" x 60 . "\n";

my @patterns = (
    [qr/Perl/,           "Mentions of 'Perl'"],
    [qr/\d{4}/,          "4-digit numbers (years)"],
    [qr/\b[A-Z]\w+/,     "Capitalized words"],
    [qr/\b\w{10,}\b/,    "Long words (10+ chars)"],
);

for my $pat (@patterns) {
    my ($re, $desc) = @$pat;
    my @matches = ($complex_text =~ /$re/g);
    printf "  %-35s %d matches\n", "$desc:", scalar @matches;
    print "    " . join(", ", @matches[0..4]) . "\n" if @matches;
}
```

---

## สรุป Part 12

ใน Part นี้คุณได้เรียนรู้:
- ✅ Regex modifiers: /i /g /m /s /x /e /r
- ✅ POSIX character classes
- ✅ Lookahead และ Lookbehind
- ✅ Named captures และ backreferences
- ✅ tr/// และ transliteration
- ✅ Advanced substitution patterns
- ✅ Regex optimization
- ✅ Real-world patterns (email, URL, IP, log)
- ✅ Text analysis

**ถัดไป: [Part 13 — File I/O ขั้นสูง](part_13.md)**
