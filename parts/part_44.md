# Part 44: File Processing & Advanced I/O
## Steps 431-440: CSV, JSON, XML, Binary Files, Streams

---

## Step 431: CSV Processing

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package CSV;

sub new {
    my ($class, %opts) = @_;
    return bless {
        delimiter  => $opts{delimiter} // ",",
        quote      => $opts{quote}     // '"',
        escape     => $opts{escape}    // '"',
        headers    => $opts{headers}   // 1,
        encoding   => $opts{encoding}  // "utf-8",
        _headers   => undef,
    }, $class;
}

sub parse_line {
    my ($self, $line) = @_;
    my @fields;
    my $in_quote = 0;
    my $current  = "";
    my $i = 0;
    my @chars = split //, $line;
    
    while ($i < @chars) {
        my $c = $chars[$i];
        
        if ($in_quote) {
            if ($c eq $self->{quote}) {
                if ($i+1 < @chars && $chars[$i+1] eq $self->{quote}) {
                    # Escaped quote
                    $current .= $c;
                    $i++;
                } else {
                    $in_quote = 0;
                }
            } else {
                $current .= $c;
            }
        } else {
            if ($c eq $self->{quote}) {
                $in_quote = 1;
            } elsif ($c eq $self->{delimiter}) {
                push @fields, $current;
                $current = "";
            } else {
                $current .= $c;
            }
        }
        $i++;
    }
    push @fields, $current;
    return @fields;
}

sub parse {
    my ($self, $text) = @_;
    my @lines = split /\r?\n/, $text;
    my @rows;
    my @headers;
    
    for my $line (@lines) {
        next unless $line =~ /\S/;
        my @fields = $self->parse_line($line);
        
        if ($self->{headers} && !@headers) {
            @headers = @fields;
            $self->{_headers} = \@headers;
            next;
        }
        
        if (@headers) {
            my %row;
            @row{@headers} = @fields;
            push @rows, \%row;
        } else {
            push @rows, \@fields;
        }
    }
    return @rows;
}

sub format_line {
    my ($self, @fields) = @_;
    my @out;
    for my $f (@fields) {
        $f //= "";
        if ($f =~ /[$self->{delimiter}$self->{quote}\r\n]/) {
            $f =~ s/$self->{quote}/$self->{quote}$self->{quote}/g;
            $f = "$self->{quote}$f$self->{quote}";
        }
        push @out, $f;
    }
    return join($self->{delimiter}, @out);
}

sub generate {
    my ($self, @rows) = @_;
    my @lines;
    
    if ($self->{_headers}) {
        push @lines, $self->format_line(@{$self->{_headers}});
    } elsif (@rows && ref $rows[0] eq "HASH") {
        my @keys = sort keys %{$rows[0]};
        $self->{_headers} = \@keys;
        push @lines, $self->format_line(@keys);
    }
    
    for my $row (@rows) {
        if (ref $row eq "HASH") {
            push @lines, $self->format_line(map { $row->{$_} } @{$self->{_headers}});
        } else {
            push @lines, $self->format_line(@$row);
        }
    }
    
    return join("\r\n", @lines) . "\r\n";
}

sub transform {
    my ($self, $input_csv, $transformer) = @_;
    my @rows = $self->parse($input_csv);
    my @transformed = map { $transformer->($_) } @rows;
    @transformed = grep { defined } @transformed;
    return $self->generate(@transformed);
}
}

package main;

printf "=== CSV Processing ===\n\n";

my $csv = CSV->new(headers => 1);

my $data = <<'END';
name,age,city,salary
Alice,30,New York,95000
Bob,25,"San Francisco",88000
Carol,35,"Austin, TX",102000
Dave,28,Chicago,"75,000"
Eve,32,"Seattle",91500
END

printf "Parsing CSV:\n";
my @rows = $csv->parse($data);
for my $r (@rows) {
    printf "  %-8s age=%s city=%-15s salary=%s\n",
        $r->{name}, $r->{age}, $r->{city}, $r->{salary};
}

# Generate
printf "\nGenerating CSV:\n";
$csv->{_headers} = [qw(name age city salary)];
my $new_csv = $csv->generate(@rows);
print $new_csv;

# Transform
printf "\nTransform (add raise):\n";
$csv->{_headers} = undef;
my $transformed = $csv->transform($data, sub {
    my $r = shift;
    (my $sal = $r->{salary}) =~ s/,//g;
    $r->{salary} = $sal * 1.1;
    $r->{salary} = sprintf "%.2f", $r->{salary};
    return $r;
});
print $transformed;

# Statistics
printf "\nStats from CSV:\n";
my @salaries = map { (my $s=$_->{salary})=~s/,//g; $s+0 } @rows;
my $total = 0; $total += $_ for @salaries;
my $avg   = $total / @salaries;
my $max   = (sort { $b<=>$a } @salaries)[0];
my $min   = (sort { $a<=>$b } @salaries)[0];
printf "  count=%d avg=\$%.0f min=\$%.0f max=\$%.0f total=\$%.0f\n",
    scalar @rows, $avg, $min, $max, $total;
```

---

## Step 432: JSON Processing

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

{
package JSON::Utils;

my $json = JSON::PP->new->utf8->pretty->canonical;
my $json_compact = JSON::PP->new->utf8->canonical;

sub pretty  { $json->encode($_[1]) }
sub compact { $json_compact->encode($_[1]) }
sub decode  { $json->decode($_[1]) }

sub merge {
    my ($class, $base, $override) = @_;
    my %result = %{$base // {}};
    for my $k (keys %{$override // {}}) {
        if (ref $result{$k} eq "HASH" && ref $override->{$k} eq "HASH") {
            $result{$k} = $class->merge($result{$k}, $override->{$k});
        } else {
            $result{$k} = $override->{$k};
        }
    }
    return \%result;
}

sub flatten {
    my ($class, $data, $prefix, $result) = @_;
    $prefix //= "";
    $result //= {};
    
    if (ref $data eq "HASH") {
        for my $k (keys %$data) {
            my $new_key = $prefix ? "${prefix}.${k}" : $k;
            $class->flatten($data->{$k}, $new_key, $result);
        }
    } elsif (ref $data eq "ARRAY") {
        for my $i (0..$#$data) {
            $class->flatten($data->[$i], "${prefix}[$i]", $result);
        }
    } else {
        $result->{$prefix} = $data;
    }
    return $result;
}

sub get_path {
    my ($class, $data, $path) = @_;
    my @parts = split /\./, $path;
    my $current = $data;
    for my $part (@parts) {
        return undef unless defined $current;
        if ($part =~ /^(\w+)\[(\d+)\]$/) {
            $current = (ref $current eq "HASH") ? $current->{$1} : undef;
            $current = (ref $current eq "ARRAY") ? $current->[$2] : undef;
        } else {
            $current = (ref $current eq "HASH") ? $current->{$part} : undef;
        }
    }
    return $current;
}

sub set_path {
    my ($class, $data, $path, $value) = @_;
    my @parts = split /\./, $path;
    my $current = $data;
    for my $i (0..$#parts-1) {
        my $part = $parts[$i];
        $current->{$part} //= {};
        $current = $current->{$part};
    }
    $current->{$parts[-1]} = $value;
    return $data;
}

sub transform {
    my ($class, $data, $rules) = @_;
    my $result = {};
    for my $target (keys %$rules) {
        my $source = $rules->{$target};
        if (ref $source eq "CODE") {
            $result->{$target} = $source->($data);
        } else {
            $result->{$target} = $class->get_path($data, $source);
        }
    }
    return $result;
}

sub validate_schema {
    my ($class, $data, $schema) = @_;
    my @errors;
    
    for my $field (keys %$schema) {
        my $rules = $schema->{$field};
        my $val   = $data->{$field};
        
        push @errors, "$field is required" if $rules->{required} && !defined $val;
        next unless defined $val;
        push @errors, "$field must be string" if $rules->{type} eq "string" && ref $val;
        push @errors, "$field must be number" if $rules->{type} eq "number" && $val !~ /^\d+(\.\d+)?$/;
        push @errors, "$field min length" if $rules->{min_length} && length($val) < $rules->{min_length};
    }
    return @errors;
}
}

package main;

printf "=== JSON Processing ===\n\n";

# Complex JSON data
my $user_json = <<'END';
{
    "id": 1,
    "name": "Alice Smith",
    "email": "alice@example.com",
    "address": {
        "street": "123 Elm St",
        "city": "Springfield",
        "country": "US"
    },
    "scores": [95, 87, 92, 88, 94],
    "settings": {
        "theme": "dark",
        "notifications": {
            "email": true,
            "sms": false
        }
    }
}
END

my $user = JSON::Utils->decode($user_json);
printf "Parsed: name=%s city=%s\n\n", $user->{name}, $user->{address}{city};

# Path access
printf "Path access:\n";
printf "  address.city = %s\n", JSON::Utils->get_path($user, "address.city");
printf "  settings.notifications.email = %s\n",
    JSON::Utils->get_path($user, "settings.notifications.email") ? "true" : "false";

# Flatten
printf "\nFlattened:\n";
my $flat = JSON::Utils->flatten($user);
printf "  %s = %s\n", $_, $flat->{$_} for sort keys %$flat;

# Merge
printf "\nMerge:\n";
my $defaults = { theme=>"light", lang=>"en", per_page=>25 };
my $overrides = { theme=>"dark", per_page=>50 };
my $merged = JSON::Utils->merge($defaults, $overrides);
printf "  %s = %s\n", $_, $merged->{$_} for sort keys %$merged;

# Transform
printf "\nTransform (extract fields):\n";
my $transformed = JSON::Utils->transform($user, {
    full_name => "name",
    email     => "email",
    city      => "address.city",
    avg_score => sub { my $d=shift; my @s=@{$d->{scores}}; my $t=0;$t+=$_ for @s; $t/(@s||1) },
});
printf "  %s = %s\n", $_, $transformed->{$_} for sort keys %$transformed;

# Schema validation
printf "\nSchema validation:\n";
my @errors = JSON::Utils->validate_schema($user, {
    id    => { required=>1, type=>"number" },
    name  => { required=>1, type=>"string", min_length=>2 },
    email => { required=>1, type=>"string" },
    phone => { required=>1, type=>"string" },
});
printf "  %s\n", @errors ? join("\n  ", @errors) : "Valid!";
```

---

## Step 433: XML Processing

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package XML::Parser;

sub parse {
    my ($class, $xml) = @_;
    $xml =~ s/<!--.*?-->//gs;  # Remove comments
    $xml =~ s/<\?.*?\?>//gs;   # Remove processing instructions
    return $class->_parse_element(\$xml);
}

sub _parse_element {
    my ($class, $ref) = @_;
    $$ref =~ s/^\s+//;
    
    # Self-closing: <tag attr="val"/>
    if ($$ref =~ s{^<(\w[\w:]*)((?:\s+[\w:]+\s*=\s*"[^"]*")*)\s*/>}{}) {
        my ($tag, $attr_str) = ($1, $2);
        return { tag => $tag, attrs => _parse_attrs($attr_str), children => [], text => "" };
    }
    
    # Opening tag
    return undef unless $$ref =~ s{^<(\w[\w:]*)((?:\s+[\w:]+\s*=\s*"[^"]*")*)\s*>}{};
    my ($tag, $attr_str) = ($1, $2);
    my $node = { tag => $tag, attrs => _parse_attrs($attr_str), children => [], text => "" };
    
    # Content
    while ($$ref !~ m{^</\Q$tag\E>}) {
        $$ref =~ s/^\s+//;
        last unless $$ref;
        
        if ($$ref =~ /^</) {
            my $child = $class->_parse_element($ref);
            push @{$node->{children}}, $child if $child;
        } else {
            $$ref =~ s/^([^<]+)//;
            my $text = $1 // "";
            $text =~ s/^\s+|\s+$//g;
            $node->{text} .= $text if $text;
        }
    }
    $$ref =~ s{^</\Q$tag\E>}{};
    
    return $node;
}

sub _parse_attrs {
    my $str = shift // "";
    my %attrs;
    while ($str =~ /\s*([\w:]+)\s*=\s*"([^"]*)"/g) {
        $attrs{$1} = $2;
    }
    return \%attrs;
}

sub find {
    my ($class, $node, $tag) = @_;
    my @results;
    push @results, $node if $node->{tag} eq $tag;
    for my $child (@{$node->{children} // []}) {
        push @results, $class->find($child, $tag);
    }
    return @results;
}

sub to_hash {
    my ($class, $node) = @_;
    my %h = %{$node->{attrs} // {}};
    $h{_text} = $node->{text} if $node->{text};
    for my $child (@{$node->{children} // []}) {
        my $key = $child->{tag};
        my $val = $class->to_hash($child);
        if (exists $h{$key}) {
            $h{$key} = [$h{$key}] unless ref $h{$key} eq "ARRAY";
            push @{$h{$key}}, $val;
        } else {
            $h{$key} = $val;
        }
    }
    return \%h;
}
}

{
package XML::Builder;

sub new { bless { parts => [], indent => 0, indent_str => "  " }, $_[0] }

sub element {
    my ($self, $tag, %opts) = @_;
    my $attrs = "";
    for my $k (sort keys %{$opts{attrs}//{}}) {
        my $v = $opts{attrs}{$k};
        $v =~ s/&/&amp;/g; $v =~ s/</&lt;/g; $v =~ s/>/&gt;/g; $v =~ s/"/&quot;/g;
        $attrs .= " $k=\"$v\"";
    }
    
    if (defined $opts{text}) {
        my $t = $opts{text};
        $t =~ s/&/&amp;/g; $t =~ s/</&lt;/g; $t =~ s/>/&gt;/g;
        push @{$self->{parts}}, $self->{indent_str} x $self->{indent} . "<$tag$attrs>$t</$tag>";
    } elsif ($opts{children}) {
        push @{$self->{parts}}, $self->{indent_str} x $self->{indent} . "<$tag$attrs>";
        $self->{indent}++;
        $opts{children}->($self);
        $self->{indent}--;
        push @{$self->{parts}}, $self->{indent_str} x $self->{indent} . "</$tag>";
    } else {
        push @{$self->{parts}}, $self->{indent_str} x $self->{indent} . "<$tag$attrs/>";
    }
    return $self;
}

sub raw { push @{$_[0]->{parts}}, $_[1]; $_[0] }
sub build { join "\n", @{$_[0]->{parts}} }
}

package main;

printf "=== XML Processing ===\n\n";

my $xml = <<'END';
<?xml version="1.0" encoding="UTF-8"?>
<catalog>
  <book id="B001" lang="en">
    <title>Programming Perl</title>
    <author>Larry Wall</author>
    <price currency="USD">45.99</price>
    <tags>
      <tag>programming</tag>
      <tag>perl</tag>
    </tags>
  </book>
  <book id="B002" lang="en">
    <title>Learning Perl</title>
    <author>Randal L. Schwartz</author>
    <price currency="USD">39.99</price>
    <tags>
      <tag>beginner</tag>
      <tag>perl</tag>
    </tags>
  </book>
</catalog>
END

my $doc = XML::Parser->parse($xml);
printf "Root tag: %s\n", $doc->{tag};
printf "Children: %d\n\n", scalar @{$doc->{children}};

# Find all books
my @books = XML::Parser->find($doc, "book");
printf "Books found: %d\n", scalar @books;
for my $book (@books) {
    my ($title)  = XML::Parser->find($book, "title");
    my ($author) = XML::Parser->find($book, "author");
    my ($price)  = XML::Parser->find($book, "price");
    printf "  [%s] %s by %s - %s %s\n",
        $book->{attrs}{id}, $title->{text}, $author->{text},
        $price->{attrs}{currency}, $price->{text};
}

# to_hash
printf "\nBook as hash:\n";
my $hash = XML::Parser->to_hash($books[0]);
use Data::Dumper; local $Data::Dumper::Indent=1; local $Data::Dumper::Terse=1;
printf "  id=%s\n  title=%s\n  tags=%s\n",
    $hash->{id}, $hash->{title}{_text}//"",
    join(", ", ref($hash->{tags}{tag}) eq "ARRAY"
        ? map{$_->{_text}} @{$hash->{tags}{tag}}
        : ($hash->{tags}{tag}{_text}//""));

# Build XML
printf "\nBuilding XML:\n";
my $builder = XML::Builder->new;
$builder->raw('<?xml version="1.0" encoding="UTF-8"?>');
$builder->element("response", attrs=>{status=>"ok",version=>"1.0"}, children=>sub {
    my $b = shift;
    $b->element("user", attrs=>{id=>"1"}, children=>sub {
        my $b=shift;
        $b->element("name",  text=>"Alice <Smith>");
        $b->element("email", text=>"alice\@test.com");
        $b->element("active", text=>"true");
    });
    $b->element("meta", attrs=>{generated=>"2024-01-08"});
});

print $builder->build . "\n";
```

---

## Step 434: Binary File Processing

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package BinaryFile;

sub new {
    my ($class, %opts) = @_;
    return bless {
        data        => $opts{data} // "",
        byte_order  => $opts{byte_order} // "big",  # big or little endian
        _pos        => 0,
    }, $class;
}

sub from_file {
    my ($class, $path) = @_;
    open my $fh, "<:raw", $path or die "Cannot open $path: $!\n";
    local $/; my $data = <$fh>;
    close $fh;
    return $class->new(data => $data);
}

sub write_file {
    my ($self, $path) = @_;
    open my $fh, ">:raw", $path or die "Cannot write $path: $!\n";
    print $fh $self->{data};
    close $fh;
}

# Read methods
sub read_bytes  { my($s,$n)=@_; my $d=substr($s->{data},$s->{_pos},$n); $s->{_pos}+=$n; $d }
sub read_uint8  { unpack "C",  $_[0]->read_bytes(1) }
sub read_uint16 { unpack($_[0]->{byte_order} eq "big" ? "n" : "v", $_[0]->read_bytes(2)) }
sub read_uint32 { unpack($_[0]->{byte_order} eq "big" ? "N" : "V", $_[0]->read_bytes(4)) }
sub read_int32  { unpack($_[0]->{byte_order} eq "big" ? "l>" : "l<", $_[0]->read_bytes(4)) }
sub read_float  { unpack("f", $_[0]->read_bytes(4)) }
sub read_double { unpack("d", $_[0]->read_bytes(8)) }
sub read_string { my($s,$n)=@_; my $str=$s->read_bytes($n); $str=~s/\0//g; $str }
sub read_cstring {
    my $self = shift;
    my $str = "";
    while (my $c = $self->read_bytes(1)) {
        last if ord($c) == 0;
        $str .= $c;
    }
    return $str;
}

# Write methods
sub write_bytes  { $_[0]->{data} .= $_[1] }
sub write_uint8  { $_[0]->write_bytes(pack "C",  $_[1]) }
sub write_uint16 { $_[0]->write_bytes(pack($_[0]->{byte_order} eq "big"?"n":"v", $_[1])) }
sub write_uint32 { $_[0]->write_bytes(pack($_[0]->{byte_order} eq "big"?"N":"V", $_[1])) }
sub write_int32  { $_[0]->write_bytes(pack($_[0]->{byte_order} eq "big"?"l>":"l<", $_[1])) }
sub write_string { my($s,$str,$len)=@_; $s->write_bytes(pack("A$len",$str)) }
sub write_cstring{ $_[0]->write_bytes($_[1] . "\0") }

sub seek_to { $_[0]->{_pos} = $_[1] }
sub pos     { $_[0]->{_pos} }
sub size    { length $_[0]->{data} }
sub at_end  { $_[0]->{_pos} >= $_[0]->size }

sub hexdump {
    my ($self, $offset, $length) = @_;
    $offset //= 0; $length //= 64;
    my $out = "";
    for (my $i = $offset; $i < $offset+$length && $i < $self->size; $i+=16) {
        $out .= sprintf "%04x  ", $i;
        my @bytes = map { ord(substr($self->{data},$i+$_,1)) } 0..15;
        @bytes = @bytes[0..[$i+15,$self->size-1]->[0]-$i];
        $out .= sprintf "%02x ", $_ for @bytes;
        $out .= "   " x (16-scalar @bytes);
        $out .= " |";
        $out .= sprintf "%s", map { ($_ >= 32 && $_ < 127) ? chr($_) : "." } @bytes;
        $out .= "|\n";
    }
    return $out;
}
}

package main;

printf "=== Binary File Processing ===\n\n";

# Create a binary file
my $bf = BinaryFile->new(byte_order => "big");

# Write a custom binary format:
# Magic: 4 bytes "PBIN"
# Version: uint16
# Record count: uint32
# Records: id (uint32) + name (16-byte string) + score (float)

$bf->write_bytes("PBIN");
$bf->write_uint16(1);    # version
$bf->write_uint32(3);    # 3 records

my @records = (
    { id => 1, name => "Alice",   score => 95.5 },
    { id => 2, name => "Bob",     score => 88.0 },
    { id => 3, name => "Carol",   score => 92.3 },
);

for my $rec (@records) {
    $bf->write_uint32($rec->{id});
    $bf->write_string($rec->{name}, 16);
    $bf->write_bytes(pack("f", $rec->{score}));
}

printf "Written %d bytes\n\n", $bf->size;

# Hex dump
printf "Hexdump:\n%s\n", $bf->hexdump(0, 48);

# Read back
$bf->seek_to(0);
my $magic = $bf->read_bytes(4);
my $ver   = $bf->read_uint16;
my $count = $bf->read_uint32;

printf "Magic: %s, Version: %d, Records: %d\n\n", $magic, $ver, $count;

printf "Records:\n";
for (1..$count) {
    my $id    = $bf->read_uint32;
    my $name  = $bf->read_string(16);
    my $score = unpack("f", $bf->read_bytes(4));
    printf "  id=%d name=%-10s score=%.1f\n", $id, $name, $score;
}

# BMP header example
printf "\n--- BMP Header Parser ---\n";
my $bmp = BinaryFile->new(byte_order => "little");
$bmp->write_bytes("BM");         # Signature
$bmp->write_uint32(54+100*100*3);# File size
$bmp->write_uint32(0);           # Reserved
$bmp->write_uint32(54);          # Data offset
$bmp->write_uint32(40);          # Header size
$bmp->write_int32(100);          # Width
$bmp->write_int32(100);          # Height
$bmp->write_uint16(1);           # Color planes
$bmp->write_uint16(24);          # Bits per pixel

$bmp->seek_to(0);
my $sig    = $bmp->read_bytes(2);
my $fsz    = $bmp->read_uint32;
$bmp->read_uint32;               # Reserved
my $offset = $bmp->read_uint32;
$bmp->read_uint32;               # Header size
my $width  = $bmp->read_int32;
my $height = $bmp->read_int32;
$bmp->read_uint16;
my $bpp    = $bmp->read_uint16;
printf "BMP: sig=%s size=%d offset=%d %dx%d %dbpp\n", $sig, $fsz, $offset, $width, $height, $bpp;
```

---

## Step 435: Stream Processing

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Stream;

sub new {
    my ($class, @source) = @_;
    my $items = \@source;
    return bless { source => sub { shift @$items }, done => 0 }, $class;
}

sub from_file {
    my ($class, $fh) = @_;
    return bless {
        source => sub { my $line = <$fh>; defined $line ? do { chomp $line; $line } : undef },
        done   => 0,
    }, $class;
}

sub from_array {
    my ($class, $arr) = @_;
    my $i = 0;
    return bless { source => sub { $i < @$arr ? $arr->[$i++] : undef }, done => 0 }, $class;
}

sub next {
    my $self = shift;
    return undef if $self->{done};
    my $val = $self->{source}->();
    $self->{done} = 1 unless defined $val;
    return $val;
}

sub map {
    my ($self, $fn) = @_;
    return bless {
        source => sub {
            while (1) {
                my $v = $self->next;
                return undef unless defined $v;
                my $result = eval { $fn->($v) };
                return $result unless $@;
            }
        },
        done => 0,
    }, ref $self;
}

sub filter {
    my ($self, $pred) = @_;
    return bless {
        source => sub {
            while (1) {
                my $v = $self->next;
                return undef unless defined $v;
                return $v if $pred->($v);
            }
        },
        done => 0,
    }, ref $self;
}

sub take {
    my ($self, $n) = @_;
    my $count = 0;
    return bless {
        source => sub {
            return undef if $count++ >= $n;
            $self->next;
        },
        done => 0,
    }, ref $self;
}

sub skip {
    my ($self, $n) = @_;
    my $skipped = 0;
    return bless {
        source => sub {
            while ($skipped < $n) {
                $self->next; $skipped++;
            }
            $self->next;
        },
        done => 0,
    }, ref $self;
}

sub reduce {
    my ($self, $fn, $init) = @_;
    my $acc = $init;
    while (defined(my $v = $self->next)) {
        $acc = $fn->($acc, $v);
    }
    return $acc;
}

sub each {
    my ($self, $fn) = @_;
    while (defined(my $v = $self->next)) {
        $fn->($v);
    }
}

sub to_array {
    my $self = shift;
    my @result;
    $self->each(sub { push @result, $_[0] });
    return @result;
}

sub chunk {
    my ($self, $size) = @_;
    return bless {
        source => sub {
            my @chunk;
            while (@chunk < $size) {
                my $v = $self->next;
                return @chunk ? \@chunk : undef unless defined $v;
                push @chunk, $v;
            }
            return \@chunk;
        },
        done => 0,
    }, ref $self;
}

sub count { my $s=shift; my $n=0; $n++ while defined $s->next; $n }
}

package main;

printf "=== Stream Processing ===\n\n";

# Number stream
printf "1-10 filter even, map x2, take 3:\n";
my @result = Stream->new(1..10)
    ->filter(sub { $_[0] % 2 == 0 })
    ->map(sub { $_[0] * 2 })
    ->take(3)
    ->to_array;
printf "  [%s]\n\n", join(", ", @result);

# Sum
my $sum = Stream->new(1..100)->reduce(sub { $_[0]+$_[1] }, 0);
printf "sum(1..100) = %d\n\n", $sum;

# Chunk processing
printf "Chunks of 3 from 1..9:\n";
Stream->new(1..9)->chunk(3)->each(sub {
    printf "  [%s]\n", join(",", @{$_[0]});
});

# String processing pipeline
printf "\nString pipeline:\n";
my @words = qw(apple banana cherry date elderberry fig grape);
my @long_sorted = Stream->from_array(\@words)
    ->filter(sub { length($_[0]) > 5 })
    ->map(sub { uc $_[0] })
    ->to_array;
printf "  Words > 5 chars: %s\n\n", join(", ", sort @long_sorted);

# Log line processing simulation
printf "Log processing simulation:\n";
my @log_lines = (
    "[ERROR] 2024-01-08 10:01:02 - Connection timeout",
    "[INFO]  2024-01-08 10:01:03 - Request processed",
    "[ERROR] 2024-01-08 10:01:05 - DB query failed",
    "[WARN]  2024-01-08 10:01:06 - High memory usage",
    "[ERROR] 2024-01-08 10:01:10 - Null pointer exception",
    "[INFO]  2024-01-08 10:01:12 - Health check OK",
);

my @errors = Stream->from_array(\@log_lines)
    ->filter(sub { $_[0] =~ /\[ERROR\]/ })
    ->map(sub {
        my $line = shift;
        $line =~ /\[(\w+)\]\s+(\S+ \S+) - (.+)/;
        { level => $1, ts => $2, msg => $3 }
    })
    ->to_array;

printf "  Found %d errors:\n", scalar @errors;
printf "  %s: %s\n", $_->{ts}, $_->{msg} for @errors;
```

---

## Step 436: File Watcher

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(sha256_hex);

{
package FileWatcher;

sub new {
    my ($class, %opts) = @_;
    return bless {
        paths     => {},
        handlers  => {},
        poll_interval => $opts{poll_interval} // 1,
        recursive => $opts{recursive} // 0,
    }, $class;
}

sub watch {
    my ($self, $path, $events, $handler) = @_;
    $self->{paths}{$path} = {
        path     => $path,
        events   => ref($events) ? $events : [$events],
        handler  => $handler,
        stat     => $self->_stat($path),
        hash     => $self->_hash($path),
    };
    return $self;
}

sub unwatch { delete $_[0]->{paths}{$_[1]} }

sub check {
    my $self = shift;
    my @events;
    
    for my $path (keys %{$self->{paths}}) {
        my $entry   = $self->{paths}{$path};
        my $new_stat= $self->_stat($path);
        my $new_hash= $self->_hash($path);
        
        if (!$new_stat && $entry->{stat}) {
            # File deleted
            push @events, { type=>"deleted", path=>$path };
            $entry->{stat} = undef;
            $entry->{hash} = undef;
        } elsif ($new_stat && !$entry->{stat}) {
            # File created
            push @events, { type=>"created", path=>$path };
            $entry->{stat} = $new_stat;
            $entry->{hash} = $new_hash;
        } elsif ($new_stat && $entry->{stat}) {
            # Check modified
            if ($new_hash ne ($entry->{hash}//"")) {
                push @events, { type=>"modified", path=>$path,
                    old_size => $entry->{stat}{size}, new_size => $new_stat->{size} };
                $entry->{stat} = $new_stat;
                $entry->{hash} = $new_hash;
            }
        }
    }
    
    # Fire handlers
    for my $event (@events) {
        my $entry = $self->{paths}{$event->{path}} or next;
        if (grep { $_ eq $event->{type} || $_ eq "all" } @{$entry->{events}}) {
            $entry->{handler}->($event);
        }
    }
    
    return @events;
}

sub _stat {
    my ($self, $path) = @_;
    return undef unless -e $path;
    my @s = stat($path);
    return { size => $s[7], mtime => $s[9] };
}

sub _hash {
    my ($self, $path) = @_;
    return "" unless -f $path;
    open my $fh, "<", $path or return "";
    local $/; my $content = <$fh>;
    close $fh;
    return sha256_hex($content);
}

sub run_once { $_[0]->check }
}

package main;

use File::Temp qw(tempdir tempfile);
use File::Basename;

printf "=== File Watcher ===\n\n";

my $tmpdir = tempdir(CLEANUP => 1);
my @watch_events;

my $watcher = FileWatcher->new;

# Watch files
my $config_file = "$tmpdir/config.json";
my $data_file   = "$tmpdir/data.txt";

# Create files
open my $fh1, ">", $config_file; print $fh1 '{"key":"initial"}'; close $fh1;
open my $fh2, ">", $data_file;   print $fh2 "line1\nline2"; close $fh2;

$watcher->watch($config_file, ["modified","deleted"], sub {
    push @watch_events, "CONFIG: $_[0]{type} (size: $_[0]{new_size})";
});
$watcher->watch($data_file, "all", sub {
    push @watch_events, "DATA: $_[0]{type}";
});

# Initial check (no changes)
my @e = $watcher->check;
printf "Initial check: %d events\n", scalar @e;

# Modify config
open $fh1, ">", $config_file;
print $fh1 '{"key":"updated","new_field":true}';
close $fh1;

@e = $watcher->check;
printf "After config modify: %d events\n", scalar @e;

# Delete data file
unlink $data_file;
@e = $watcher->check;
printf "After delete: %d events\n", scalar @e;

# Re-create data file
open $fh2, ">", $data_file; print $fh2 "new data"; close $fh2;
@e = $watcher->check;
printf "After re-create: %d events\n\n", scalar @e;

printf "All events:\n";
printf "  %s\n", $_ for @watch_events;
```

---

## Step 437: Archive & Compression

```perl
#!/usr/bin/perl
use strict;
use warnings;
use MIME::Base64 qw(encode_base64 decode_base64);
use Compress::Zlib;

{
package Archive;

# Simple TAR-like format
sub create {
    my ($class, @files) = @_;
    my $archive = "";
    
    for my $file (@files) {
        my $name    = $file->{name};
        my $content = $file->{content} // "";
        my $size    = length($content);
        
        # Header: name (100), size (12 octal), checksum placeholder
        my $header = pack("A100 A12 A8 A8 A8 A12 A12 A8 a1 a100 A6 A2",
            $name, sprintf("%011o", $size), "0000755", "0000000", "0000000",
            sprintf("%011o", time()), "        ", " " x 8, "0", "", "ustar ", "00");
        
        # Pad to 512 bytes
        $header .= "\0" x (512 - length($header));
        
        # Content padded to 512 boundary
        my $padded = $content . "\0" x ((512 - $size % 512) % 512);
        
        $archive .= $header . $padded;
    }
    
    # End-of-archive: two 512-byte blocks of zeros
    $archive .= "\0" x 1024;
    return $archive;
}

sub list {
    my ($class, $archive) = @_;
    my @files;
    my $pos = 0;
    
    while ($pos + 512 <= length($archive)) {
        my $header = substr($archive, $pos, 512);
        last if $header eq "\0" x 512;
        
        my $name = unpack("A100", $header);
        my $size = oct(unpack("x100 A12", $header));
        
        last unless $name;
        push @files, { name => $name, size => $size, pos => $pos + 512 };
        $pos += 512 + int(($size + 511) / 512) * 512;
    }
    return @files;
}

sub extract {
    my ($class, $archive, $target) = @_;
    my @files = $class->list($archive);
    for my $f (@files) {
        return substr($archive, $f->{pos}, $f->{size}) if $f->{name} eq $target;
    }
    return undef;
}
}

{
package Compress;

sub gzip {
    my ($class, $data) = @_;
    my $gz = Compress::Zlib::deflateInit(
        -Level => Z_BEST_COMPRESSION,
        -WindowBits => MAX_WBITS + 16,  # gzip mode
    ) or return $data;  # fallback
    my ($output, $status) = $gz->deflate($data);
    my ($flush, $s2) = $gz->flush;
    return $output . $flush;
}

sub gunzip {
    my ($class, $data) = @_;
    my $gz = Compress::Zlib::inflateInit(-WindowBits => MAX_WBITS + 16) or return $data;
    my ($output, $status) = $gz->inflate($data);
    return $output;
}

sub compress_ratio {
    my ($class, $original, $compressed) = @_;
    return length($original) ? (1 - length($compressed)/length($original)) * 100 : 0;
}
}

package main;

printf "=== Archive & Compression ===\n\n";

# Create archive
printf "Creating archive:\n";
my @files = (
    { name => "README.txt",    content => "This is the README file.\n" . "A" x 100 },
    { name => "config.json",   content => '{"host":"localhost","port":8080,"debug":true}' },
    { name => "src/main.pl",   content => "#!/usr/bin/perl\nuse strict;\nuse warnings;\nprint 'Hello!';\n" },
    { name => "data/users.csv",content => "id,name,email\n1,Alice,alice\@test.com\n2,Bob,bob\@test.com\n" },
);

my $archive = Archive->create(@files);
printf "  Archive size: %d bytes\n", length($archive);

# List
printf "\nArchive contents:\n";
my @listed = Archive->list($archive);
printf "  %-30s %d bytes\n", $_->{name}, $_->{size} for @listed;

# Extract
printf "\nExtracting config.json:\n";
my $extracted = Archive->extract($archive, "config.json");
printf "  Content: %s\n\n", $extracted;

# Compression
printf "Compression:\n";
my $sample_text = "Hello World! " x 1000;  # Highly compressible
printf "  Original size: %d bytes\n", length($sample_text);

my $compressed = Compress->gzip($sample_text);
printf "  Compressed:    %d bytes\n", length($compressed);
printf "  Ratio:         %.1f%%\n\n", Compress->compress_ratio($sample_text, $compressed);

my $decompressed = Compress->gunzip($compressed);
printf "  Decompressed matches: %s\n",
    $decompressed eq $sample_text ? "YES" : "NO";
```

---

## Step 438-440: File Locking, Large File Processing, Capstone

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Fcntl qw(:flock SEEK_SET);

# File locking
printf "=== File Locking ===\n\n";

{
package FileLock;

sub new {
    my ($class, $path) = @_;
    return bless { path => $path, fh => undef }, $class;
}

sub lock_exclusive {
    my $self = shift;
    open $self->{fh}, "+>>", $self->{path} or die "Cannot open $self->{path}: $!\n";
    flock($self->{fh}, LOCK_EX) or die "Cannot lock: $!\n";
    seek $self->{fh}, 0, SEEK_SET;
    return $self;
}

sub lock_shared {
    my $self = shift;
    open $self->{fh}, "<", $self->{path} or die "Cannot open $self->{path}: $!\n";
    flock($self->{fh}, LOCK_SH) or die "Cannot lock: $!\n";
    return $self;
}

sub try_lock {
    my $self = shift;
    open $self->{fh}, "+>>", $self->{path} or return 0;
    return flock($self->{fh}, LOCK_EX | LOCK_NB) ? 1 : 0;
}

sub unlock {
    my $self = shift;
    flock($self->{fh}, LOCK_UN) if $self->{fh};
    close $self->{fh} if $self->{fh};
    $self->{fh} = undef;
}

sub read_all {
    my $self = shift;
    local $/; readline($self->{fh});
}

sub write_all {
    my ($self, $data) = @_;
    truncate $self->{fh}, 0;
    seek $self->{fh}, 0, SEEK_SET;
    print { $self->{fh} } $data;
}

sub DESTROY { $_[0]->unlock }
}

use File::Temp qw(tempfile);
my ($tmp_fh, $tmp_path) = tempfile(UNLINK => 1);
print $tmp_fh "initial content\n";
close $tmp_fh;

my $lock = FileLock->new($tmp_path);
$lock->lock_exclusive;
$lock->write_all("updated content by exclusive lock\n");
my $content = $lock->read_all;
$lock->unlock;

printf "File after exclusive write: %s", $content;

my $try = FileLock->new($tmp_path);
printf "try_lock: %s\n\n", $try->try_lock ? "acquired" : "already locked";

# Large file streaming
printf "=== Large File Processing ===\n\n";

# Generate a "large" file in memory
my $large_file_data = join("\n", map { "line$_," . "x" x 80 } 1..10000);

printf "Processing %d bytes in chunks:\n", length($large_file_data);

my $line_count = 0;
my $match_count = 0;
my @sample;

open my $fh, "<", \$large_file_data;
while (<$fh>) {
    $line_count++;
    if (/line(5\d\d\d),/) {
        $match_count++;
        push @sample, "line$1" if @sample < 3;
    }
}
close $fh;

printf "  Total lines: %d\n", $line_count;
printf "  Lines 5000-5999: %d\n", $match_count;
printf "  Sample: %s\n\n", join(", ", @sample);

# Capstone: File processing pipeline
printf "=== File Processing Pipeline ===\n\n";

my $csv_data = <<'END';
id,name,department,salary,start_date
1,Alice Smith,Engineering,95000,2020-03-15
2,Bob Jones,Marketing,72000,2019-07-01
3,Carol White,Engineering,105000,2018-11-20
4,Dave Brown,Sales,68000,2021-01-10
5,Eve Davis,Engineering,98000,2020-09-05
6,Frank Miller,Marketing,75000,2019-03-22
7,Grace Wilson,Sales,71000,2022-05-01
8,Henry Taylor,Engineering,88000,2021-08-15
END

# Parse, transform, aggregate
open my $csv_fh, "<", \$csv_data;
my @headers;
my @emp_data;

while (my $line = <$csv_fh>) {
    chomp $line;
    my @fields = split /,/, $line;
    if (!@headers) { @headers = @fields; next; }
    my %row; @row{@headers} = @fields;
    push @emp_data, \%row;
}
close $csv_fh;

# Aggregate by department
my %dept_stats;
for my $emp (@emp_data) {
    my $dept = $emp->{department};
    $dept_stats{$dept}{count}++;
    $dept_stats{$dept}{total_salary} += $emp->{salary};
    push @{$dept_stats{$dept}{employees}}, $emp->{name};
}

printf "Department Statistics:\n";
printf "%-15s  count  avg_salary  employees\n", "Department";
printf "%s\n", "-" x 70;
for my $dept (sort keys %dept_stats) {
    my $s = $dept_stats{$dept};
    printf "%-15s  %5d  %10.0f  %s\n",
        $dept, $s->{count}, $s->{total_salary}/$s->{count},
        join(", ", sort @{$s->{employees}});
}

# Top earners per dept
printf "\nTop earner per department:\n";
for my $dept (sort keys %dept_stats) {
    my ($top) = sort { $b->{salary} <=> $a->{salary} }
                grep { $_->{department} eq $dept } @emp_data;
    printf "  %-15s => %s (\$%s)\n", $dept, $top->{name}, $top->{salary};
}
```

---

## สรุป Part 44 — File Processing & Advanced I/O

### สิ่งที่เรียนรู้:
- **CSV** — RFC 4180 parser, quoted fields, generation, transformation
- **JSON** — Path access, merge, flatten, schema validation
- **XML** — SAX-style parser, builder, attribute/child handling
- **Binary Files** — Struct pack/unpack, hexdump, custom formats
- **Stream Processing** — Lazy map/filter/reduce/chunk pipeline
- **File Watcher** — Poll-based change detection with SHA hashes
- **Archive** — TAR-style header+content format, list/extract
- **Compression** — Zlib gzip/gunzip with ratio tracking
- **File Locking** — flock LOCK_EX/SH/NB, exclusive write, try_lock
- **Capstone** — Multi-stage CSV → aggregate pipeline

**ถัดไป: [Part 45 — Advanced CPAN Module Development](part_45.md)**
