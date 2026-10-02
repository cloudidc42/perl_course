# Part 36: Template Toolkit (TT2)
## Steps 351-360: Template Toolkit — The Professional Templating System

---

## Step 351: Template Toolkit Basics

```perl
#!/usr/bin/perl
# tt_basics.pl — Template Toolkit simulation
use strict;
use warnings;

# Full TT2 engine simulation
{
package Template::Engine;

sub new {
    my ($class, %config) = @_;
    return bless {
        INCLUDE_PATH => $config{INCLUDE_PATH} // ["."],
        INTERPOLATE  => $config{INTERPOLATE}  // 0,
        PRE_CHOMP    => $config{PRE_CHOMP}    // 0,
        POST_CHOMP   => $config{POST_CHOMP}   // 0,
        TRIM         => $config{TRIM}         // 0,
        VARIABLES    => $config{VARIABLES}    // {},
        stash        => {},
    }, $class;
}

sub process {
    my ($self, $input, $vars, $output) = @_;
    
    # Add global vars
    my %all_vars = (%{$self->{VARIABLES}}, %{$vars // {}});
    
    my $template_text;
    if (ref $input eq 'SCALAR') {
        $template_text = $$input;
    } elsif (!ref $input) {
        # File
        for my $dir (@{$self->{INCLUDE_PATH}}) {
            my $path = "$dir/$input";
            if (-f $path) {
                open my $fh, "<", $path or die "Cannot open $path: $!";
                $template_text = do { local $/; <$fh> };
                last;
            }
        }
        die "Template not found: $input" unless defined $template_text;
    }
    
    my $result = $self->_render($template_text, \%all_vars);
    
    if (ref $output eq 'SCALAR') {
        $$output = $result;
    } else {
        print $result;
    }
    return 1;
}

sub _render {
    my ($self, $text, $vars) = @_;
    
    # Process directives
    my $ctx = { vars => $vars, macros => {} };
    return $self->_parse($text, $ctx);
}

sub _parse {
    my ($self, $text, $ctx) = @_;
    my $output = "";
    
    # Split into tokens: [% ... %] and literal text
    my @parts = split /(\[%[-~]?.*?[-~]?%\])/s, $text;
    
    my @block_stack;  # For IF/FOR/WHILE nesting
    my @output_stack = (\$output);
    my $skip = 0;
    
    for my $part (@parts) {
        if ($part =~ /^\[%[-~]?\s*(.*?)\s*[-~]?%\]$/s) {
            my $directive = $1;
            $directive =~ s/^\s+|\s+$//g;
            
            $self->_handle_directive($directive, $ctx, \@block_stack, \@output_stack, \$skip);
        } else {
            # Literal text
            unless ($skip) {
                my $current = $output_stack[-1];
                $$current .= $part;
            }
        }
    }
    
    return $output;
}

sub _handle_directive {
    my ($self, $dir, $ctx, $bstack, $ostack, $skip_ref) = @_;
    
    if ($dir =~ /^SET\s+(\w+)\s*=\s*(.+)$/i) {
        my ($name, $expr) = ($1, $2);
        unless ($$skip_ref) {
            $ctx->{vars}{$name} = $self->_eval_expr($expr, $ctx);
        }
    }
    elsif ($dir =~ /^(\w[\w.]*)\s*=\s*(.+)$/) {
        my ($name, $expr) = ($1, $2);
        unless ($$skip_ref) {
            $ctx->{vars}{$name} = $self->_eval_expr($expr, $ctx);
        }
    }
    elsif ($dir =~ /^IF\s+(.+)$/i) {
        my $cond = $self->_eval_expr($1, $ctx);
        push @$bstack, { type => "IF", cond => !!$cond, done => !!$cond };
        $$skip_ref++ unless $cond;
    }
    elsif ($dir =~ /^ELSIF\s+(.+)$/i || $dir =~ /^ELSE\s*IF\s+(.+)$/i) {
        my $frame = $bstack->[-1];
        if ($frame && $frame->{type} eq "IF") {
            my $cond = $self->_eval_expr($1, $ctx);
            if ($frame->{done}) {
                $$skip_ref = 1;
            } elsif ($cond) {
                $frame->{done} = 1;
                $$skip_ref = 0;
            } else {
                $$skip_ref = 1;
            }
        }
    }
    elsif ($dir =~ /^ELSE$/i) {
        my $frame = $bstack->[-1];
        if ($frame && $frame->{type} eq "IF") {
            $$skip_ref = $frame->{done} ? 1 : 0;
            $frame->{done} = 1;
        }
    }
    elsif ($dir =~ /^END$/i) {
        my $frame = pop @$bstack;
        if ($frame) {
            if ($frame->{type} eq "FOR" || $frame->{type} eq "FOREACH") {
                # Re-render loop body
                unless ($$skip_ref) {
                    my $body = ${$frame->{body_ref}};
                    my $current = $ostack->[-1];
                    my $loop_out = "";
                    my $i = 0;
                    for my $item (@{$frame->{items}}) {
                        $ctx->{vars}{$frame->{var}} = $item;
                        $ctx->{vars}{loop} = { index => $i, count => $i+1,
                            first => $i==0, last => $i==$#{$frame->{items}},
                            size => scalar @{$frame->{items}} };
                        $loop_out .= $self->_parse($body, { %$ctx });
                        $i++;
                    }
                    delete $ctx->{vars}{$frame->{var}};
                    delete $ctx->{vars}{loop};
                    # Replace placeholder
                    $$current =~ s/\x00LOOP$frame->{id}\x00/$loop_out/;
                }
                $$skip_ref-- if $$skip_ref && $$skip_ref > 0;
                pop @$ostack;
            } elsif ($frame->{type} eq "IF") {
                $$skip_ref-- if $$skip_ref && $$skip_ref > 0;
                $$skip_ref = 0 if $$skip_ref < 0;
            } elsif ($frame->{type} eq "BLOCK") {
                my $name = $frame->{name};
                $ctx->{macros}{$name} = ${$frame->{body_ref}};
                pop @$ostack;
                $$skip_ref-- if $$skip_ref;
            }
        }
    }
    elsif ($dir =~ /^(?:FOREACH|FOR)\s+(\w+)\s+IN\s+(.+)$/i) {
        my ($var, $expr) = ($1, $2);
        my $items = $self->_eval_expr($expr, $ctx);
        $items = [] unless ref $items eq 'ARRAY';
        
        my $loop_body = "";
        my $loop_id   = ++$ctx->{loop_counter};
        my $current   = $ostack->[-1];
        $$current .= "\x00LOOP${loop_id}\x00";
        
        push @$bstack, { type => "FOR", var => $var, items => $items,
                         body_ref => \$loop_body, id => $loop_id };
        push @$ostack, \$loop_body;
        $$skip_ref = 0;
    }
    elsif ($dir =~ /^BLOCK\s+(\w+)$/i) {
        my $name     = $1;
        my $blk_body = "";
        push @$bstack, { type => "BLOCK", name => $name, body_ref => \$blk_body };
        push @$ostack, \$blk_body;
        $$skip_ref++;
    }
    elsif ($dir =~ /^INCLUDE\s+(\S+)$/i) {
        # Would include a file
        unless ($$skip_ref) {
            my $file = $1; $file =~ s/^['"]|['"]$//g;
            my $current = $ostack->[-1];
            $$current .= "[INCLUDE $file]";
        }
    }
    elsif ($dir =~ /^INSERT\s+(\S+)$/i) {
        unless ($$skip_ref) {
            my $file = $1; $file =~ s/^['"]|['"]$//g;
            my $current = $ostack->[-1];
            $$current .= "[INSERT $file]";
        }
    }
    else {
        # Expression output
        unless ($$skip_ref) {
            my $val = $self->_eval_expr($dir, $ctx);
            my $current = $ostack->[-1];
            $$current .= defined $val ? $val : "";
        }
    }
}

sub _eval_expr {
    my ($self, $expr, $ctx) = @_;
    $expr =~ s/^\s+|\s+$//g;
    
    # Quoted string
    return $1 if $expr =~ /^"(.*)"$/s;
    return $1 if $expr =~ /^'(.*)'$/s;
    
    # Number
    return $expr+0 if $expr =~ /^-?\d+(\.\d+)?$/;
    
    # Dot notation: var.key.subkey
    if ($expr =~ /^([\w.]+)$/) {
        return $self->_resolve_var($expr, $ctx);
    }
    
    # Method calls: var.method
    if ($expr =~ /^([\w.]+)\.(size|length|keys|values|join|defined|reverse|sort|first|last|max|min|chunk|unique)(?:\((.*?)\))?$/) {
        my ($var, $method, $args) = ($1, $2, $3);
        my $val = $self->_resolve_var($var, $ctx);
        return $self->_call_vmethod($val, $method, $args, $ctx);
    }
    
    # Arithmetic
    if ($expr =~ /^(.+?)\s*([+\-*\/])\s*(.+)$/) {
        my ($a, $op, $b) = ($1, $2, $3);
        my $va = $self->_eval_expr($a, $ctx) // 0;
        my $vb = $self->_eval_expr($b, $ctx) // 0;
        return $op eq "+" ? $va+$vb : $op eq "-" ? $va-$vb : $op eq "*" ? $va*$vb : $vb ? $va/$vb : 0;
    }
    
    # Comparisons
    if ($expr =~ /^(.+?)\s*(==|!=|>=|<=|>|<|eq|ne)\s*(.+)$/) {
        my ($a, $op, $b) = ($1, $2, $3);
        my $va = $self->_eval_expr($a, $ctx) // "";
        my $vb = $self->_eval_expr($b, $ctx) // "";
        my %ops = ("==" => sub{$_[0]==$_[1]}, "!=" => sub{$_[0]!=$_[1]},
                   ">=" => sub{$_[0]>=$_[1]}, "<=" => sub{$_[0]<=$_[1]},
                   ">"  => sub{$_[0]>$_[1]},  "<"  => sub{$_[0]<$_[1]},
                   "eq" => sub{$_[0]eq$_[1]}, "ne" => sub{$_[0]ne$_[1]});
        return $ops{$op}->($va, $vb) ? 1 : 0;
    }
    
    # NOT
    if ($expr =~ /^NOT\s+(.+)$/i) {
        return $self->_eval_expr($1, $ctx) ? 0 : 1;
    }
    
    return undef;
}

sub _resolve_var {
    my ($self, $path, $ctx) = @_;
    my @parts = split /\./, $path;
    my $val   = $ctx->{vars}{shift @parts};
    
    for my $key (@parts) {
        last unless defined $val;
        if (ref $val eq 'HASH')  { $val = $val->{$key} }
        elsif (ref $val eq 'ARRAY' && $key =~ /^\d+$/) { $val = $val->[$key] }
        else { $val = undef }
    }
    return $val;
}

sub _call_vmethod {
    my ($self, $val, $method, $args, $ctx) = @_;
    $args //= "";
    $args = $self->_eval_expr($args, $ctx) if $args;
    
    if (ref $val eq 'ARRAY') {
        return scalar @$val      if $method eq "size" || $method eq "length";
        return $val->[0]          if $method eq "first";
        return $val->[-1]         if $method eq "last";
        return join($args//" ",$val) if $method eq "join";
        return [reverse @$val]   if $method eq "reverse";
        return [sort @$val]      if $method eq "sort";
        my $seen = {}; return [grep { !$seen->{$_}++ } @$val] if $method eq "unique";
    }
    if (ref $val eq 'HASH') {
        return [keys %$val]   if $method eq "keys";
        return [values %$val] if $method eq "values";
        return scalar keys %$val if $method eq "size";
    }
    if (!ref $val) {
        return length($val)  if $method eq "length" || $method eq "size";
        return 1             if $method eq "defined" && defined $val;
    }
    return defined $val ? 1 : 0 if $method eq "defined";
    return undef;
}
}

package main;

printf "=== Template Toolkit (TT2) ===\n\n";

my $tt = Template::Engine->new;

# Test 1: Basic variable interpolation
my $tmpl1 = 'Hello, [% name %]! You have [% count %] messages.';
my $out1;
$tt->process(\$tmpl1, { name => "Alice", count => 5 }, \$out1);
printf "Basic: %s\n\n", $out1;

# Test 2: IF/ELSE
my $tmpl2 = <<'TMPL';
[% IF role == "admin" %]
Admin panel: <a href="/admin">Manage</a>
[% ELSIF role == "mod" %]
Moderator tools available.
[% ELSE %]
Welcome, regular user!
[% END %]
TMPL

for my $role ("admin", "mod", "user") {
    my $out;
    $tt->process(\$tmpl2, { role => $role }, \$out);
    $out =~ s/^\s+|\s+$//g;
    printf "Role=%s: %s\n", $role, $out;
}
printf "\n";

# Test 3: FOREACH
my $tmpl3 = <<'TMPL';
<ul>
[% FOREACH item IN items %]
  <li>[% loop.count %]. [% item.name %] ([% item.price %])</li>
[% END %]
</ul>
TMPL

my $out3;
$tt->process(\$tmpl3, {
    items => [
        { name => "Apple",  price => "\$1.50" },
        { name => "Banana", price => "\$0.75" },
        { name => "Cherry", price => "\$3.00" },
    ]
}, \$out3);
printf "Loop:\n%s\n", $out3;

# Test 4: SET and arithmetic
my $tmpl4 = <<'TMPL';
[% SET total = price * qty %]
Price: [% price %] x [% qty %] = [% total %]
TMPL
my $out4;
$tt->process(\$tmpl4, { price => 9.99, qty => 3 }, \$out4);
$out4 =~ s/\n+/\n/g; $out4 =~ s/^\s+|\s+$//g;
printf "Arithmetic: %s\n\n", $out4;
```

---

## Step 352: TT2 Filters and Directives

```perl
#!/usr/bin/perl
use strict;
use warnings;

# TT2 Filters
{
package TT2::Filters;

my %filters = (
    html        => sub { my $s=shift; $s=~s/&/&amp;/g; $s=~s/</&lt;/g; $s=~s/>/&gt;/g; $s=~s/"/&quot;/g; $s },
    html_entity => sub { my $s=shift; $s=~s/&/&amp;/g; $s=~s/</&lt;/g; $s=~s/>/&gt;/g; $s },
    uri         => sub { my $s=shift; $s=~s/([^A-Za-z0-9\-_\.~])/ sprintf("%%%02X",ord($1)) /ge; $s },
    upper       => sub { uc $_[0] },
    lower       => sub { lc $_[0] },
    ucfirst     => sub { ucfirst lc $_[0] },
    trim        => sub { my $s = $_[0]; $s =~ s/^\s+|\s+$//g; $s },
    collapse    => sub { my $s = $_[0]; $s =~ s/\s+/ /g; $s =~ s/^\s+|\s+$//g; $s },
    null        => sub { "" },
    length      => sub { length $_[0] },
    repeat      => sub { my ($s, $n) = @_; $s x ($n//1) },
    replace     => sub { my ($s, $from, $to) = @_; $s =~ s/\Q$from\E/$to/g; $s },
    remove      => sub { my ($s, $re) = @_; $s =~ s/$re//g; $s },
    truncate    => sub { my ($s,$n,$suf)=@_; $n//=32; $suf//="..."; length($s)<=$n ? $s : substr($s,0,$n-length($suf)).$suf },
    indent      => sub { my ($s,$n)=@_; my $p=" "x($n//4); $p.join("\n$p",split /\n/,$s) },
    nl2br       => sub { my $s = $_[0]; $s =~ s/\n/<br>\n/g; $s },
    strip_tags  => sub { my $s = $_[0]; $s =~ s/<[^>]+>//g; $s },
    js          => sub { my $s = shift; $s =~ s/\\/\\\\/g; $s =~ s/'/\\'/g; $s =~ s/"/\\"/g; $s =~ s/\n/\\n/g; $s },
    json        => sub { require JSON::PP; JSON::PP->new->utf8->encode($_[0]) },
    markdown    => sub {
        my $s = shift;
        $s =~ s/\*\*(.*?)\*\*/<strong>$1<\/strong>/g;
        $s =~ s/\*(.*?)\*/<em>$1<\/em>/g;
        $s =~ s/`(.*?)`/<code>$1<\/code>/g;
        $s =~ s/^# (.+)$/<h1>$1<\/h1>/mg;
        $s =~ s/^## (.+)$/<h2>$1<\/h2>/mg;
        $s
    },
    commify     => sub {
        my $n = sprintf "%.2f", $_[0];
        while ($n =~ s/^(-?\d+)(\d{3})/$1,$2/) {}
        $n
    },
    format_date => sub {
        my ($epoch, $fmt) = @_;
        $fmt //= "%Y-%m-%d";
        my @t = localtime($epoch // time());
        $fmt =~ s/%Y/sprintf("%04d", $t[5]+1900)/e;
        $fmt =~ s/%m/sprintf("%02d", $t[4]+1)/e;
        $fmt =~ s/%d/sprintf("%02d", $t[3])/e;
        $fmt =~ s/%H/sprintf("%02d", $t[2])/e;
        $fmt =~ s/%M/sprintf("%02d", $t[1])/e;
        $fmt =~ s/%S/sprintf("%02d", $t[0])/e;
        $fmt
    },
);

sub register {
    my ($class, $name, $code) = @_;
    $filters{$name} = $code;
}

sub apply {
    my ($class, $name, $value, @args) = @_;
    my $f = $filters{$name} or die "Unknown filter: $name";
    return $f->($value, @args);
}

sub list { sort keys %filters }
}

package main;

printf "=== TT2 Filters ===\n\n";

my @filter_tests = (
    ["html",        "<script>alert('XSS')</script>"],
    ["upper",       "hello world"],
    ["lower",       "HELLO WORLD"],
    ["ucfirst",     "hello world"],
    ["trim",        "   spaces   "],
    ["truncate",    "This is a very long string that needs to be truncated", 30],
    ["uri",         "hello world & foo=bar"],
    ["repeat",      "abc", 3],
    ["replace",     "Hello World", "World", "Perl"],
    ["nl2br",       "line1\nline2\nline3"],
    ["commify",     1234567.89],
    ["strip_tags",  "<p>Hello <b>World</b></p>"],
    ["js",          "He said \"hello\" and it's fine\nnewline"],
    ["format_date", 0],
    ["length",      "Hello, World!"],
);

for my $test (@filter_tests) {
    my ($name, $val, @args) = @$test;
    my $result = TT2::Filters->apply($name, $val, @args);
    $result =~ s/\n/\\n/g;
    printf "  %-15s: %s\n", $name, $result;
}

printf "\nRegistered filters: %s\n", join(", ", TT2::Filters->list);

# Custom filter
TT2::Filters->register("currency", sub {
    my ($amount, $symbol) = @_;
    $symbol //= '$';
    return sprintf "%s%.2f", $symbol, $amount;
});

printf "\nCustom filter: %s\n", TT2::Filters->apply("currency", 42.5, "€");
```

---

## Step 353: TT2 Macros and Wrappers

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Macro system for TT2
{
package TT2::Macros;

my %macros;

sub define {
    my ($class, $name, $params, $body) = @_;
    $macros{$name} = { params => $params, body => $body };
}

sub call {
    my ($class, $name, %args) = @_;
    my $macro = $macros{$name} or die "Macro not defined: $name";
    my $body = $macro->{body};
    
    # Set defaults
    for my $param (@{$macro->{params}}) {
        my ($pname, $default) = @$param;
        $args{$pname} //= $default;
    }
    
    # Simple substitution
    $body =~ s/\[%\s*(\w+)\s*%\]/defined $args{$1} ? $args{$1} : ""/ge;
    return $body;
}

sub list { sort keys %macros }
}

package main;

printf "=== TT2 Macros ===\n\n";

# Define macros
TT2::Macros->define("link", [["url","#"],["text","link"],["class",""]],
    '<a href="[% url %]"[% class %]>[% text %]</a>');

TT2::Macros->define("button", [["label","Submit"],["type","button"],["class","btn"]],
    '<button type="[% type %]" class="[% class %]">[% label %]</button>');

TT2::Macros->define("form_input", [["name",""],["type","text"],["value",""],["placeholder",""]],
    '<input type="[% type %]" name="[% name %]" value="[% value %]" placeholder="[% placeholder %]">');

TT2::Macros->define("alert", [["message",""],["type","info"]],
    '<div class="alert alert-[% type %]">[% message %]</div>');

TT2::Macros->define("badge", [["text",""],["color","secondary"]],
    '<span class="badge bg-[% color %]">[% text %]</span>');

# Use macros
printf "%s\n", TT2::Macros->call("link", url => "/home", text => "Home", class => ' class="nav-link"');
printf "%s\n", TT2::Macros->call("link", url => "/about", text => "About");
printf "%s\n", TT2::Macros->call("button", label => "Save", type => "submit", class => "btn btn-primary");
printf "%s\n", TT2::Macros->call("form_input", name => "email", type => "email", placeholder => "Enter email");
printf "%s\n", TT2::Macros->call("alert", message => "Success!", type => "success");
printf "%s\n", TT2::Macros->call("badge", text => "New", color => "danger");

printf "\nMacros: %s\n", join(", ", TT2::Macros->list);

# Wrapper pattern (layout)
{
my $layout = <<'HTML';
<!DOCTYPE html>
<html>
<head><title>[% title %]</title></head>
<body>
<nav>[% nav %]</nav>
<main>
[% content %]
</main>
<footer>[% footer %]</footer>
</body>
</html>
HTML

sub wrap_content {
    my (%vars) = @_;
    my $out = $layout;
    $out =~ s/\[%\s*(\w+)\s*%\]/$vars{$1}/g;
    return $out;
}

my $page = wrap_content(
    title   => "My Page",
    nav     => '<a href="/">Home</a> | <a href="/about">About</a>',
    content => "<h1>Welcome</h1><p>Perl Template Toolkit rocks!</p>",
    footer  => "&copy; 2024 My Site",
);

printf "\nWrapper output (%.200s...)\n", $page =~ s/\n/ /gr;
}
```

---

## Step 354: TT2 Configuration and Plugins

```perl
#!/usr/bin/perl
use strict;
use warnings;

# TT2 Plugin simulation
{
package TT2::Plugin;

sub new { bless {}, $_[0] }
}

{
package TT2::Plugin::Date;
use parent -norequire, 'TT2::Plugin';

sub new {
    my $self = bless {}, $_[0];
    my @t = localtime;
    $self->{epoch} = time();
    $self->{year}  = $t[5]+1900;
    $self->{month} = $t[4]+1;
    $self->{day}   = $t[3];
    return $self;
}

sub format {
    my ($self, $fmt) = @_;
    $fmt //= "%Y-%m-%d";
    my @t = localtime($self->{epoch});
    $fmt =~ s/%Y/sprintf("%04d",$t[5]+1900)/e;
    $fmt =~ s/%m/sprintf("%02d",$t[4]+1)/e;
    $fmt =~ s/%d/sprintf("%02d",$t[3])/e;
    $fmt =~ s/%H/sprintf("%02d",$t[2])/e;
    $fmt =~ s/%M/sprintf("%02d",$t[1])/e;
    return $fmt;
}

sub now { sprintf "%04d-%02d-%02d", (localtime)[5]+1900, (localtime)[4]+1, (localtime)[3] }
}

{
package TT2::Plugin::Math;
use parent -norequire, 'TT2::Plugin';
use POSIX qw(floor ceil);

sub abs  { abs $_[1] }
sub ceil { ceil $_[1] }
sub floor{ floor $_[1] }
sub max  { my ($self, @n) = @_; my $m = shift @n; $m = $_ > $m ? $_ : $m for @n; $m }
sub min  { my ($self, @n) = @_; my $m = shift @n; $m = $_ < $m ? $_ : $m for @n; $m }
sub round{ my ($self, $n, $d) = @_; $d //= 0; sprintf "%.${d}f", $n }
sub sqrt { sqrt $_[1] }
sub pow  { $_[1] ** $_[2] }
sub pi   { 3.14159265358979 }
sub int  { int $_[1] }
}

{
package TT2::Plugin::String;
use parent -norequire, 'TT2::Plugin';

sub new {
    my ($class, $str) = @_;
    return bless { value => $str // "" }, $class;
}

sub value    { $_[0]->{value} }
sub upper    { uc $_[0]->{value} }
sub lower    { lc $_[0]->{value} }
sub length   { length $_[0]->{value} }
sub trim     { my $s = $_[0]->{value}; $s =~ s/^\s+|\s+$//g; $s }
sub repeat   { $_[0]->{value} x ($_[1]//1) }
sub replace  { my ($self,$from,$to) = @_; (my $s = $self->{value}) =~ s/\Q$from\E/$to/g; $s }
sub split    { [split /\Q$_[1]\E/, $_[0]->{value}] }
sub contains { index($_[0]->{value}, $_[1]) >= 0 ? 1 : 0 }
sub starts_with { substr($_[0]->{value},0,length($_[1])) eq $_[1] ? 1 : 0 }
sub ends_with   { my $l=length($_[1]); substr($_[0]->{value},-$l) eq $_[1] ? 1 : 0 }
sub substr   { substr $_[0]->{value}, $_[1], $_[2] }
}

{
package TT2::Plugin::List;
use parent -norequire, 'TT2::Plugin';
use List::Util qw(sum min max first reduce);

sub new {
    my ($class, @items) = @_;
    @items = @{$items[0]} if @items == 1 && ref $items[0] eq 'ARRAY';
    return bless { items => \@items }, $class;
}

sub items   { @{$_[0]->{items}} }
sub size    { scalar @{$_[0]->{items}} }
sub first   { $_[0]->{items}[0] }
sub last    { $_[0]->{items}[-1] }
sub reverse { [reverse @{$_[0]->{items}}] }
sub sort    { [sort @{$_[0]->{items}}] }
sub unique  { my %s; [grep { !$s{$_}++ } @{$_[0]->{items}}] }
sub join    { join $_[1]//",", @{$_[0]->{items}} }
sub sum     { my $s=0; $s+=$_ for @{$_[0]->{items}}; $s }
sub max     { my $m = $_[0]->{items}[0]; $m = $_ > $m ? $_ : $m for @{$_[0]->{items}}; $m }
sub min     { my $m = $_[0]->{items}[0]; $m = $_ < $m ? $_ : $m for @{$_[0]->{items}}; $m }
sub grep    { my ($self,$code) = @_; [grep { $code->($_) } @{$self->{items}}] }
sub map     { my ($self,$code) = @_; [map  { $code->($_) } @{$self->{items}}] }
sub slice   { my ($self,$s,$e) = @_; [@{$self->{items}}[$s..$e]] }
sub push    { push @{$_[0]->{items}}, $_[1]; $_[0] }
sub pop     { pop  @{$_[0]->{items}} }
sub chunk   {
    my ($self, $size) = @_;
    my @result;
    my @items = @{$self->{items}};
    push @result, [splice @items, 0, $size] while @items;
    return \@result;
}
}

package main;

printf "=== TT2 Plugins ===\n\n";

# Date plugin
my $date = TT2::Plugin::Date->new;
printf "Date plugin:\n";
printf "  now:    %s\n", $date->now;
printf "  format: %s\n", $date->format("%d/%m/%Y");

# Math plugin
my $math = TT2::Plugin::Math->new;
printf "\nMath plugin:\n";
printf "  pi:       %.4f\n",  $math->pi;
printf "  sqrt(16): %s\n",    $math->sqrt(16);
printf "  pow(2,10): %d\n",   $math->pow(2,10);
printf "  round(3.14159, 3): %s\n", $math->round(3.14159, 3);
printf "  max(3,1,4,1,5): %d\n", $math->max(3,1,4,1,5);
printf "  min(3,1,4,1,5): %d\n", $math->min(3,1,4,1,5);

# String plugin
my $str = TT2::Plugin::String->new("  Hello, World!  ");
printf "\nString plugin:\n";
printf "  value:      '%s'\n", $str->value;
printf "  trim:       '%s'\n", $str->trim;
printf "  upper:      '%s'\n", $str->upper;
printf "  length:     %d\n",   $str->length;
printf "  contains:   %d\n",   $str->contains("World");
printf "  starts_with: %d\n",  $str->starts_with("  Hello");

# List plugin
my $list = TT2::Plugin::List->new([5,3,1,4,2,3,1]);
printf "\nList plugin:\n";
printf "  size:   %d\n",  $list->size;
printf "  sum:    %d\n",  $list->sum;
printf "  max:    %d\n",  $list->max;
printf "  min:    %d\n",  $list->min;
printf "  sort:   %s\n",  join(",", @{$list->sort});
printf "  unique: %s\n",  join(",", @{$list->unique});
printf "  chunk:  %s\n",  join(" | ", map { join(",",@$_) } @{$list->chunk(3)});
```

---

## Step 355: TT2 Template Inheritance

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Template inheritance (EXTENDS/BLOCK)
{
package TT2::InheritEngine;

my %templates;

sub register {
    my ($class, $name, $text) = @_;
    $templates{$name} = $text;
}

sub render {
    my ($class, $name, %vars) = @_;
    my $tmpl = $templates{$name} or die "Template not found: $name";
    return $class->_process($tmpl, \%vars, $name);
}

sub _process {
    my ($class, $tmpl, $vars, $current_name) = @_;
    
    # Check for EXTENDS
    if ($tmpl =~ /^\s*\[%\s*EXTENDS\s+'([^']+)'\s*%\]/s ||
        $tmpl =~ /^\s*\[%\s*EXTENDS\s+"([^"]+)"\s*%\]/s) {
        my $parent_name = $1;
        
        # Extract BLOCK definitions from child
        my %child_blocks;
        while ($tmpl =~ /\[%\s*BLOCK\s+(\w+)\s*%\](.*?)\[%\s*END\s*%\]/gs) {
            $child_blocks{$1} = $2;
        }
        
        # Get parent template
        my $parent = $templates{$parent_name} or die "Parent not found: $parent_name";
        
        # Merge: replace parent blocks with child overrides
        my $out = $parent;
        while ($out =~ /\[%\s*BLOCK\s+(\w+)\s*%\](.*?)\[%\s*END\s*%\]/gs) {
            my ($bname, $default) = ($1, $2);
            my $content = $child_blocks{$bname} // $default;
            # Replace the BLOCK..END with content
            $out =~ s/\[%\s*BLOCK\s+\Q$bname\E\s*%\].*?\[%\s*END\s*%\]/$content/s;
        }
        
        # Apply vars
        $out =~ s/\[%\s*([\w.]+)\s*%\]/_resolve($1,$vars)/ge;
        return $out;
    }
    
    # No inheritance — direct render
    my $out = $tmpl;
    $out =~ s/\[%\s*([\w.]+)\s*%\]/_resolve($1,$vars)/ge;
    return $out;
}

sub _resolve {
    my ($path, $vars) = @_;
    my @parts = split /\./, $path;
    my $val   = $vars->{shift @parts};
    $val = ref($val) eq 'HASH' ? $val->{$_} : undef for @parts;
    return defined $val ? $val : "";
}
}

package main;

printf "=== TT2 Template Inheritance ===\n\n";

# Base layout
TT2::InheritEngine->register("base.html", <<'TMPL');
<!DOCTYPE html>
<html>
<head>
  <title>[% BLOCK title %]My Site[% END %]</title>
  <style>[% BLOCK styles %]body{font-family:sans-serif}[% END %]</style>
</head>
<body>
  <header>[% BLOCK header %]<h1>Default Header</h1>[% END %]</header>
  <main>[% BLOCK content %]<p>Default content</p>[% END %]</main>
  <footer>[% BLOCK footer %]&copy; 2024[% END %]</footer>
</body>
</html>
TMPL

# Home page extends base
TT2::InheritEngine->register("home.html", <<'TMPL');
[% EXTENDS 'base.html' %]
[% BLOCK title %]Home — My Site[% END %]
[% BLOCK header %]<h1>Welcome Home!</h1><nav><a href="/">Home</a></nav>[% END %]
[% BLOCK content %]
<h2>Latest News</h2>
<p>Hello, [% user.name %]! Welcome to TT2 inheritance.</p>
[% END %]
TMPL

# About page
TT2::InheritEngine->register("about.html", <<'TMPL');
[% EXTENDS 'base.html' %]
[% BLOCK title %]About Us[% END %]
[% BLOCK content %]
<h2>About</h2>
<p>We love Perl and Template Toolkit!</p>
[% END %]
TMPL

# Render pages
for my $page ("home.html", "about.html") {
    my $html = TT2::InheritEngine->render($page, user => { name => "Alice" });
    printf "=== %s ===\n%s\n", $page, $html;
}
```

---

## Step 356: TT2 Configuration System

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package TT2::Config;

my %defaults = (
    INCLUDE_PATH    => ["."],
    OUTPUT_PATH     => undef,
    ENCODING        => "utf-8",
    START_TAG       => "\\[%",
    END_TAG         => "%\\]",
    TAG_STYLE       => "default",
    PRE_CHOMP       => 0,
    POST_CHOMP      => 0,
    TRIM            => 0,
    INTERPOLATE     => 0,
    ANYCASE         => 0,
    DELIMITER       => ":",
    ABSOLUTE        => 0,
    RELATIVE        => 0,
    DEFAULT         => undef,
    ERROR           => undef,
    AUTO_RESET      => 1,
    RECURSION       => 0,
    VARIABLES       => {},
    CONSTANTS       => {},
    CONSTANT_NAMESPACE => "const",
    NAMESPACE       => undef,
    LOAD_PLUGINS    => [],
    LOAD_FILTERS    => [],
    TOLERANT        => 0,
    SERVICE         => undef,
    CONTEXT         => undef,
    STASH           => undef,
    PARSER          => undef,
    COMPILE_EXT     => undef,
    COMPILE_DIR     => undef,
    CACHE_SIZE      => undef,
    STAT_TTL        => undef,
    EXPOSE_BLOCKS   => 0,
    DEBUG           => 0,
    DEBUG_FORMAT    => "[debug]",
);

sub new {
    my ($class, %args) = @_;
    my %config = %defaults;
    for my $key (keys %args) {
        my $uc = uc $key;
        if (exists $defaults{$uc}) {
            $config{$uc} = $args{$key};
        } else {
            $config{$uc} = $args{$key};
        }
    }
    return bless \%config, $class;
}

sub get { $_[0]->{uc $_[1]} }
sub set { $_[0]->{uc $_[1]} = $_[2] }
sub all { %{$_[0]} }

sub include_path {
    my ($self, @paths) = @_;
    $self->{INCLUDE_PATH} = \@paths if @paths;
    return $self->{INCLUDE_PATH};
}

sub add_include_path {
    my ($self, $path) = @_;
    push @{$self->{INCLUDE_PATH}}, $path;
}

sub is_debug { $_[0]->{DEBUG} }

sub dump {
    my $self = shift;
    my %c = %$self;
    for my $k (sort keys %c) {
        my $v = $c{$k};
        next unless defined $v;
        if (ref $v eq 'ARRAY') { $v = "[" . join(", ", @$v) . "]" }
        elsif (ref $v eq 'HASH') { $v = "{...}" }
        printf "  %-25s = %s\n", $k, $v;
    }
}
}

package main;

printf "=== TT2 Configuration ===\n\n";

my $config = TT2::Config->new(
    INCLUDE_PATH => ["/templates", "/shared/templates"],
    COMPILE_DIR  => "/tmp/tt_cache",
    DEBUG        => 1,
    VARIABLES    => { app_name => "My App", version => "1.0" },
    CONSTANTS    => { MAX_ITEMS => 100, DEFAULT_LANG => "en" },
);

printf "Config values:\n";
printf "  INCLUDE_PATH: %s\n", join(", ", @{$config->get("INCLUDE_PATH")});
printf "  COMPILE_DIR:  %s\n", $config->get("COMPILE_DIR");
printf "  DEBUG:        %d\n", $config->get("DEBUG");

$config->add_include_path("/custom/templates");
printf "\nAfter adding path: %s\n", join(", ", @{$config->include_path});

printf "\nAll non-default settings:\n";
my %all = $config->all;
for my $k (sort keys %all) {
    my $v = $all{$k};
    next unless defined $v;
    next if ref($v) eq 'ARRAY' && !@$v;
    next if ref($v) eq 'HASH'  && !%$v;
    if (ref $v eq 'ARRAY') { printf "  %s: [%s]\n", $k, join(",",@$v) }
    elsif (ref $v eq 'HASH') {
        printf "  %s: {%s}\n", $k, join(", ", map {"$_=$v->{$_}"} keys %$v);
    } else {
        printf "  %s: %s\n", $k, $v;
    }
}
```

---

## Step 357: TT2 Advanced Features

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Advanced TT2 features simulation
{
package TT2::Advanced;

# PERL directive (execute Perl in template)
sub eval_perl {
    my ($code, $vars) = @_;
    local $_ = $vars;
    my $result = eval $code;
    return $@ ? "Error: $@" : ($result // "");
}

# RAWPERL (raw output from Perl)
sub raw_perl {
    my ($code, $vars) = @_;
    my $output = "";
    local *STDOUT;
    open STDOUT, ">", \$output;
    eval $code;
    close STDOUT;
    return $@ ? "Error: $@" : $output;
}

# Virtual Methods
my %vmethods = (
    scalar => {
        length    => sub { length $_[0] },
        upper     => sub { uc $_[0] },
        lower     => sub { lc $_[0] },
        ucfirst   => sub { ucfirst lc $_[0] },
        defined   => sub { defined $_[0] ? 1 : 0 },
        split     => sub { [split /\Q$_[1]\E/, $_[0]] },
        chunk     => sub { my $s=length$_[0]; my $n=$_[1]//1; [map{substr$_[0],$_*$n,$n}0..int($s/$n)-1] },
        repeat    => sub { $_[0] x ($_[1]//1) },
        replace   => sub { (my $s=$_[0])=~s/\Q$_[1]\E/$_[2]//g; $s },
        match     => sub { [$_[0]=~/$_[1]/g] },
        search    => sub { $_[0] =~ /$_[1]/ ? 1 : 0 },
        trim      => sub { (my $s=$_[0])=~s/^\s+|\s+$//g; $s },
        html      => sub { (my $s=$_[0])=~s/&/&amp;/g; $s=~s/</&lt;/g; $s=~s/>/&gt;/g; $s },
        uri       => sub { (my $s=$_[0])=~s/([^A-Za-z0-9\-_.~])/sprintf"%%%02X",ord$1/ge; $s },
        format    => sub { sprintf $_[1], $_[0] },
    },
    list => {
        size     => sub { scalar @{$_[0]} },
        first    => sub { $_[0][0] },
        last     => sub { $_[0][-1] },
        max      => sub { my $m=$_[0][0]; $m=$_>$m?$_:$m for @{$_[0]}; $m },
        min      => sub { my $m=$_[0][0]; $m=$_<$m?$_:$m for @{$_[0]}; $m },
        reverse  => sub { [reverse @{$_[0]}] },
        sort     => sub { [sort @{$_[0]}] },
        unshift  => sub { unshift @{$_[0]}, $_[1]; $_[0] },
        push     => sub { push @{$_[0]}, $_[1]; $_[0] },
        pop      => sub { pop @{$_[0]} },
        shift    => sub { shift @{$_[0]} },
        unique   => sub { my %s; [grep{!$s{$_}++}@{$_[0]}] },
        join     => sub { join $_[1]//"", @{$_[0]} },
        grep     => sub { [grep{/$_[1]/}@{$_[0]}] },
        slice    => sub { [@{$_[0]}[$_[1]..$_[2]]] },
        sum      => sub { my $s=0; $s+=$_ for @{$_[0]}; $s },
        merge    => sub { [@{$_[0]}, @{$_[1]}] },
        chunk    => sub { my @r; my @c=@{$_[0]}; push@r,[splice@c,0,$_[1]]while@c; \@r },
        nsort    => sub { [sort{$a->{$_[1]}//"" cmp $b->{$_[1]}//""}@{$_[0]}] },
        hash     => sub { my %h; $h{$_}=1 for @{$_[0]}; \%h },
    },
    hash => {
        size     => sub { scalar keys %{$_[0]} },
        keys     => sub { [sort keys %{$_[0]}] },
        values   => sub { [map{$_[0]{$_}}sort keys%{$_[0]}] },
        each     => sub { [map{[$_,$_[0]{$_}]}sort keys%{$_[0]}] },
        exists   => sub { exists $_[0]{$_[1]} ? 1 : 0 },
        defined  => sub { defined $_[0]{$_[1]} ? 1 : 0 },
        delete   => sub { delete $_[0]{$_[1]}; $_[0] },
        import   => sub { %{$_[0]}=%{$_[1]}; $_[0] },
        merge    => sub { {%{$_[0]},%{$_[1]}} },
        pairs    => sub { [map{[$_,$_[0]{$_}]}sort keys%{$_[0]}] },
        list     => sub { [map{"$_=$_[0]{$_}"}sort keys%{$_[0]}] },
        json     => sub { require JSON::PP; JSON::PP->new->utf8->encode($_[0]) },
    },
);

sub call_vmethod {
    my ($class, $type, $val, $method, @args) = @_;
    my $f = $vmethods{$type}{$method} or die "No vmethod $type.$method";
    return $f->($val, @args);
}

sub vmethod_list {
    my ($class, $type) = @_;
    return sort keys %{$vmethods{$type}//{}} ;
}
}

package main;

printf "=== TT2 Virtual Methods ===\n\n";

# Scalar vmethods
my $str = "Hello, World!";
printf "Scalar vmethods on '%s':\n", $str;
for my $m ("length","upper","lower","ucfirst","trim","html","uri") {
    my $r = TT2::Advanced->call_vmethod("scalar", $str, $m);
    printf "  .%s: %s\n", $m, $r;
}

printf "\n  .replace('World','Perl'): %s\n", TT2::Advanced->call_vmethod("scalar","Hello, World!","replace","World","Perl");
printf "  .split(','): [%s]\n", join("|", @{TT2::Advanced->call_vmethod("scalar","a,b,c","split",",")});
printf "  .format('%%05d'): %s\n", TT2::Advanced->call_vmethod("scalar","42","format","%05d");

# List vmethods
my $arr = [5,3,1,4,2,3,1];
printf "\nList vmethods on [%s]:\n", join(",",@$arr);
for my $m ("size","first","last","max","min","sum") {
    my $r = TT2::Advanced->call_vmethod("list", $arr, $m);
    printf "  .%s: %s\n", $m, $r;
}
printf "  .sort: [%s]\n", join(",", @{TT2::Advanced->call_vmethod("list",$arr,"sort")});
printf "  .unique: [%s]\n", join(",", @{TT2::Advanced->call_vmethod("list",$arr,"unique")});
printf "  .chunk(3): %s\n", join(" | ", map{"[".join(",",@$_)."]"} @{TT2::Advanced->call_vmethod("list",$arr,"chunk",3)});

# Hash vmethods
my $hash = { a => 1, b => 2, c => 3 };
printf "\nHash vmethods:\n";
printf "  .size: %d\n",   TT2::Advanced->call_vmethod("hash",$hash,"size");
printf "  .keys: [%s]\n", join(",", @{TT2::Advanced->call_vmethod("hash",$hash,"keys")});
printf "  .list: [%s]\n", join(",", @{TT2::Advanced->call_vmethod("hash",$hash,"list")});
```

---

## Step 358: TT2 for Email Templates

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package TT2::EmailRenderer;

my %templates = (
    welcome => {
        subject => 'Welcome to [% site_name %], [% user.name %]!',
        html    => <<'HTML',
<html><body>
<h2>Welcome, [% user.name %]!</h2>
<p>Thank you for joining <strong>[% site_name %]</strong>.</p>
<p>Your account details:</p>
<ul>
  <li>Email: [% user.email %]</li>
  <li>Username: [% user.username %]</li>
</ul>
<p><a href="[% confirm_url %]">Click here to confirm your email</a></p>
<p>This link expires in [% expire_hours %] hours.</p>
</body></html>
HTML
        text    => <<'TEXT',
Welcome, [% user.name %]!

Thank you for joining [% site_name %].

Your account details:
  Email: [% user.email %]
  Username: [% user.username %]

Confirm your email: [% confirm_url %]
This link expires in [% expire_hours %] hours.

-- [% site_name %] Team
TEXT
    },
    password_reset => {
        subject => 'Password Reset — [% site_name %]',
        html    => <<'HTML',
<html><body>
<h2>Password Reset Request</h2>
<p>Hi [% user.name %],</p>
<p>We received a request to reset your password.</p>
<p><a href="[% reset_url %]">Reset your password</a></p>
<p>This link expires at <strong>[% expire_time %]</strong>.</p>
<p>If you didn't request this, please ignore this email.</p>
</body></html>
HTML
        text    => <<'TEXT',
Hi [% user.name %],

We received a request to reset your password for [% site_name %].

Reset your password: [% reset_url %]
This link expires at [% expire_time %].

If you didn't request this, please ignore this email.

-- Security Team, [% site_name %]
TEXT
    },
    order_confirmation => {
        subject => 'Order #[% order.id %] Confirmed — [% site_name %]',
        html    => <<'HTML',
<html><body>
<h2>Order Confirmed!</h2>
<p>Hi [% customer.name %], your order has been placed.</p>
<table border="1" cellpadding="5">
  <tr><th>Item</th><th>Qty</th><th>Price</th></tr>
  [% FOREACH item IN order.items %]
  <tr><td>[% item.name %]</td><td>[% item.qty %]</td><td>$[% item.price %]</td></tr>
  [% END %]
  <tr><td colspan="2"><strong>Total</strong></td><td><strong>$[% order.total %]</strong></td></tr>
</table>
<p>Estimated delivery: [% delivery_date %]</p>
</body></html>
HTML
        text    => <<'TEXT',
Hi [% customer.name %],

Your order #[% order.id %] has been confirmed.

Items:
[% FOREACH item IN order.items %]  - [% item.name %] x[% item.qty %]: $[% item.price %]
[% END %]
Total: $[% order.total %]

Estimated delivery: [% delivery_date %]

-- [% site_name %]
TEXT
    },
);

sub render {
    my ($class, $tmpl_name, %vars) = @_;
    my $tmpl = $templates{$tmpl_name} or die "Unknown template: $tmpl_name";
    
    return {
        subject => $class->_interpolate($tmpl->{subject}, \%vars),
        html    => $class->_interpolate($tmpl->{html},    \%vars),
        text    => $class->_interpolate($tmpl->{text},    \%vars),
    };
}

sub _interpolate {
    my ($class, $text, $vars) = @_;
    my $out = $text;
    
    # Simple FOREACH
    $out =~ s/\[%\s*FOREACH\s+(\w+)\s+IN\s+([\w.]+)\s*%\](.*?)\[%\s*END\s*%\]/
        my ($var,$col,$body) = ($1,$2,$3);
        my $items = $class->_resolve($col,$vars);
        my $r = "";
        for my $item (ref $items eq 'ARRAY' ? @$items : ()) {
            (my $b = $body) =~ s!\[%\s*([\w.]+)\s*%\]!
                defined $item->{$1} ? $item->{$1} : ""
            !ge;
            $r .= $b;
        }
        $r
    /gse;
    
    # Variable interpolation
    $out =~ s/\[%\s*([\w.]+)\s*%\]/$class->_resolve($1,$vars)/ge;
    
    return $out;
}

sub _resolve {
    my ($class, $path, $vars) = @_;
    my @parts = split /\./, $path;
    my $val   = $vars->{shift @parts};
    $val = ref $val eq 'HASH' ? $val->{$_} : undef for @parts;
    return $val // "";
}
}

package main;

printf "=== TT2 Email Templates ===\n\n";

# Welcome email
my $welcome = TT2::EmailRenderer->render("welcome",
    site_name    => "MyApp",
    confirm_url  => "https://myapp.com/confirm/abc123",
    expire_hours => 24,
    user => { name => "Alice Smith", email => 'alice@example.com', username => "alice99" },
);
printf "Subject: %s\n", $welcome->{subject};
printf "Text preview:\n%s\n", substr($welcome->{text}, 0, 200);

# Password reset
my $reset = TT2::EmailRenderer->render("password_reset",
    site_name  => "MyApp",
    reset_url  => "https://myapp.com/reset/xyz789",
    expire_time=> "2024-01-15 18:00 UTC",
    user => { name => "Bob" },
);
printf "\nSubject: %s\n", $reset->{subject};

# Order confirmation
my $order = TT2::EmailRenderer->render("order_confirmation",
    site_name    => "Shop",
    delivery_date=> "Jan 20, 2024",
    customer => { name => "Carol" },
    order => {
        id    => "ORD-2024-001",
        total => "29.97",
        items => [
            { name => "Perl Book",  qty => 1, price => "19.99" },
            { name => "Coffee Mug", qty => 2, price => "4.99" },
        ],
    },
);
printf "\nSubject: %s\n", $order->{subject};
printf "Text preview:\n%s\n", substr($order->{text}, 0, 300);
```

---

## Step 359: TT2 Static Site Generator

```perl
#!/usr/bin/perl
use strict;
use warnings;
use File::Path qw(make_path);

{
package StaticSite;

sub new {
    my ($class, %opts) = @_;
    return bless {
        src_dir  => $opts{src_dir}  // "src",
        out_dir  => $opts{out_dir}  // "output",
        layout   => $opts{layout}   // "layout.html",
        site     => $opts{site}     // {},
        pages    => [],
    }, $class;
}

sub add_page {
    my ($self, %page) = @_;
    push @{$self->{pages}}, {
        title   => $page{title}   // "Untitled",
        slug    => $page{slug}    // lc($page{title} =~ s/\s+/-/gr =~ s/[^a-z0-9-]//gr),
        content => $page{content} // "",
        date    => $page{date}    // "2024-01-01",
        tags    => $page{tags}    // [],
        layout  => $page{layout}  // "page",
    };
}

sub build {
    my $self = shift;
    
    my @built;
    for my $page (@{$self->{pages}}) {
        my $html = $self->_render_page($page);
        my $url  = "/" . $page->{slug} . ".html";
        push @built, { url => $url, title => $page->{title}, html => $html };
    }
    
    # Index
    push @built, {
        url   => "/index.html",
        title => "Home",
        html  => $self->_render_index(\@built),
    };
    
    return @built;
}

my $page_layout = <<'HTML';
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>[% page.title %] | [% site.name %]</title>
  <meta name="description" content="[% site.description %]">
</head>
<body>
<header>
  <h1><a href="/">[% site.name %]</a></h1>
  <nav>[% site.nav %]</nav>
</header>
<main>
  <article>
    <h2>[% page.title %]</h2>
    <time>[% page.date %]</time>
    [% page.content %]
  </article>
</main>
<footer>[% site.footer %]</footer>
</body>
</html>
HTML

my $index_layout = <<'HTML';
<!DOCTYPE html>
<html><head><title>[% site.name %]</title></head>
<body>
<h1>[% site.name %]</h1>
<ul class="pages">
[% pages_list %]
</ul>
</body></html>
HTML

sub _render_page {
    my ($self, $page) = @_;
    my %site = %{$self->{site}};
    my $html = $page_layout;
    
    my %vars = (
        "page.title"   => $page->{title},
        "page.date"    => $page->{date},
        "page.content" => $page->{content},
        "site.name"    => $site{name}        // "My Site",
        "site.description" => $site{description} // "",
        "site.nav"     => $site{nav}         // "",
        "site.footer"  => $site{footer}      // "&copy; 2024",
    );
    
    $html =~ s/\[%\s*([\w.]+)\s*%\]/$vars{$1} \/\/ ""/ge;
    return $html;
}

sub _render_index {
    my ($self, $pages) = @_;
    my $items = "";
    for my $p (@$pages) {
        next if $p->{url} eq "/index.html";
        $items .= sprintf '<li><a href="%s">%s</a></li>', $p->{url}, $p->{title};
    }
    
    my $html = $index_layout;
    my %vars = (
        "site.name"  => $self->{site}{name} // "My Site",
        "pages_list" => $items,
    );
    $html =~ s/\[%\s*([\w.]+)\s*%\]/$vars{$1} \/\/ ""/ge;
    return $html;
}
}

package main;

printf "=== Static Site Generator ===\n\n";

my $site = StaticSite->new(
    site => {
        name        => "Perl Blog",
        description => "A blog about Perl programming",
        nav         => '<a href="/">Home</a> | <a href="/about.html">About</a>',
        footer      => "&copy; 2024 Perl Blog. Built with TT2.",
    }
);

$site->add_page(
    title   => "Getting Started with Perl",
    slug    => "getting-started",
    date    => "2024-01-10",
    content => "<p>Perl is a versatile language. Let's learn it!</p>",
    tags    => ["perl","beginner"],
);

$site->add_page(
    title   => "Template Toolkit Guide",
    slug    => "template-toolkit",
    date    => "2024-01-15",
    content => "<p>TT2 is the gold standard for Perl templating.</p>",
    tags    => ["perl","templates","web"],
);

$site->add_page(
    title   => "About",
    slug    => "about",
    date    => "2024-01-01",
    content => "<p>This blog is about Perl programming.</p>",
    tags    => [],
);

my @pages = $site->build;
printf "Built %d pages:\n", scalar @pages;
for my $p (@pages) {
    printf "  %s — %s (%.1f KB)\n", $p->{url}, $p->{title}, length($p->{html})/1024;
}

printf "\nIndex HTML (truncated):\n%.300s...\n",
    (grep { $_->{url} eq "/index.html" } @pages)[0]{html} =~ s/\s+/ /gr;
```

---

## Step 360: Capstone — TT2-Powered Web Application

```perl
#!/usr/bin/perl
# tt2_webapp.pl — Full app using TT2-style templates
use strict;
use warnings;
use DBI;
use JSON::PP;

# Database setup
my $dbh = DBI->connect("dbi:SQLite::memory:", "", "", {RaiseError=>1, AutoCommit=>1});
for my $sql (
    "CREATE TABLE articles (id INTEGER PRIMARY KEY AUTOINCREMENT, title TEXT NOT NULL, slug TEXT UNIQUE, body TEXT, published INTEGER DEFAULT 0, author TEXT, created_at INTEGER DEFAULT (strftime('%s','now')))",
    "CREATE TABLE comments (id INTEGER PRIMARY KEY AUTOINCREMENT, article_id INTEGER, author TEXT, body TEXT, created_at INTEGER DEFAULT (strftime('%s','now')))",
) { $dbh->do($sql) }

for my $a (
    ["Perl Basics", "perl-basics", "Perl is a high-level programming language...", 1, "Alice"],
    ["Advanced OOP", "advanced-oop", "Object-oriented programming in Perl with Moose...", 1, "Bob"],
    ["Web Development", "web-dev", "Building web apps with CGI and frameworks...", 1, "Alice"],
    ["Draft Post", "draft", "This is a draft...", 0, "Carol"],
) {
    $dbh->do("INSERT INTO articles (title,slug,body,published,author) VALUES (?,?,?,?,?)", undef, @$a);
}

for my $c (
    [1,"Dave","Great introduction!"], [1,"Eve","Very helpful."],
    [2,"Frank","Love the Moose examples!"], [3,"Grace","Awesome web dev guide."],
) {
    $dbh->do("INSERT INTO comments (article_id,author,body) VALUES (?,?,?)", undef, @$c);
}

# Template system
{
package App::Template;

sub render {
    my ($class, $layout, %vars) = @_;
    
    my %layouts = (
        article_list => <<'HTML',
<!DOCTYPE html><html>
<head><title>[% site_name %] — Articles</title></head>
<body>
<h1>[% site_name %]</h1>
[% IF search %]<p>Search: "[% search %]" — [% total %] results</p>[% END %]
<div class="articles">
[% ARTICLES %]
</div>
[% IF has_more %]<a href="?page=[% next_page %]">Next &raquo;</a>[% END %]
</body></html>
HTML
        article_item => <<'HTML',
<article>
  <h2><a href="/[% slug %]">[% title %]</a></h2>
  <p>By [% author %] | [% date %] | [% comments %] comments</p>
  <p>[% excerpt %]</p>
</article>
HTML
        article_detail => <<'HTML',
<!DOCTYPE html><html>
<head><title>[% title %] — [% site_name %]</title></head>
<body>
<article>
  <h1>[% title %]</h1>
  <p>By <strong>[% author %]</strong> | [% date %]</p>
  <div class="body">[% body %]</div>
</article>
<section class="comments">
  <h3>[% comment_count %] Comments</h3>
[% COMMENTS %]
</section>
</body></html>
HTML
        comment_item => <<'HTML',
<div class="comment">
  <strong>[% author %]</strong>: [% body %]
</div>
HTML
    );
    
    my $tmpl = $layouts{$layout} or die "Unknown layout: $layout";
    $tmpl =~ s/\[%\s*([\w.]+)\s*%\]/defined $vars{$1} ? $vars{$1} : ""/ge;
    return $tmpl;
}
}

# Application
{
package App;

sub article_list {
    my ($dbh, %params) = @_;
    my $page   = $params{page}   // 1;
    my $limit  = 10;
    my $offset = ($page - 1) * $limit;
    my $search = $params{search} // "";
    
    my ($where, @bind) = ("WHERE a.published=1", );
    if ($search) { $where .= " AND (a.title LIKE ? OR a.body LIKE ?)"; push @bind, "%$search%","$search%" }
    
    my $total = $dbh->selectrow_array("SELECT COUNT(*) FROM articles a $where", undef, @bind);
    my $articles = $dbh->selectall_arrayref(
        "SELECT a.*, (SELECT COUNT(*) FROM comments c WHERE c.article_id=a.id) as comment_count FROM articles a $where ORDER BY a.created_at DESC LIMIT ? OFFSET ?",
        {Slice=>{}}, @bind, $limit, $offset
    );
    
    my $items = join("", map {
        my $a = $_;
        App::Template->render("article_item",
            slug     => $a->{slug},
            title    => $a->{title},
            author   => $a->{author},
            date     => "2024-01-" . sprintf("%02d", $a->{id}),
            comments => $a->{comment_count},
            excerpt  => substr($a->{body}, 0, 80) . "...",
        )
    } @$articles);
    
    return App::Template->render("article_list",
        site_name => "Perl Blog",
        search    => $search,
        total     => $total,
        ARTICLES  => $items,
        has_more  => ($offset + $limit < $total) ? 1 : 0,
        next_page => $page + 1,
    );
}

sub article_detail {
    my ($dbh, $slug) = @_;
    my $article = $dbh->selectrow_hashref(
        "SELECT * FROM articles WHERE slug=? AND published=1", undef, $slug);
    return "<h1>404 Not Found</h1>" unless $article;
    
    my $comments = $dbh->selectall_arrayref(
        "SELECT * FROM comments WHERE article_id=? ORDER BY created_at", {Slice=>{}}, $article->{id});
    
    my $comment_html = join("", map {
        App::Template->render("comment_item", author => $_->{author}, body => $_->{body})
    } @$comments);
    
    return App::Template->render("article_detail",
        title         => $article->{title},
        author        => $article->{author},
        body          => $article->{body},
        date          => "2024-01-01",
        site_name     => "Perl Blog",
        comment_count => scalar @$comments,
        COMMENTS      => $comment_html,
    );
}
}

# Render
printf "=== TT2 Blog App ===\n\n";

my $list_html = App::article_list($dbh);
printf "Article list HTML: %.200s...\n\n", $list_html =~ s/\s+/ /gr;

my $detail_html = App::article_detail($dbh, "perl-basics");
printf "Article detail HTML: %.200s...\n\n", $detail_html =~ s/\s+/ /gr;

my $search_html = App::article_list($dbh, search => "Moose");
printf "Search results HTML: %.200s...\n", $search_html =~ s/\s+/ /gr;

printf "\nTotal articles: %d\n",
    $dbh->selectrow_array("SELECT COUNT(*) FROM articles WHERE published=1");
printf "Total comments: %d\n",
    $dbh->selectrow_array("SELECT COUNT(*) FROM comments");
```

---

## สรุป Part 36 — Template Toolkit (TT2)

### สิ่งที่เรียนรู้:
- **Template Engine** — Tokenizer, directive parsing, FOREACH/IF/ELSE/END/BLOCK
- **Filters** — html, uri, upper, truncate, markdown, commify, format_date
- **Macros** — MACRO definition and reuse pattern
- **Template Inheritance** — EXTENDS + BLOCK override system
- **Configuration** — All TT2 config options
- **Virtual Methods** — Scalar/List/Hash vmethods
- **Email Templates** — welcome, password_reset, order_confirmation
- **Static Site Generator** — Build pages from templates
- **Capstone** — Full blog app with TT2 rendering

**ถัดไป: [Part 37 — Advanced DBI & ORM Patterns](part_37.md)**
