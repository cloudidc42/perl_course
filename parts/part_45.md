# Part 45: Advanced Regex & Text Processing
## Steps 441-450: Patterns, Parsers, Templates, NLP

---

## Step 441: Advanced Regex Techniques

```perl
#!/usr/bin/perl
use strict;
use warnings;

printf "=== Advanced Regex Techniques ===\n\n";

# Named captures
printf "1. Named captures:\n";
my $date_str = "2024-01-15 was a Monday";
if ($date_str =~ /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/) {
    printf "  year=%s month=%s day=%s\n\n", $+{year}, $+{month}, $+{day};
}

# Non-greedy
printf "2. Non-greedy vs greedy:\n";
my $html = "<b>bold</b> and <i>italic</i>";
(my $greedy)     = $html =~ /(<.+>)/;
my @non_greedy   = $html =~ /(<.+?>)/g;
printf "  greedy: %s\n", $greedy;
printf "  non-greedy: %s\n\n", join(", ", @non_greedy);

# Lookahead/lookbehind
printf "3. Lookahead/lookbehind:\n";
my $text = "foo123bar456baz789";
my @nums_before_bar = $text =~ /(\d+)(?=bar)/g;
my @nums_after_foo  = $text =~ /(?<=foo)(\d+)/g;
printf "  Digits before 'bar': %s\n", join(", ", @nums_before_bar);
printf "  Digits after 'foo':  %s\n\n", join(", ", @nums_after_foo);

# Negative lookahead/lookbehind
printf "4. Negative lookahead:\n";
my @words_not_before_world = ("hello world", "hello Perl", "hello there") ;
for my $s (@words_not_before_world) {
    if ($s =~ /hello(?!\s+world)/) {
        printf "  '$s' -> matches (not followed by 'world')\n";
    }
}

# Atomic groups and possessive
printf "\n5. Alternation optimization:\n";
for my $word (qw(cat car can cap cab cog)) {
    if ($word =~ /^c(?:a[trn]|o[gd])$/) {
        printf "  Matched: $word\n";
    }
}

# Global substitution with transform
printf "\n6. s/// with /e (eval):\n";
my $template = "Price: \$price_usd USD = \$price_gbp GBP";
my %prices = (price_usd => 100, price_gbp => 79);
(my $filled = $template) =~ s/\$(\w+)/$prices{$1} \/\/ "N\/A"/ge;
printf "  %s\n\n", $filled;

# Regex objects
printf "7. Compiled regex (qr//):\n";
my @patterns = map { qr/$_/i } qw(perl python ruby javascript);
my @sentences = (
    "I love Perl programming",
    "Python is also great",
    "JavaScript for web",
    "Go is fast"
);
for my $s (@sentences) {
    for my $pat (@patterns) {
        if ($s =~ $pat) {
            printf "  '%s' -> matched /%s/\n", $s, $pat;
        }
    }
}

# Split with captures
printf "\n8. Split with captures:\n";
my $csv_with_quotes = 'one,"two,three",four,"five"';
my @fields;
while ($csv_with_quotes =~ /(?:"([^"]*)"|([^,]+))(?:,|$)/g) {
    push @fields, defined $1 ? $1 : $2;
}
printf "  Fields: %s\n\n", join(" | ", @fields);

# Recursive-style matching (balanced parens)
printf "9. Balanced bracket extraction:\n";
my $expr = "(a+(b*c)+(d/(e-f)))";
my @groups;
while ($expr =~ /\(([^()]*)\)/g) {
    push @groups, $1;
}
printf "  Innermost groups: %s\n\n", join(", ", @groups);

# Unicode
printf "10. Unicode patterns:\n";
use utf8;
my $unicode_text = "สวัสดี Hello 你好 مرحبا";
my @words_unicode = $unicode_text =~ /(\S+)/g;
printf "  Words: %d\n", scalar @words_unicode;
```

---

## Step 442: Lexer & Tokenizer

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Lexer;

use constant {
    TOK_NUMBER  => "NUMBER",
    TOK_STRING  => "STRING",
    TOK_IDENT   => "IDENT",
    TOK_OP      => "OP",
    TOK_PUNCT   => "PUNCT",
    TOK_KEYWORD => "KEYWORD",
    TOK_NEWLINE => "NEWLINE",
    TOK_EOF     => "EOF",
    TOK_COMMENT => "COMMENT",
};

my %KEYWORDS = map { $_ => 1 } qw(
    if else elsif while for foreach do next last return
    sub my our local use require print say
);

sub new {
    my ($class, $input) = @_;
    return bless {
        input  => $input,
        pos    => 0,
        line   => 1,
        col    => 1,
        tokens => [],
    }, $class;
}

sub tokenize {
    my $self = shift;
    while ($self->{pos} < length($self->{input})) {
        my $tok = $self->_next_token;
        push @{$self->{tokens}}, $tok if $tok;
    }
    push @{$self->{tokens}}, { type => TOK_EOF, value => "", line => $self->{line} };
    return @{$self->{tokens}};
}

sub _next_token {
    my $self = shift;
    my $rest = substr($self->{input}, $self->{pos});
    
    # Whitespace (skip, track newlines)
    if ($rest =~ /^(\n)/) {
        $self->{pos}++; $self->{line}++; $self->{col} = 1;
        return { type => TOK_NEWLINE, value => "\n", line => $self->{line}-1 };
    }
    if ($rest =~ /^([ \t\r]+)/) {
        $self->{pos} += length($1); $self->{col} += length($1);
        return undef;
    }
    
    # Comment
    if ($rest =~ /^(#[^\n]*)/) {
        my $tok = { type => TOK_COMMENT, value => $1, line => $self->{line} };
        $self->{pos} += length($1);
        return $tok;
    }
    
    # String
    if ($rest =~ /^('(?:[^'\\]|\\.)*'|"(?:[^"\\]|\\.)*")/) {
        my $tok = { type => TOK_STRING, value => $1, line => $self->{line} };
        $self->{pos} += length($1);
        return $tok;
    }
    
    # Number
    if ($rest =~ /^(0x[0-9a-fA-F]+|\d+(?:\.\d+)?(?:[eE][+-]?\d+)?)/) {
        my $tok = { type => TOK_NUMBER, value => $1, line => $self->{line} };
        $self->{pos} += length($1);
        return $tok;
    }
    
    # Identifier or keyword
    if ($rest =~ /^([a-zA-Z_]\w*)/) {
        my $word = $1;
        my $type = $KEYWORDS{$word} ? TOK_KEYWORD : TOK_IDENT;
        my $tok = { type => $type, value => $word, line => $self->{line} };
        $self->{pos} += length($word);
        return $tok;
    }
    
    # Multi-char operators
    if ($rest =~ /^(=>|->|==|!=|<=|>=|=~|!~|\|\||&&|<<|>>|\.\.)/) {
        my $tok = { type => TOK_OP, value => $1, line => $self->{line} };
        $self->{pos} += length($1);
        return $tok;
    }
    
    # Single-char
    my $c = substr($rest, 0, 1);
    my $type = ($c =~ /[+\-*\/=<>!&|^~%]/) ? TOK_OP : TOK_PUNCT;
    my $tok = { type => $type, value => $c, line => $self->{line} };
    $self->{pos}++;
    return $tok;
}
}

package main;

printf "=== Lexer / Tokenizer ===\n\n";

my $code = <<'END';
sub greet {
    my ($name) = @_;
    if ($name eq "world") {
        return "Hello, World!";  # classic
    }
    return "Hi, $name!";
}
my $x = 42 + 3.14;
END

my $lexer = Lexer->new($code);
my @tokens = $lexer->tokenize;

# Exclude newlines and comments for clean display
my @significant = grep { $_->{type} !~ /NEWLINE|COMMENT/ } @tokens;
printf "%-10s %s\n", "Type", "Value";
printf "%s\n", "-" x 30;
for my $tok (@significant) {
    next if $tok->{type} eq "EOF";
    printf "%-10s %s\n", $tok->{type}, $tok->{value};
}

# Token stats
printf "\nToken statistics:\n";
my %counts;
$counts{$_->{type}}++ for @tokens;
printf "  %-12s %d\n", $_, $counts{$_} for sort keys %counts;
```

---

## Step 443: Expression Parser

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package ExprParser;

# Grammar:
#   expr   = term (('+' | '-') term)*
#   term   = factor (('*' | '/') factor)*
#   factor = number | '(' expr ')' | '-' factor
#   number = [0-9]+('.'[0-9]+)?

sub new {
    my ($class, $input) = @_;
    $input =~ s/\s+//g;
    return bless { input => $input, pos => 0 }, $class;
}

sub parse   { my $self = shift; my $r = $self->_expr; $r }
sub _peek   { substr($_[0]->{input}, $_[0]->{pos}, 1) }
sub _consume { my($s,$n)=@_; $n//=1; my $c=substr($s->{input},$s->{pos},$n); $s->{pos}+=$n; $c }
sub _at_end { $_[0]->{pos} >= length($_[0]->{input}) }

sub _expr {
    my $self = shift;
    my $left = $self->_term;
    while (!$self->_at_end && $self->_peek =~ /[+\-]/) {
        my $op = $self->_consume;
        my $right = $self->_term;
        $left = { op => $op, left => $left, right => $right };
    }
    return $left;
}

sub _term {
    my $self = shift;
    my $left = $self->_factor;
    while (!$self->_at_end && $self->_peek =~ /[*\/]/) {
        my $op = $self->_consume;
        my $right = $self->_factor;
        $left = { op => $op, left => $left, right => $right };
    }
    return $left;
}

sub _factor {
    my $self = shift;
    if ($self->_peek eq "-") {
        $self->_consume;
        return { op => "neg", operand => $self->_factor };
    }
    if ($self->_peek eq "(") {
        $self->_consume;
        my $expr = $self->_expr;
        $self->_consume if $self->_peek eq ")";
        return $expr;
    }
    return $self->_number;
}

sub _number {
    my $self = shift;
    my $start = $self->{pos};
    while (!$self->_at_end && $self->_peek =~ /[\d.]/) {
        $self->_consume;
    }
    return substr($self->{input}, $start, $self->{pos}-$start) + 0;
}

sub eval {
    my ($class_or_self, $node) = @_;
    # Allow call as class or instance
    if (!ref $node) { $node = $class_or_self; }
    
    return $node unless ref $node;
    
    if (ref $node eq "HASH") {
        if (defined $node->{op}) {
            my $op = $node->{op};
            if ($op eq "neg") { return -$class_or_self->eval($node->{operand}) }
            my $l = $class_or_self->eval($node->{left});
            my $r = $class_or_self->eval($node->{right});
            return $op eq "+" ? $l+$r : $op eq "-" ? $l-$r :
                   $op eq "*" ? $l*$r : $op eq "/" ? ($r ? $l/$r : 0) : 0;
        }
    }
    return $node+0;
}

sub to_string {
    my ($class, $node) = @_;
    return "$node" unless ref $node eq "HASH";
    if ($node->{op} eq "neg") { return "(-" . $class->to_string($node->{operand}) . ")" }
    return sprintf "(%s %s %s)",
        $class->to_string($node->{left}), $node->{op}, $class->to_string($node->{right});
}
}

package main;

printf "=== Expression Parser ===\n\n";

my @expressions = (
    "2 + 3",
    "10 - 4 * 2",
    "(10 - 4) * 2",
    "3.14 * 2 * 2",
    "100 / (5 + 5)",
    "-(3 + 4) * 2",
    "1 + 2 + 3 + 4 + 5",
    "2 * (3 + (4 * 5))",
);

printf "%-30s  %-40s  %s\n", "Expression", "AST", "Result";
printf "%s\n", "-" x 80;
for my $expr (@expressions) {
    my $parser = ExprParser->new($expr);
    my $ast    = $parser->parse;
    my $result = ExprParser->eval($ast);
    my $ast_str = ExprParser->to_string($ast);
    printf "%-30s  %-40s  %g\n", $expr, $ast_str, $result;
}
```

---

## Step 444: Template Engine

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Template;

sub new {
    my ($class, %opts) = @_;
    return bless {
        delimiters => $opts{delimiters} // ["{{", "}}"],
        templates  => {},
        helpers    => {},
        partials   => {},
    }, $class;
}

sub add_helper {
    my ($self, $name, $fn) = @_;
    $self->{helpers}{$name} = $fn;
}

sub add_partial {
    my ($self, $name, $tmpl) = @_;
    $self->{partials}{$name} = $tmpl;
}

sub render {
    my ($self, $template, $data) = @_;
    $data //= {};
    my $output = $template;
    
    # Partials: {{> partial_name}}
    $output =~ s/\{\{>\s*(\w+)\s*\}\}/$self->_render_partial($1, $data)/ge;
    
    # Blocks: {{#section}}...{{/section}}
    $output =~ s/\{\{#(\w+)\}\}(.*?)\{\{\/\1\}\}/$self->_render_block($1, $2, $data)/gse;
    
    # Unless: {{^key}}...{{/key}}
    $output =~ s/\{\^\s*(\w+)\s*\}\}(.*?)\{\{\/\1\}\}/$self->_render_unless($1, $2, $data)/gse;
    
    # Helpers: {{helper arg}}
    $output =~ s/\{\{(\w+)\s+([^}]+)\}\}/$self->_call_helper($1, $2, $data)/ge;
    
    # Variables: {{var}} or {{var.path}}
    $output =~ s/\{\{(\w[\w.]*)\}\}/$self->_resolve($1, $data)/ge;
    
    return $output;
}

sub _resolve {
    my ($self, $path, $data) = @_;
    my @parts = split /\./, $path;
    my $val = $data;
    for my $p (@parts) {
        return "" unless defined $val;
        $val = ref $val eq "HASH" ? $val->{$p} : undef;
    }
    return defined $val ? _escape_html($val) : "";
}

sub _render_block {
    my ($self, $key, $inner, $data) = @_;
    my $val = $data->{$key};
    return "" unless $val;
    
    if (ref $val eq "ARRAY") {
        my $out = "";
        for my $item (@$val) {
            my $ctx = ref $item eq "HASH" ? { %$data, %$item } : { %$data, "." => $item };
            $out .= $self->render($inner, $ctx);
        }
        return $out;
    }
    return $self->render($inner, { %$data, %{ref $val eq "HASH" ? $val : {}} });
}

sub _render_unless {
    my ($self, $key, $inner, $data) = @_;
    return $data->{$key} ? "" : $self->render($inner, $data);
}

sub _render_partial {
    my ($self, $name, $data) = @_;
    return "" unless my $partial = $self->{partials}{$name};
    return $self->render($partial, $data);
}

sub _call_helper {
    my ($self, $name, $arg, $data) = @_;
    return "" unless my $helper = $self->{helpers}{$name};
    $arg =~ s/^\s+|\s+$//g;
    my $val = $data->{$arg} // $arg;
    return $helper->($val, $data);
}

sub _escape_html {
    my $s = shift // "";
    $s =~ s/&/&amp;/g; $s =~ s/</&lt;/g; $s =~ s/>/&gt;/g;
    $s =~ s/"/&quot;/g; $s =~ s/'/&#39;/g;
    return $s;
}
}

package main;

printf "=== Template Engine ===\n\n";

my $tmpl = Template->new;

# Register helpers
$tmpl->add_helper("upper", sub { uc $_[0] });
$tmpl->add_helper("lower", sub { lc $_[0] });
$tmpl->add_helper("date",  sub {
    my @t = localtime(time);
    sprintf "%04d-%02d-%02d", $t[5]+1900, $t[4]+1, $t[3]
});

# Register partials
$tmpl->add_partial("greeting", "Hello, {{name}}!");
$tmpl->add_partial("footer",   "-- Generated on {{date}} --");

# Complex template
my $template = <<'TMPL';
{{> greeting}}

Users ({{count}} total):
{{#users}}
  - {{upper name}} <{{email}}>
    Roles: {{#roles}}[{{.}}]{{/roles}}
{{/users}}

{{^premium}}
Note: Upgrade for premium features.
{{/premium}}

{{> footer}}
TMPL

my $data = {
    name    => "Admin",
    count   => 3,
    date    => "2024-01-08",
    premium => 0,
    users   => [
        { name => "alice",  email => "alice\@test.com", roles => ["admin","user"] },
        { name => "bob",    email => "bob\@test.com",   roles => ["user"] },
        { name => "carol",  email => "carol\@test.com", roles => ["user","mod"] },
    ],
};

my $result = $tmpl->render($template, $data);
print $result;
```

---

## Step 445: Text Statistics & NLP Basics

```perl
#!/usr/bin/perl
use strict;
use warnings;
use List::Util qw(sum max min);

{
package TextStats;

my %STOP_WORDS = map { $_ => 1 } qw(
    a an the and or but in on at to for of with is are was were be been
    it its this that these those i me my we our you your he she him her
    they them their what which who whom when where why how all each every
    both few more most other some such no nor not only own same so than too
    very just can will would should could may might shall do does did
);

sub new {
    my ($class, $text) = @_;
    my $self = bless { text => $text }, $class;
    $self->_analyze;
    return $self;
}

sub _analyze {
    my $self = shift;
    my $text = $self->{text};
    
    # Sentences
    $self->{sentences} = [ split /(?<=[.!?])\s+/, $text ];
    
    # Words (lowercase, no punctuation)
    my @raw_words = map { lc } ($text =~ /\b([a-zA-Z]+(?:'[a-zA-Z]+)?)\b/g);
    $self->{word_count} = scalar @raw_words;
    
    # Word frequency
    my %freq;
    $freq{$_}++ for @raw_words;
    $self->{word_freq} = \%freq;
    
    # Significant words (excluding stop words)
    $self->{keywords} = { map { $_ => $freq{$_} }
        grep { !$STOP_WORDS{$_} && length($_) > 2 } keys %freq };
    
    # Characters
    $self->{char_count}       = length($text);
    $self->{char_no_space}    = () = $text =~ /\S/g;
    
    # Paragraphs
    $self->{paragraphs}       = [ split /\n\n+/, $text ];
    
    # Average word length
    my $total_len = sum(map { length } @raw_words) // 0;
    $self->{avg_word_length}  = @raw_words ? $total_len / @raw_words : 0;
    
    # Average sentence length in words
    $self->{avg_sent_length}  = @{$self->{sentences}} ?
        $self->{word_count} / @{$self->{sentences}} : 0;
    
    # Flesch-Kincaid readability approximation
    my $syllables = sum(map { _count_syllables($_) } @raw_words) // 0;
    my $n_sents   = scalar @{$self->{sentences}} || 1;
    $self->{flesch_score}     = 206.835
        - 1.015 * ($self->{word_count} / $n_sents)
        - 84.6  * ($syllables / ($self->{word_count}||1));
}

sub _count_syllables {
    my $word = lc shift;
    $word =~ s/e$//;
    my @vowels = $word =~ /[aeiouy]+/g;
    return @vowels || 1;
}

sub top_words {
    my ($self, $n) = @_;
    $n //= 10;
    return (sort { $self->{keywords}{$b} <=> $self->{keywords}{$a} }
            keys %{$self->{keywords}})[0..$n-1];
}

sub concordance {
    my ($self, $word) = @_;
    $word = lc $word;
    my @results;
    while ($self->{text} =~ /(\S+\s+\S+\s+)\b$word\b(\s+\S+\s+\S+)/gi) {
        push @results, { before => $1, word => $word, after => $2 };
    }
    return @results;
}

sub ngrams {
    my ($self, $n) = @_;
    $n //= 2;
    my @words = map { lc } ($self->{text} =~ /\b([a-zA-Z]+)\b/g);
    my %ngrams;
    for my $i (0..$#words-$n+1) {
        my $gram = join(" ", @words[$i..$i+$n-1]);
        $ngrams{$gram}++;
    }
    return %ngrams;
}

sub summary {
    my $self = shift;
    return {
        chars             => $self->{char_count},
        chars_no_space    => $self->{char_no_space},
        words             => $self->{word_count},
        unique_words      => scalar keys %{$self->{word_freq}},
        sentences         => scalar @{$self->{sentences}},
        paragraphs        => scalar @{$self->{paragraphs}},
        avg_word_length   => sprintf("%.2f", $self->{avg_word_length}),
        avg_sent_length   => sprintf("%.1f", $self->{avg_sent_length}),
        flesch_score      => sprintf("%.1f", $self->{flesch_score}),
    };
}
}

package main;

printf "=== Text Statistics & NLP ===\n\n";

my $text = <<'END';
Perl is a high-level, general-purpose, interpreted, dynamic programming language.
Originally developed by Larry Wall in 1987, Perl has evolved significantly over the decades.
It borrows features from C, shell script, AWK, and sed.

Perl is well known for its regular expression support and string manipulation capabilities.
The language is used for web development, system administration, network programming, and bioinformatics.
Many large websites and web applications have been built using Perl and CGI scripting.

The Perl community values creativity and pragmatism above all. There is more than one way to do it.
This philosophy, often called TIMTOWTDI, distinguishes Perl from more opinionated languages like Python.
END

my $stats = TextStats->new($text);
my $sum = $stats->summary;

printf "Document Statistics:\n";
printf "  %-20s %s\n", $_, $sum->{$_} for sort keys %$sum;

printf "\nTop 10 Keywords:\n";
printf "  %-20s %d\n", $_, $stats->{keywords}{$_} for $stats->top_words(10);

printf "\nBigrams (count >= 2):\n";
my %bigrams = $stats->ngrams(2);
for my $gram (sort { $bigrams{$b} <=> $bigrams{$a} } keys %bigrams) {
    last if $bigrams{$gram} < 2;
    printf "  %-30s %d\n", $gram, $bigrams{$gram};
}

printf "\nConcordance for 'perl':\n";
for my $c ($stats->concordance("perl")) {
    printf "  ...%s[%s]%s...\n", $c->{before}, $c->{word}, $c->{after};
}

printf "\nReadability: Flesch score %.1f ", $stats->{flesch_score};
my $score = $stats->{flesch_score};
printf "(%s)\n", $score >= 70 ? "Easy" : $score >= 50 ? "Standard" : "Difficult";
```

---

## Step 446-450: Pattern Matching, Slug/Search, Diff, Highlight, Capstone

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Fuzzy search (edit distance)
printf "=== Fuzzy Search ===\n\n";

{
package FuzzyMatch;

sub levenshtein {
    my ($a, $b) = @_;
    my @aa = split //, lc $a;
    my @bb = split //, lc $b;
    my @dp = map { [$_+0] } 0..@aa;
    
    for my $j (0..$#bb) {
        $dp[$_][$j+1] = $dp[$_][$j] for 0..$#dp;
    }
    
    my (@m) = ([0..$#bb+0]);
    for my $j (1..@bb) { $m[0][$j] = $j }
    
    for my $i (1..@aa) {
        $m[$i][0] = $i;
        for my $j (1..@bb) {
            my $cost = $aa[$i-1] eq $bb[$j-1] ? 0 : 1;
            $m[$i][$j] = (sort { $a<=>$b }
                $m[$i-1][$j]+1,
                $m[$i][$j-1]+1,
                $m[$i-1][$j-1]+$cost
            )[0];
        }
    }
    return $m[-1][-1];
}

sub similarity {
    my ($a, $b) = @_;
    my $max = length($a) > length($b) ? length($a) : length($b);
    return 0 unless $max;
    return 1 - levenshtein($a, $b) / $max;
}

sub search {
    my ($class, $query, @candidates) = @_;
    return map { { word => $_, score => $class->similarity($query, $_) } }
           sort { $class->similarity($query, $b) <=> $class->similarity($query, $a) }
           @candidates;
}
}

my @words = qw(perl pearl earl hurl curl peril peal farm harm charm);
printf "Fuzzy search for 'pearl':\n";
for my $r (FuzzyMatch->search("pearl", @words)) {
    printf "  %-12s score=%.3f\n", $r->{word}, $r->{score} if $r->{score} > 0.4;
}

# Slug generator
printf "\n=== Slug Generator ===\n\n";

sub slugify {
    my $str = lc shift;
    $str =~ s/[áàâä]/a/g; $str =~ s/[éèêë]/e/g;
    $str =~ s/[íìîï]/i/g; $str =~ s/[óòôö]/o/g;
    $str =~ s/[úùûü]/u/g; $str =~ s/ñ/n/g;
    $str =~ s/[^a-z0-9\s-]//g;
    $str =~ s/\s+/-/g;
    $str =~ s/-{2,}/-/g;
    $str =~ s/^-|-$//g;
    return $str;
}

my @titles = (
    "Hello World! This is a Test",
    "  Advanced Perl Programming (2024)  ",
    "CGI & Web Development: Best Practices",
    "100% Pure Perl -- No C!",
);
printf "%-45s  %s\n", "Title", "Slug";
printf "%s\n", "-" x 70;
printf "%-45s  %s\n", $_, slugify($_) for @titles;

# Diff algorithm
printf "\n=== Text Diff (LCS-based) ===\n\n";

{
package Diff;

sub lcs {
    my ($a, $b) = @_;
    my @aa = @$a; my @bb = @$b;
    my @dp = map { [(0) x (@bb+1)] } 0..@aa;
    for my $i (1..@aa) {
        for my $j (1..@bb) {
            $dp[$i][$j] = $aa[$i-1] eq $bb[$j-1] ? $dp[$i-1][$j-1]+1
                        : ($dp[$i-1][$j]>$dp[$i][$j-1] ? $dp[$i-1][$j] : $dp[$i][$j-1]);
        }
    }
    # Backtrack
    my @lcs;
    my ($i,$j) = (@aa, @bb);
    while ($i > 0 && $j > 0) {
        if ($aa[$i-1] eq $bb[$j-1]) { unshift @lcs, $aa[$i-1]; $i--; $j-- }
        elsif ($dp[$i-1][$j] >= $dp[$i][$j-1]) { $i-- }
        else { $j-- }
    }
    return @lcs;
}

sub diff {
    my ($class, $old, $new) = @_;
    my @old_lines = split /\n/, $old;
    my @new_lines = split /\n/, $new;
    my @common = $class->lcs(\@old_lines, \@new_lines);
    
    my @result;
    my ($oi, $ni, $ci) = (0, 0, 0);
    while ($oi < @old_lines || $ni < @new_lines) {
        if ($ci < @common && $oi < @old_lines && $old_lines[$oi] eq $common[$ci]
                          && $ni < @new_lines && $new_lines[$ni] eq $common[$ci]) {
            push @result, { type => " ", line => $common[$ci] };
            $oi++; $ni++; $ci++;
        } elsif ($ni < @new_lines && ($ci >= @common || $new_lines[$ni] ne $common[$ci])) {
            push @result, { type => "+", line => $new_lines[$ni++] };
        } elsif ($oi < @old_lines) {
            push @result, { type => "-", line => $old_lines[$oi++] };
        } else { last }
    }
    return @result;
}
}

my $old_text = "Hello World\nThis is line 2\nLine 3 is here\nGoodbye World";
my $new_text = "Hello Perl\nThis is line 2\nNew line inserted\nLine 3 is here\nGoodbye World";

printf "Diff output:\n";
for my $d (Diff->diff($old_text, $new_text)) {
    printf "%s %s\n", $d->{type}, $d->{line};
}

# Syntax highlight (very basic)
printf "\n=== Syntax Highlighting ===\n\n";

{
package Highlighter;

my %TOKEN_COLORS = (
    keyword  => "\e[1;34m",
    string   => "\e[0;32m",
    number   => "\e[0;33m",
    comment  => "\e[0;37m",
    operator => "\e[0;35m",
    reset    => "\e[0m",
);

my %KEYWORDS = map { $_ => 1 } qw(
    sub my our local use if else elsif while for foreach return print say next last
);

sub highlight {
    my ($class, $code) = @_;
    my @tokens;
    while ($code =~ s{^(
        (#[^\n]*)|                            # comment
        ('(?:[^'\\]|\\.)*'|"(?:[^"\\]|\\.)*")| # string
        (\b(?:\d+(?:\.\d+)?)\b)|              # number
        (\b[a-zA-Z_]\w*\b)|                   # word
        ([+\-*\/=<>!&|^~%,.;:(){}[\]])         # operator/punct
    )}{}x) {
        if    (defined $2) { push @tokens, {type=>"comment",  text=>$2} }
        elsif (defined $3) { push @tokens, {type=>"string",   text=>$3} }
        elsif (defined $4) { push @tokens, {type=>"number",   text=>$4} }
        elsif (defined $5) {
            push @tokens, { type => ($KEYWORDS{$5} ? "keyword" : "word"), text => $5 }
        }
        elsif (defined $6) { push @tokens, {type=>"operator", text=>$6} }
    }
    
    my $out = "";
    for my $tok (@tokens) {
        my $color = $TOKEN_COLORS{$tok->{type}} // "";
        $out .= $color . $tok->{text} . $TOKEN_COLORS{reset};
    }
    return $out;
}
}

my $perl_code = 'sub greet { my ($n) = @_; return "Hello, $n!"; }';
printf "Original: %s\n", $perl_code;
printf "Highlighted: %s\n", Highlighter->highlight($perl_code);
printf "(Colors visible in terminal)\n";

# Capstone: Search engine
printf "\n=== Mini Search Engine ===\n\n";

{
package SearchEngine;

sub new { bless { docs => [], index => {} }, shift }

sub add_document {
    my ($self, $id, $title, $body) = @_;
    push @{$self->{docs}}, { id=>$id, title=>$title, body=>$body };
    my $idx = $#{ $self->{docs} };
    my @words = map { lc } ("$title $body" =~ /\b([a-zA-Z]{3,})\b/g);
    for my $w (@words) {
        $self->{index}{$w}{$idx}++ unless $w =~ /^(?:the|and|for|with|that|this|are|was)$/;
    }
}

sub search {
    my ($self, $query) = @_;
    my @terms = map { lc } ($query =~ /\b(\w{3,})\b/g);
    my %scores;
    for my $term (@terms) {
        if (my $hits = $self->{index}{$term}) {
            $scores{$_} += $hits->{$_} for keys %$hits;
        }
        # Fuzzy: partial match
        for my $word (keys %{$self->{index}}) {
            if ($word =~ /\Q$term\E/i && $word ne $term) {
                $scores{$_} += $self->{index}{$word}{$_} * 0.5
                    for keys %{$self->{index}{$word}};
            }
        }
    }
    return map { { %{$self->{docs}[$_]}, score => $scores{$_} } }
           sort { $scores{$b} <=> $scores{$a} }
           keys %scores;
}
}

my $se = SearchEngine->new;
$se->add_document(1, "Perl Basics", "Introduction to Perl programming language for beginners");
$se->add_document(2, "Advanced Perl", "Advanced Perl programming with Moose and CPAN modules");
$se->add_document(3, "Web Development", "Building web applications with Perl CGI and databases");
$se->add_document(4, "Python Tutorial", "Introduction to Python programming for web development");
$se->add_document(5, "JavaScript Guide", "Modern JavaScript development with Node.js and React");

printf "%-8s  %-6s  %s\n", "Score", "ID", "Title";
printf "%s\n", "-" x 40;
for my $r ($se->search("Perl web programming")) {
    printf "%-8.2f  %-6s  %s\n", $r->{score}, $r->{id}, $r->{title};
}
```

---

## สรุป Part 45 — Advanced Regex & Text Processing

### สิ่งที่เรียนรู้:
- **Advanced Regex** — Named captures, lookahead/behind, atomic groups, qr//
- **Lexer/Tokenizer** — Token types, keywords, operators, string/number literals
- **Expression Parser** — Recursive descent, AST, eval, infix → tree
- **Template Engine** — Variables, blocks, arrays, unless, partials, helpers
- **Text Statistics** — Word freq, n-grams, concordance, Flesch readability
- **Fuzzy Search** — Levenshtein distance, similarity score
- **Slug Generator** — Unicode normalization, URL-safe strings
- **Diff Algorithm** — LCS-based line diff (like `diff` command)
- **Syntax Highlighter** — Token-based ANSI color highlighting
- **Mini Search Engine** — Inverted index, TF scoring, fuzzy expansion

**ถัดไป: [Part 46 — Network Programming](part_46.md)**
