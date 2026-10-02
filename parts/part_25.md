# Part 25: Advanced Regular Expressions
## Steps 241-250: Regex ระดับสูง

---

## Step 241: Regex พื้นฐาน Review

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Regex fundamentals review
# =====================

my $text = "The quick brown fox jumps over the lazy dog";

# Basic match
if ($text =~ /fox/) {
    print "Found fox\n";
}

# Case insensitive
if ($text =~ /THE/i) {
    print "Found THE (case insensitive)\n";
}

# Capture groups
if ($text =~ /(\w+) fox/) {
    printf "Word before fox: %s\n", $1;
}

# Multiple captures
if ($text =~ /(\w+) (\w+) fox/) {
    printf "Two words before fox: %s %s\n", $1, $2;
}

# Non-capturing group
if ($text =~ /(?:\w+ ){3}(fox)/) {
    printf "Captured: %s\n", $1;
}

# Named captures
if ($text =~ /(?<adj>\w+) (?<animal>fox)/) {
    printf "Adj: %s, Animal: %s\n", $+{adj}, $+{animal};
}

# Global match — list context
my @words = ($text =~ /\b\w{4}\b/g);
printf "4-letter words: %s\n", join(", ", @words);

# Substitution
(my $modified = $text) =~ s/fox/cat/;
print "Modified: $modified\n";

# Global substitution
(my $shouted = $text) =~ s/\b(\w)/uc($1)/ge;
print "Title Case: $shouted\n";

# tr (transliterate)
(my $vowels_removed = $text) =~ tr/aeiouAEIOU//d;
print "No vowels: $vowels_removed\n";

my $count = ($text =~ tr/aeiouAEIOU//);
printf "Vowel count: %d\n", $count;

# Anchors
my @lines = ("apple pie", "pineapple", "apple", "green apple");
my @starts_apple = grep { /^apple/ }   @lines;
my @ends_apple   = grep { /apple$/ }   @lines;
printf "Starts with apple: %s\n", join(", ", @starts_apple);
printf "Ends with apple: %s\n",   join(", ", @ends_apple);

# Alternation
my @fruits = grep { /\b(apple|banana|cherry)\b/i } (
    "I love apple", "banana split", "Cherry pie", "grape juice"
);
printf "Fruits: %s\n", join("; ", @fruits);
```

---

## Step 242: Quantifiers and Greedy/Lazy

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Greedy vs Lazy quantifiers
# =====================

my $html = '<b>bold</b> and <i>italic</i>';

# Greedy — matches as much as possible
if ($html =~ /<(.+)>/) {
    printf "Greedy: %s\n", $1;   # b>bold</b> and <i>italic</i
}

# Lazy — matches as little as possible
if ($html =~ /<(.+?)>/) {
    printf "Lazy: %s\n", $1;     # b
}

# All tags (greedy vs lazy)
my @greedy_tags = ($html =~ /<(.+)>/g);
my @lazy_tags   = ($html =~ /<(.+?)>/g);
printf "Greedy tags: %s\n", join(", ", @greedy_tags);
printf "Lazy tags:   %s\n", join(", ", @lazy_tags);

# =====================
# Quantifier summary
# =====================

printf "\n--- Quantifier tests ---\n";
my @tests = (
    ["a+",   "aaa",  "one or more"],
    ["a*",   "aaa",  "zero or more"],
    ["a?",   "a",    "zero or one"],
    ["a{3}", "aaa",  "exactly 3"],
    ["a{2,4}","aaaa","2 to 4"],
    ["a{2,}", "aaaa","2 or more"],
);

for my $t (@tests) {
    my ($pat, $str, $desc) = @$t;
    printf "%-12s %-8s %s: %s\n", "/$pat/", $str, $desc,
        $str =~ /^$pat$/ ? "match" : "no match";
}

# =====================
# Possessive (via atomic group workaround)
# =====================

# Greedy backtracking example
my $str = "aaaaab";
if ($str =~ /a+b/) {
    printf "\nGreedy 'a+b' matched in: %s\n", $str;
}

# Atomic group — no backtracking
if ($str =~ /(?:a+)b/) {
    printf "Atomic 'a+b' matched\n";
}

# =====================
# Multiline
# =====================

my $multi = "Line 1\nLine 2\nLine 3\n";

my @lines_m = ($multi =~ /^Line \d+$/mg);
printf "\nMultiline matches: %d lines\n", scalar @lines_m;

# . matches newline with /s
if ($multi =~ /Line 1(.+)Line 3/s) {
    my $between = $1;
    $between =~ s/\n/\\n/g;
    printf "Between first and last: '%s'\n", $between;
}

# =====================
# Extended mode /x
# =====================

my $date_re = qr/
    (\d{4})   # year
    [-\/]     # separator
    (\d{1,2}) # month
    [-\/]     # separator
    (\d{1,2}) # day
/x;

for my $d ("2024-01-15", "2024/12/31", "2024-3-5") {
    if ($d =~ $date_re) {
        printf "Date: year=%s month=%02d day=%02d\n", $1, $2, $3;
    }
}
```

---

## Step 243: Character Classes

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# POSIX and Unicode character classes
# =====================

my @samples = (
    "Hello World",
    "hello123",
    "  spaces  ",
    "UPPERCASE",
    "mixed_Case_123",
    "!@#\$%^",
    "αβγδ",        # Greek
    "日本語",       # Japanese
    "café",        # accented
);

for my $s (@samples) {
    printf "%-20s  ", $s;
    printf "[alpha]"    if $s =~ /[[:alpha:]]/;
    printf "[digit]"    if $s =~ /[[:digit:]]/;
    printf "[space]"    if $s =~ /[[:space:]]/;
    printf "[upper]"    if $s =~ /[[:upper:]]/;
    printf "[lower]"    if $s =~ /[[:lower:]]/;
    printf "[punct]"    if $s =~ /[[:punct:]]/;
    printf "[word]"     if $s =~ /\w/;
    printf "\n";
}

# =====================
# Unicode properties
# =====================

printf "\n--- Unicode ---\n";
use feature 'unicode_strings';

my @words = ("hello", "WORLD", "Café", "日本", "αβγ", "123", "  ");
for my $w (@words) {
    printf "%-10s ", $w;
    printf "\\p{L}" if $w =~ /\p{L}/u;   # Letter
    printf "\\p{Lu}" if $w =~ /\p{Lu}/u; # Uppercase
    printf "\\p{Ll}" if $w =~ /\p{Ll}/u; # Lowercase
    printf "\\p{N}"  if $w =~ /\p{N}/u;  # Number
    printf "\\p{Z}"  if $w =~ /\p{Z}/u;  # Separator
    printf "\n";
}

# =====================
# Negated classes
# =====================

printf "\n--- Negated ---\n";
my $text = "Hello World 123!";
(my $no_alpha = $text) =~ s/[[:alpha:]]//g;
(my $no_digit = $text) =~ s/\d//g;
(my $no_space = $text) =~ s/\s//g;
(my $only_word = $text) =~ s/\W//g;

printf "No alpha: '%s'\n", $no_alpha;
printf "No digit: '%s'\n", $no_digit;
printf "No space: '%s'\n", $no_space;
printf "Only word: '%s'\n", $only_word;

# =====================
# Custom character classes
# =====================

my $hex_re   = qr/[0-9A-Fa-f]/;
my $ident_re = qr/[A-Za-z_][A-Za-z0-9_]*/;
my $phone_re = qr/[\d\s\-\+\(\)]+/;

for my $hex ("1A3F", "GHJ", "0xFF", "deadbeef") {
    printf "%-12s is %s hex string\n", $hex, ($hex =~ /^${hex_re}+$/ ? "a valid" : "NOT a valid");
}

for my $id ("hello", "my_var", "_private", "123bad", "valid_123") {
    printf "%-12s is %s identifier\n", $id, ($id =~ /^$ident_re$/ ? "a valid" : "NOT a valid");
}
```

---

## Step 244: Lookahead and Lookbehind

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Zero-width assertions
# =====================

# Positive lookahead (?=...)
my @prices = ("$10.99", "€20", "£5.50", "100USD", "50");
my @dollar = grep { /\d+(?=\s*USD|\$)/ } @prices;
printf "USD prices: %s\n", join(", ", @dollar);

# Extract number before USD
for my $p (@prices) {
    if ($p =~ /(\d+(?:\.\d+)?)(?=\s*USD)/) {
        printf "  Amount: %s\n", $1;
    }
}

# Negative lookahead (?!...)
my @words = qw(pre prefix preview premonition prize present);
my @not_prev = grep { /^pre(?!v)/ } @words;
printf "\nStarting with pre- (not prev-): %s\n", join(", ", @not_prev);

# Positive lookbehind (?<=...)
my $text = "I paid 100 dollars and 50 euros and 200 pounds";
my @amounts = ($text =~ /(?<=paid )\d+|\d+(?= dollars)/g);
printf "\nAmounts after 'paid': %s\n", join(", ", @amounts);

# Extract words after specific words
while ($text =~ /(?<=\b(?:paid|and) )(\d+)/g) {
    printf "  Found: %s\n", $1;
}

# Negative lookbehind (?<!...)
my @words2 = ("cat", "scat", "scatter", "cats", "concatenate");
my @no_s_before_cat = grep { /(?<!s)cat/ } @words2;
printf "\ncat not preceded by s: %s\n", join(", ", @no_s_before_cat);

# =====================
# Practical: Password validator
# =====================

sub validate_password {
    my $pwd = shift;
    my @errors;
    push @errors, "min 8 chars"     unless length($pwd) >= 8;
    push @errors, "needs uppercase" unless $pwd =~ /(?=.*[A-Z])/;
    push @errors, "needs lowercase" unless $pwd =~ /(?=.*[a-z])/;
    push @errors, "needs digit"     unless $pwd =~ /(?=.*\d)/;
    push @errors, "needs special"   unless $pwd =~ /(?=.*[!@#\$%^&*])/;
    return @errors;
}

printf "\n--- Password Validation ---\n";
for my $pwd ("abc", "password", "Password1", "Password1!", "P@ssw0rd!") {
    my @errors = validate_password($pwd);
    if (@errors) {
        printf "%-15s FAIL: %s\n", $pwd, join(", ", @errors);
    } else {
        printf "%-15s PASS\n", $pwd;
    }
}

# =====================
# Lookaround for split
# =====================

# Split on word boundary between digit and letter
my $mixed = "Hello123World456Test";
my @parts = split /(?<=\d)(?=[A-Za-z])|(?<=[A-Za-z])(?=\d)/, $mixed;
printf "\nSplit mixed: %s\n", join(" | ", @parts);

# Split camelCase
my $camel = "helloWorldFooBarBaz";
my @camel_parts = split /(?<=[a-z])(?=[A-Z])/, $camel;
printf "CamelCase split: %s\n", join(", ", @camel_parts);
```

---

## Step 245: Regex in Data Processing

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Parsing structured data with regex
# =====================

# Apache log parser
my @log_lines = (
    '127.0.0.1 - alice [10/Oct/2024:12:00:00 +0700] "GET /index.html HTTP/1.1" 200 1234',
    '10.0.0.1 - bob [11/Oct/2024:08:30:00 +0700] "POST /api/data HTTP/1.1" 201 56',
    '192.168.1.1 - - [11/Oct/2024:09:00:00 +0700] "GET /static/style.css HTTP/1.1" 304 0',
    '1.2.3.4 - charlie [12/Oct/2024:14:00:00 +0700] "DELETE /api/item/5 HTTP/1.1" 404 89',
);

my $log_re = qr/
    ^
    (\S+)           # ip
    \s+ \S+ \s+     # ident, auth
    (\S+)           # user
    \s+
    \[([^\]]+)\]    # date/time
    \s+
    "([A-Z]+)       # method
    \s+
    (\S+)           # path
    \s+ [^"]*"      # protocol
    \s+
    (\d+)           # status
    \s+
    (\d+)           # bytes
    $
/x;

printf "%-15s %-10s %-6s %-30s %s\n", "IP", "User", "Status", "Path", "Bytes";
printf "-" x 70, "\n";

my %stats;
for my $line (@log_lines) {
    if ($line =~ $log_re) {
        my ($ip, $user, $dt, $method, $path, $status, $bytes) = ($1,$2,$3,$4,$5,$6,$7);
        printf "%-15s %-10s %-6s %-30s %s\n", $ip, $user, $status, "$method $path", $bytes;
        $stats{$status}++;
        $stats{total_bytes} += $bytes;
    }
}

printf "\nStatus counts:\n";
printf "  %s: %d\n", $_, $stats{$_} for sort grep { /^\d/ } keys %stats;
printf "Total bytes: %d\n", $stats{total_bytes};

# =====================
# Email extraction
# =====================

my $email_text = q{
Contact us at info@example.com or support@company.org
For billing: billing@example.com (NOT: bad@@email or missing@)
Secondary: alice.jones+tag@subdomain.example.co.uk
};

my $email_re = qr/\b[A-Za-z0-9._%+\-]+\@[A-Za-z0-9.\-]+\.[A-Za-z]{2,}\b/;
my @emails = ($email_text =~ /$email_re/g);
printf "\nExtracted emails:\n";
printf "  %s\n", $_ for @emails;

# =====================
# URL extraction and parsing
# =====================

my $url_re = qr{
    (https?)                    # scheme
    ://
    ([A-Za-z0-9\-\.]+)         # host
    (?::(\d+))?                 # optional port
    (/[^?\s]*)?                 # path
    (?:\?([^\s\#]*))?           # query
    (?:\#(\S*))?                # fragment
}x;

my @urls = (
    "https://example.com/path?q=hello&page=1#section",
    "http://localhost:8080/api/v1/users",
    "https://sub.domain.co.uk:443/page",
);

for my $url (@urls) {
    if ($url =~ $url_re) {
        printf "\nURL: %s\n", $url;
        printf "  scheme: %s\n",   $1;
        printf "  host:   %s\n",   $2;
        printf "  port:   %s\n",   $3//"(default)";
        printf "  path:   %s\n",   $4//"(none)";
        printf "  query:  %s\n",   $5//"(none)";
        printf "  frag:   %s\n",   $6//"(none)";
    }
}
```

---

## Step 246: Regex Object (qr//)

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# qr// — compiled regex object
# =====================

# Create reusable patterns
my $ip_octet  = qr/(?:25[0-5]|2[0-4]\d|[01]?\d\d?)/;
my $ip_re     = qr/$ip_octet\.$ip_octet\.$ip_octet\.$ip_octet/;
my $ipv6_seg  = qr/[0-9A-Fa-f]{1,4}/;

for my $ip ("192.168.1.1", "10.0.0.1", "256.1.1.1", "0.0.0.0", "999.999.999.999") {
    printf "%-18s %s\n", $ip, ($ip =~ /^$ip_re$/ ? "valid IPv4" : "INVALID");
}

# =====================
# Pattern library
# =====================

my %patterns = (
    phone_th => qr/(?:0[689]\d{8}|02\d{7})/,     # Thai phone
    ssn      => qr/\d{3}-\d{2}-\d{4}/,            # US SSN format
    postcode => qr/\d{5}/,                         # Thai/US zip
    date_iso => qr/\d{4}-(?:0[1-9]|1[0-2])-(?:0[1-9]|[12]\d|3[01])/,
    time_24h => qr/(?:[01]\d|2[0-3]):[0-5]\d(?::[0-5]\d)?/,
    hex_color => qr/\#[0-9A-Fa-f]{6}(?:[0-9A-Fa-f]{2})?/,
    slug     => qr/[a-z0-9]+(?:-[a-z0-9]+)*/,
);

my @test_data = (
    ["0812345678",  "phone_th"],
    ["02-123-4567", "phone_th"],   # invalid format
    ["123-45-6789", "ssn"],
    ["10110",       "postcode"],
    ["2024-12-31",  "date_iso"],
    ["25:00",       "time_24h"],
    ["#FF5733",     "hex_color"],
    ["my-article-slug", "slug"],
    ["Invalid SLUG!", "slug"],
);

printf "\nPattern matching:\n";
for my $item (@test_data) {
    my ($val, $pat_name) = @$item;
    my $pat = $patterns{$pat_name};
    printf "  %-22s %-12s %s\n", $val, $pat_name, ($val =~ /^$pat$/ ? "MATCH" : "FAIL");
}

# =====================
# Dynamic pattern building
# =====================

sub build_keyword_re {
    my @keywords = @_;
    my $pattern = join "|", map { quotemeta $_ } @keywords;
    return qr/\b(?:$pattern)\b/i;
}

my $tech_re = build_keyword_re("perl", "python", "ruby", "javascript", "go", "rust");

my @sentences = (
    "I love Perl and Python",
    "JavaScript is popular",
    "Learning Go and Rust",
    "Java is not Go",
);

for my $s (@sentences) {
    my @found = ($s =~ /$tech_re/g);
    printf "%-35s => %s\n", $s, join(", ", @found) || "(none)";
}
```

---

## Step 247: Regex with Modifiers

```perl
#!/usr/bin/perl
use strict;
use warnings;
use feature 'say';

# =====================
# All modifier flags
# =====================

my $text = "Hello\nWORLD\nfoo bar\nbaz";

# /i — case insensitive
say "--- /i ---";
say "Match" if $text =~ /hello/i;

# /g — global
say "\n--- /g ---";
my @words = ($text =~ /\b\w+\b/g);
say "Words: @words";

# /m — multiline (^ and $ match line boundaries)
say "\n--- /m ---";
my @line_starts = ($text =~ /^\w+/mg);
say "Line starts: @line_starts";

# /s — single line (. matches \n)
say "\n--- /s ---";
if ($text =~ /Hello(.+)foo/s) {
    (my $mid = $1) =~ s/\n/\\n/g;
    say "Middle: '$mid'";
}

# /x — extended (whitespace and comments ignored)
say "\n--- /x ---";
my $date_re = qr/
    (\d{4})  # year
    -
    (\d{2})  # month
    -
    (\d{2})  # day
/x;

"2024-12-25" =~ $date_re;
printf "Date: year=%s month=%s day=%s\n", $1, $2, $3;

# /e — evaluate replacement as Perl code
say "\n--- /e ---";
my $math = "2+3 and 4*5 and 10/2";
(my $result = $math) =~ s/(\d+)([+*\/\-])(\d+)/eval("$1 $2 $3")/ge;
say "Math: $result";

# Convert to title case
my $sentence = "hello world from perl";
(my $title = $sentence) =~ s/\b(\w)/uc($1)/ge;
say "Title: $title";

# /r — non-destructive substitution (Perl 5.14+)
say "\n--- /r ---";
my $orig = "Hello World";
my $new  = $orig =~ s/World/Perl/r;
say "Orig: $orig";
say "New:  $new";

# Chain /r
my $processed = "  hello world  " =~ s/^\s+//r =~ s/\s+$//r =~ s/\b(\w)/uc($1)/ger;
say "Processed: '$processed'";

# /a, /u, /l — ASCII/Unicode/locale
say "\n--- /a Unicode ---";
my $uni = "Hello Café 123";
my @ascii_words = ($uni =~ /\b\w+\b/ag);  # ASCII only \w
my @uni_words   = ($uni =~ /\b\w+\b/ug);  # Unicode \w
say "ASCII words: @ascii_words";
say "Unicode words: @uni_words";
```

---

## Step 248: Regex Engine Internals

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Understanding backtracking
# =====================

# Catastrophic backtracking demo (safe version)
my $re_good = qr/^(\d+)+$/;   # CATASTROPHIC: avoid
my $re_safe = qr/^\d+$/;       # Safe

my $str = "12345678901234567890X";

# Time the safe version
my $start = time();
$str =~ $re_safe;
my $elapsed = time() - $start;
printf "Safe regex: %dms\n", $elapsed * 1000;

# =====================
# Regex debugging
# =====================

use re 'debug' if $ENV{DEBUG_RE};

# Without debug, show match positions
if ("hello world" =~ /(\w+)\s+(\w+)/) {
    printf "Full match: '%s'\n", $&;
    printf "Pre-match:  '%s'\n", $`;
    printf "Post-match: '%s'\n", $';
    printf "Group 1:    '%s'\n", $1;
    printf "Group 2:    '%s'\n", $2;
}

# pos() — current position in /g
my $data = "cat bat hat rat";
while ($data =~ /\b(\w)at\b/g) {
    printf "Found '%s' at position %d\n", $1, pos($data) - length($&);
}

# Reset pos
pos($data) = 0;

# =====================
# Lookahead for split
# =====================

# Split but keep delimiter
my $csv = "a,b,,c,d";
my @parts = split /,/, $csv, -1;  # -1 keeps trailing empty
printf "\nCSV parts: %s\n", join(" | ", map { "'$_'" } @parts);

# Split on lookahead
my $text = "one1two2three3four";
my @split_on_digit = split /(?=\d)/, $text;
printf "Split before digit: %s\n", join(", ", @split_on_digit);

# =====================
# Regex in hash keys
# =====================

my %handler = (
    qr/^GET /  => sub { "Handling GET: $_[0]" },
    qr/^POST / => sub { "Handling POST: $_[0]" },
    qr/^DELETE / => sub { "Handling DELETE: $_[0]" },
);

sub dispatch {
    my $request = shift;
    for my $re (keys %handler) {
        if ($request =~ $re) {
            return $handler{$re}->($request);
        }
    }
    return "Unknown: $request";
}

for my $req ("GET /index", "POST /api/data", "DELETE /item/5", "PUT /update") {
    printf "%s\n", dispatch($req);
}

# =====================
# Conditional regex (?if true|false)
# =====================

my @items = ("USD 100", "EUR 200", "100 THB", "GBP 50");
for my $item (@items) {
    if ($item =~ /^(?:(USD|EUR|GBP)\s+)?(\d+)(?:\s+(THB))?$/) {
        my $currency = $1 // $3 // "???";
        printf "Currency: %-5s Amount: %s\n", $currency, $2;
    }
}
```

---

## Step 249: Practical Regex Applications

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Template engine with regex
# =====================

sub render_template {
    my ($template, $vars) = @_;
    
    # {{= variable }}
    $template =~ s/\{\{=\s*(\w+)\s*\}\}/$vars->{$1}\/\/""/ge;
    
    # {{# if var }}...{{/ if }}
    $template =~ s/
        \{\{\#\s*if\s+(\w+)\s*\}\}
        (.*?)
        \{\{\/\s*if\s*\}\}
    /$vars->{$1} ? $2 : ""/gxes;
    
    # {{# unless var }}...{{/ unless }}
    $template =~ s/
        \{\{\#\s*unless\s+(\w+)\s*\}\}
        (.*?)
        \{\{\/\s*unless\s*\}\}
    /!$vars->{$1} ? $2 : ""/gxes;
    
    return $template;
}

my $tmpl = q{
Hello, {{= name }}!
{{# if admin }}
You are an admin.
{{/ if }}
{{# unless admin }}
You are a regular user.
{{/ unless }}
Your score is {{= score }}.
};

print render_template($tmpl, { name => "Alice", admin => 1, score => 95 });
print "-" x 30, "\n";
print render_template($tmpl, { name => "Bob", admin => 0, score => 72 });

# =====================
# Config file parser
# =====================

sub parse_ini {
    my $text = shift;
    my (%config, $section);
    
    for my $line (split /\n/, $text) {
        $line =~ s/#.*//;     # remove comments
        $line =~ s/^\s+|\s+$//g;  # trim
        next unless length $line;
        
        if ($line =~ /^\[(\w+)\]$/) {
            $section = $1;
        } elsif ($section && $line =~ /^(\w+)\s*=\s*(.*)$/) {
            $config{$section}{$1} = $2;
        }
    }
    
    return %config;
}

my $ini = q{
# Database configuration
[database]
host = localhost
port = 5432
name = myapp_db

[cache]
driver = redis  # use redis
ttl    = 3600

[app]
debug  = 1
secret = s3cr3t_k3y
};

my %cfg = parse_ini($ini);
for my $section (sort keys %cfg) {
    printf "[%s]\n", $section;
    printf "  %-10s = %s\n", $_, $cfg{$section}{$_} for sort keys %{$cfg{$section}};
}

# =====================
# Markdown parser (simple)
# =====================

sub parse_markdown {
    my $text = shift;
    
    # Headers
    $text =~ s/^### (.+)$/<h3>$1<\/h3>/mg;
    $text =~ s/^## (.+)$/<h2>$1<\/h2>/mg;
    $text =~ s/^# (.+)$/<h1>$1<\/h1>/mg;
    
    # Bold and italic
    $text =~ s/\*\*\*(.+?)\*\*\*/<strong><em>$1<\/em><\/strong>/g;
    $text =~ s/\*\*(.+?)\*\*/<strong>$1<\/strong>/g;
    $text =~ s/\*(.+?)\*/<em>$1<\/em>/g;
    
    # Code
    $text =~ s/`([^`]+)`/<code>$1<\/code>/g;
    
    # Links
    $text =~ s/\[([^\]]+)\]\(([^)]+)\)/<a href="$2">$1<\/a>/g;
    
    return $text;
}

my $md = q{# Hello World
## Introduction
This is **bold** and *italic* text.
Here is `inline code` and a [link](https://example.com).
### Section
***bold and italic***
};

print "\n--- Markdown to HTML ---\n";
print parse_markdown($md);
```

---

## Step 250: โปรแกรมสรุป — Text Analysis Engine

```perl
#!/usr/bin/perl
# text_analyzer.pl — Comprehensive text analysis using regex
use strict;
use warnings;

{
package TextAnalyzer;

sub new {
    my ($class, $text) = @_;
    return bless { text => $text, _cache => {} }, $class;
}

sub text  { $_[0]->{text} }

sub word_count {
    my $self = shift;
    return $self->{_cache}{word_count} //= do {
        my @words = ($self->text =~ /\b\w+\b/g);
        scalar @words;
    };
}

sub char_count  { length($_[0]->text) }
sub line_count  { scalar(()= $_[0]->text =~ /\n/g) + 1 }

sub sentence_count {
    my @s = ($_[0]->text =~ /[.!?]+/g);
    return scalar @s;
}

sub unique_words {
    my $self = shift;
    return $self->{_cache}{unique_words} //= do {
        my %seen;
        $seen{lc $_}++ for ($self->text =~ /\b([a-zA-Z]+)\b/g);
        \%seen;
    };
}

sub unique_word_count { scalar keys %{$_[0]->unique_words} }

sub top_words {
    my ($self, $n) = @_;
    $n //= 10;
    my $uw = $self->unique_words;
    return (sort { $uw->{$b} <=> $uw->{$a} } keys %$uw)[0..$n-1];
}

sub avg_word_length {
    my $self = shift;
    my @words = ($self->text =~ /\b([a-zA-Z]+)\b/g);
    return 0 unless @words;
    my $total = 0;
    $total += length($_) for @words;
    return $total / @words;
}

sub extract_emails {
    return ($_[0]->text =~ /\b[A-Za-z0-9._%+\-]+\@[A-Za-z0-9.\-]+\.[A-Za-z]{2,}\b/g);
}

sub extract_urls {
    return ($_[0]->text =~ m{https?://[A-Za-z0-9\-._~:/?#\[\]@!$&'()*+,;=%]+}g);
}

sub extract_numbers {
    return ($_[0]->text =~ /\b\d+(?:\.\d+)?\b/g);
}

sub extract_hashtags {
    return ($_[0]->text =~ /#([A-Za-z]\w*)/g);
}

sub extract_mentions {
    return ($_[0]->text =~ /\@([A-Za-z]\w*)/g);
}

sub reading_time {
    my $self = shift;
    my $words = $self->word_count;
    my $mins  = $words / 200;  # 200 WPM average
    return $mins < 1 ? "< 1 minute" : sprintf("%.0f minutes", $mins);
}

sub flesch_score {
    my $self = shift;
    my $words = $self->word_count;
    my $sents = $self->sentence_count || 1;
    
    # Count syllables (approximate)
    my $syllables = 0;
    for my $word ($self->text =~ /\b([a-zA-Z]+)\b/g) {
        $word = lc $word;
        my @v = ($word =~ /[aeiouy]+/g);
        $syllables += @v || 1;
    }
    
    my $score = 206.835
        - 1.015 * ($words / $sents)
        - 84.6  * ($syllables / ($words||1));
    
    return $score;
}

sub readability {
    my $score = $_[0]->flesch_score;
    return "Very Easy"   if $score >= 90;
    return "Easy"        if $score >= 80;
    return "Fairly Easy" if $score >= 70;
    return "Standard"    if $score >= 60;
    return "Fairly Hard" if $score >= 50;
    return "Hard"        if $score >= 30;
    return "Very Hard";
}

sub keyword_density {
    my ($self, $keyword) = @_;
    my $re    = qr/\b\Q$keyword\E\b/i;
    my $count = () = $self->text =~ /$re/g;
    return $count / ($self->word_count || 1) * 100;
}

sub report {
    my $self = shift;
    printf "=== Text Analysis Report ===\n\n";
    printf "Basic Stats:\n";
    printf "  Words:         %d\n",   $self->word_count;
    printf "  Unique words:  %d\n",   $self->unique_word_count;
    printf "  Characters:    %d\n",   $self->char_count;
    printf "  Lines:         %d\n",   $self->line_count;
    printf "  Sentences:     %d\n",   $self->sentence_count;
    printf "  Avg word len:  %.1f\n", $self->avg_word_length;
    printf "  Reading time:  %s\n",   $self->reading_time;
    printf "  Readability:   %s (%.1f)\n", $self->readability, $self->flesch_score;

    printf "\nTop 10 Words:\n";
    my @top = $self->top_words(10);
    my $uw  = $self->unique_words;
    printf "  %-15s %d\n", $_, $uw->{$_} for @top;

    my @emails = $self->extract_emails;
    printf "\nEmails (%d): %s\n", scalar @emails, join(", ", @emails) || "(none)";

    my @urls = $self->extract_urls;
    printf "URLs (%d): %s\n", scalar @urls, join(", ", @urls) || "(none)";

    my @tags = $self->extract_hashtags;
    printf "Hashtags: %s\n", join(", ", @tags) || "(none)" if @tags;

    my @mentions = $self->extract_mentions;
    printf "Mentions: %s\n", join(", ", @mentions) || "(none)" if @mentions;
}
}

package main;

my $sample_text = q{
Perl is a highly capable, feature-rich programming language with over 30 years
of development. Perl runs on over 100 platforms from portables to mainframes and
is suitable for both rapid prototyping and large-scale development projects.

Perl is used for system administration, web development, network programming,
GUI development, and more. Contact us at perl@example.com or info@perl.org.

Visit https://www.perl.org for more information.
Follow us @perl_lang #Perl #Programming on social media.

"Easy things should be easy, and hard things should be possible." — Larry Wall

The numbers 42, 3.14, and 100 are important in many contexts.
};

my $analyzer = TextAnalyzer->new($sample_text);
$analyzer->report;

printf "\nKeyword density for 'perl': %.1f%%\n", $analyzer->keyword_density("perl");
printf "Keyword density for 'the':  %.1f%%\n",  $analyzer->keyword_density("the");

print "\nText analysis engine complete!\n";
```

---

## สรุป Part 25

ใน Part นี้คุณได้เรียนรู้:
- ✅ Regex fundamentals: quantifiers, anchors, alternation
- ✅ Greedy vs lazy matching
- ✅ Character classes: POSIX, Unicode properties
- ✅ Lookahead and lookbehind (positive/negative)
- ✅ Regex objects with qr//
- ✅ All modifiers: i, g, m, s, x, e, r
- ✅ Regex in data processing (log parser, INI config)
- ✅ Template engine with regex
- ✅ Markdown parser
- ✅ Complete text analysis engine

**ถัดไป: [Part 26 — File I/O Advanced](part_26.md)**
