# Part 13: File I/O ขั้นสูง
## Steps 121-130: การจัดการไฟล์อย่างมืออาชีพ

---

## Step 121: File Modes และ Filehandles

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Open modes
# =====================

# < — read only (default)
open(my $fh, '<', 'input.txt') or die "Cannot open: $!";
close $fh;

# > — write (truncate)
open($fh, '>', 'output.txt') or die "Cannot open: $!";
print $fh "Hello\n";
close $fh;

# >> — append
open($fh, '>>', 'log.txt') or die "Cannot open: $!";
print $fh "New entry\n";
close $fh;

# +< — read/write (file must exist)
# +> — read/write (truncate first)
# +>> — read/append

# =====================
# Read methods
# =====================

open($fh, '<', 'output.txt') or die $!;

# Read one line
my $line = <$fh>;
chomp $line;
print "First line: $line\n";

# Read all lines
seek $fh, 0, 0;   # rewind
my @all_lines = <$fh>;
chomp @all_lines;
printf "Total lines: %d\n", scalar @all_lines;

# Read whole file at once
seek $fh, 0, 0;
my $content;
{ local $/; $content = <$fh>; }   # slurp
printf "Content length: %d\n", length $content;

close $fh;

# =====================
# Slurp mode
# =====================

sub read_file {
    my $file = shift;
    open(my $fh, '<:utf8', $file) or die "Cannot read $file: $!";
    local $/;
    my $content = <$fh>;
    close $fh;
    return $content;
}

sub write_file {
    my ($file, $content) = @_;
    open(my $fh, '>:utf8', $file) or die "Cannot write $file: $!";
    print $fh $content;
    close $fh;
}

sub append_file {
    my ($file, $content) = @_;
    open(my $fh, '>>:utf8', $file) or die "Cannot append $file: $!";
    print $fh $content;
    close $fh;
}

write_file('/tmp/test_perl.txt', "Line 1\nLine 2\nLine 3\n");
my $data = read_file('/tmp/test_perl.txt');
print "Read: $data";
append_file('/tmp/test_perl.txt', "Line 4\n");

# =====================
# Binary mode
# =====================

sub copy_binary {
    my ($src, $dst) = @_;
    open(my $in,  '<:raw', $src) or die "Cannot read $src: $!";
    open(my $out, '>:raw', $dst) or die "Cannot write $dst: $!";
    
    my $buffer;
    while (read($in, $buffer, 4096)) {
        print $out $buffer;
    }
    close $in;
    close $out;
}
```

---

## Step 122: Encoding และ Unicode

```perl
#!/usr/bin/perl
use strict;
use warnings;
use utf8;
use open ':std', ':encoding(UTF-8)';
use Encode qw(encode decode);

# =====================
# UTF-8 file I/O
# =====================

sub write_utf8 {
    my ($file, $content) = @_;
    open(my $fh, '>:encoding(UTF-8)', $file) or die $!;
    print $fh $content;
    close $fh;
}

sub read_utf8 {
    my $file = shift;
    open(my $fh, '<:encoding(UTF-8)', $file) or die $!;
    local $/;
    my $content = <$fh>;
    close $fh;
    return $content;
}

write_utf8('/tmp/thai.txt', "สวัสดีครับ\nPerl ภาษาไทย\n");
my $thai = read_utf8('/tmp/thai.txt');
print "Thai: $thai";

# =====================
# Detect encoding
# =====================

# ตรวจสอบว่าข้อมูลเป็น valid UTF-8
use Encode qw(decode FB_QUIET);

sub is_valid_utf8 {
    my $bytes = shift;
    my $decoded = eval { decode('UTF-8', $bytes, Encode::FB_CROAK) };
    return !$@;
}

my $utf8_bytes = "\xe0\xb8\xaa\xe0\xb8\xa7\xe0\xb8\xb1\xe0\xb8\xaa\xe0\xb8\x94\xe0\xb8\xb5";
print "Valid UTF-8: ", is_valid_utf8($utf8_bytes) ? "yes" : "no", "\n";

# =====================
# Latin-1 to UTF-8
# =====================

my $latin1 = "Caf\xe9";  # "Café" in Latin-1
my $utf8   = decode('Latin-1', $latin1);
print "Decoded: $utf8\n";

# =====================
# String operations with Unicode
# =====================

use Unicode::Normalize qw(NFC NFD);

my $str = "café";
my $nfc = NFC($str);
my $nfd = NFD($str);

printf "Length NFC: %d, NFD: %d\n", length($nfc), length($nfd);

# =====================
# JSON with Unicode
# =====================

use JSON::PP;

my %data = (
    name => "สมชาย",
    city => "กรุงเทพ",
    email => "somchai\@example.com",
);

my $json = encode_json(\%data);
print "JSON: $json\n";

my $decoded = decode_json($json);
print "Name: $decoded->{name}\n";
```

---

## Step 123: Directory Operations

```perl
#!/usr/bin/perl
use strict;
use warnings;
use File::Path qw(make_path remove_tree);
use File::Copy qw(copy move);
use File::Basename qw(basename dirname);
use File::Spec;

# =====================
# Directory listing
# =====================

sub list_dir {
    my ($dir, $pattern) = @_;
    $pattern //= qr/.*/;
    
    opendir(my $dh, $dir) or die "Cannot opendir $dir: $!";
    my @entries = grep { !(/^\.\.?$/) && /$pattern/ } readdir $dh;
    closedir $dh;
    
    return map { File::Spec->catfile($dir, $_) } sort @entries;
}

my @entries = list_dir('/tmp', qr/\.txt$/);
printf "txt files in /tmp: %d\n", scalar @entries;

# =====================
# File info
# =====================

sub file_info {
    my $path = shift;
    return () unless -e $path;
    
    my @stat = stat $path;
    return (
        path    => $path,
        name    => basename($path),
        dir     => dirname($path),
        size    => $stat[7],
        mtime   => $stat[9],
        is_dir  => -d $path ? 1 : 0,
        is_file => -f $path ? 1 : 0,
        mode    => sprintf("%04o", $stat[2] & 07777),
        readable   => -r $path ? 1 : 0,
        writable   => -w $path ? 1 : 0,
        executable => -x $path ? 1 : 0,
    );
}

for my $path ('/tmp', '/tmp/thai.txt', '/nonexistent') {
    my %info = file_info($path);
    next unless %info;
    printf "%-30s size=%-8s type=%s\n",
        $info{name}, $info{size}//"N/A",
        $info{is_dir} ? "dir" : "file";
}

# =====================
# Recursive file listing
# =====================

sub find_files {
    my ($dir, %opts) = @_;
    $opts{pattern} //= qr/.*/;
    $opts{recursive} //= 1;
    $opts{max_depth} //= 999;
    
    my @result;
    
    _find_recursive($dir, 0, \@result, \%opts);
    
    return @result;
}

sub _find_recursive {
    my ($dir, $depth, $result, $opts) = @_;
    return if $depth > $opts->{max_depth};
    
    opendir(my $dh, $dir) or return;
    my @entries = grep { !/^\.\.?$/ } readdir $dh;
    closedir $dh;
    
    for my $entry (sort @entries) {
        my $path = File::Spec->catfile($dir, $entry);
        
        if (-d $path && $opts->{recursive}) {
            _find_recursive($path, $depth+1, $result, $opts);
        } elsif (-f $path && $path =~ $opts->{pattern}) {
            push @$result, $path;
        }
    }
}

my @perl_files = find_files('/home/user/perl_course', pattern => qr/\.md$/, max_depth => 2);
printf "Found %d .md files\n", scalar @perl_files;

# =====================
# Create/remove directories
# =====================

make_path('/tmp/perl_test/sub1/sub2', { chmod => 0755 });
print "Created nested dirs\n";

remove_tree('/tmp/perl_test');
print "Removed dirs\n";

# =====================
# File::Basename
# =====================

my $path = "/home/user/perl_course/parts/part_01.md";
printf "basename: %s\n", basename($path);
printf "dirname:  %s\n", dirname($path);
printf "suffix:   %s\n", (File::Basename::fileparse($path, qr/\.[^.]*/))[-1];
```

---

## Step 124: File Locking

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Fcntl qw(:flock);

# =====================
# flock — file locking
# =====================

sub safe_append {
    my ($file, $content) = @_;
    
    open(my $fh, '>>', $file) or die "Cannot open $file: $!";
    
    flock($fh, LOCK_EX) or die "Cannot lock $file: $!";
    # Seek to end again after getting lock (in case another process wrote)
    seek($fh, 0, 2);
    
    print $fh $content;
    
    flock($fh, LOCK_UN);
    close $fh;
}

sub safe_read {
    my $file = shift;
    
    open(my $fh, '<', $file) or die "Cannot open $file: $!";
    flock($fh, LOCK_SH) or die "Cannot lock $file: $!";
    
    local $/;
    my $content = <$fh>;
    
    flock($fh, LOCK_UN);
    close $fh;
    
    return $content;
}

safe_append('/tmp/lock_test.txt', "Line 1\n");
safe_append('/tmp/lock_test.txt', "Line 2\n");
my $content = safe_read('/tmp/lock_test.txt');
print "Content:\n$content";

# =====================
# Non-blocking lock
# =====================

sub try_exclusive_lock {
    my $file = shift;
    
    open(my $fh, '>>', $file) or die "Cannot open $file: $!";
    
    # LOCK_NB = non-blocking
    unless (flock($fh, LOCK_EX | LOCK_NB)) {
        close $fh;
        return undef;   # already locked
    }
    
    return $fh;   # caller must close/unlock
}

my $fh = try_exclusive_lock('/tmp/app.lock');
if ($fh) {
    print "Got exclusive lock\n";
    # do work...
    flock($fh, LOCK_UN);
    close $fh;
} else {
    print "Cannot get lock — another process is running\n";
}

# =====================
# PID file pattern
# =====================

sub create_pidfile {
    my $pidfile = shift;
    
    if (-e $pidfile) {
        open(my $fh, '<', $pidfile) or return 0;
        my $pid = <$fh>;
        close $fh;
        chomp $pid;
        
        # Check if process still running
        if (kill(0, $pid)) {
            warn "Process $pid already running\n";
            return 0;
        }
        unlink $pidfile;
    }
    
    open(my $fh, '>', $pidfile) or die "Cannot create pidfile: $!";
    print $fh $$, "\n";   # $$ = current PID
    close $fh;
    
    return 1;
}

sub remove_pidfile {
    my $pidfile = shift;
    unlink $pidfile;
}

if (create_pidfile('/tmp/myapp.pid')) {
    print "Started with PID $$\n";
    # do work...
    remove_pidfile('/tmp/myapp.pid');
}
```

---

## Step 125: File::Find และ การค้นหาไฟล์

```perl
#!/usr/bin/perl
use strict;
use warnings;
use File::Find;
use File::Basename;
use POSIX qw(strftime);

# =====================
# File::Find::find
# =====================

my @found_files;
find(
    {
        wanted => sub {
            return unless -f $_;
            return unless /\.md$/i;
            push @found_files, $File::Find::name;
        },
        no_chdir => 1,
    },
    '/home/user/perl_course'
);

printf "Found %d .md files\n", scalar @found_files;

# =====================
# Advanced find with criteria
# =====================

sub find_files_advanced {
    my ($dir, %criteria) = @_;
    my @results;
    
    find(
        {
            wanted => sub {
                return unless -f $_;
                
                my $name = basename($File::Find::name);
                my @stat = stat $_;
                my ($size, $mtime) = @stat[7, 9];
                
                # Apply criteria
                return if $criteria{pattern} && $name !~ $criteria{pattern};
                return if $criteria{min_size} && $size < $criteria{min_size};
                return if $criteria{max_size} && $size > $criteria{max_size};
                return if $criteria{newer_than} && $mtime <= $criteria{newer_than};
                
                push @results, {
                    path  => $File::Find::name,
                    name  => $name,
                    size  => $size,
                    mtime => $mtime,
                };
            },
            no_chdir => 1,
        },
        $dir
    );
    
    # Sort by criteria
    if ($criteria{sort_by}) {
        if ($criteria{sort_by} eq 'size') {
            @results = sort { $b->{size} <=> $a->{size} } @results;
        } elsif ($criteria{sort_by} eq 'mtime') {
            @results = sort { $b->{mtime} <=> $a->{mtime} } @results;
        } elsif ($criteria{sort_by} eq 'name') {
            @results = sort { $a->{name} cmp $b->{name} } @results;
        }
    }
    
    return @results;
}

my @md_files = find_files_advanced(
    '/home/user/perl_course',
    pattern => qr/\.md$/,
    sort_by => 'size',
);

printf "\nLargest .md files:\n";
for my $file (@md_files[0..4]) {
    printf "  %-40s %6d bytes\n", 
        basename($file->{path}), $file->{size};
}

# =====================
# Grep through files
# =====================

sub grep_files {
    my ($dir, $pattern) = @_;
    my @matches;
    
    find(
        {
            wanted => sub {
                return unless -f $_ && /\.(pl|pm|md)$/;
                my $file = $File::Find::name;
                
                open(my $fh, '<:utf8', $file) or return;
                my $lineno = 0;
                while (my $line = <$fh>) {
                    $lineno++;
                    if ($line =~ $pattern) {
                        chomp $line;
                        push @matches, {
                            file   => $file,
                            line   => $lineno,
                            text   => $line,
                        };
                    }
                }
                close $fh;
            },
            no_chdir => 1,
        },
        $dir
    );
    
    return @matches;
}

my @results = grep_files('/home/user/perl_course', qr/Step \d+:/);
printf "\nFound 'Step N:' in %d places\n", scalar @results;
for my $r (@results[0..4]) {
    printf "  %s:%d\n", basename($r->{file}), $r->{line};
}
```

---

## Step 126: Temporary Files

```perl
#!/usr/bin/perl
use strict;
use warnings;
use File::Temp qw(tempfile tempdir);
use POSIX qw(strftime);

# =====================
# tempfile
# =====================

# Create temp file (auto-deleted when $fh goes out of scope)
my ($fh, $filename) = tempfile();
print $fh "Temporary content\n";
print $fh "Another line\n";
close $fh;

print "Temp file: $filename\n";
# File exists
print "Exists: ", (-e $filename ? "yes" : "no"), "\n";

# =====================
# tempfile with options
# =====================

my ($tmpfh, $tmpname) = tempfile(
    'myapp_XXXXXXXX',   # template (X gets replaced)
    DIR    => '/tmp',
    SUFFIX => '.tmp',
    UNLINK => 1,        # auto-delete
);

printf "Named temp: %s\n", $tmpname;
print $tmpfh "data here\n";
close $tmpfh;

# =====================
# tempdir
# =====================

my $tmpdir = tempdir(CLEANUP => 1);
print "Temp dir: $tmpdir\n";

# Use it
open(my $tf, '>', "$tmpdir/file1.txt") or die $!;
print $tf "File 1\n";
close $tf;

open($tf, '>', "$tmpdir/file2.txt") or die $!;
print $tf "File 2\n";
close $tf;

opendir(my $dh, $tmpdir) or die $!;
my @files = grep { !/^\./ } readdir $dh;
closedir $dh;
print "Files in tmpdir: @files\n";

# =====================
# Atomic write (write to temp, then rename)
# =====================

sub atomic_write {
    my ($target, $content) = @_;
    
    my ($tmpfh, $tmpfile) = tempfile(
        DIR    => dirname($target) // '/tmp',
        UNLINK => 0,
    );
    
    eval {
        print $tmpfh $content;
        close $tmpfh;
        rename $tmpfile, $target or die "Cannot rename: $!";
    };
    
    if ($@) {
        unlink $tmpfile;
        die "atomic_write failed: $@";
    }
}

sub dirname { my $p = shift; $p =~ s|/[^/]+$||; $p || "." }

atomic_write('/tmp/config.txt', "key=value\nother=setting\n");
print "Atomic write done\n";
```

---

## Step 127: CSV, JSON, YAML ไฟล์

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Text::CSV;
use JSON::PP;

# =====================
# Text::CSV
# =====================

my @people = (
    { name => "Alice",   age => 28, city => "Bangkok",  job => "Engineer" },
    { name => "Bob",     age => 35, city => "Chiang Mai", job => "Manager" },
    { name => "Carol",   age => 42, city => "Phuket",    job => "Director" },
    { name => "Dave",    age => 24, city => "Bangkok",   job => "Developer" },
);

my @headers = qw(name age city job);

# Write CSV
{
    my $csv = Text::CSV->new({ binary => 1, eol => "\n" });
    open(my $fh, '>:encoding(utf8)', '/tmp/people.csv') or die $!;
    $csv->print($fh, \@headers);
    $csv->print($fh, [ map { $_->{$_} } @headers ]) for @people;
    
    # Fix: proper column access
    close $fh;
    
    open($fh, '>:encoding(utf8)', '/tmp/people.csv') or die $!;
    $csv->print($fh, \@headers);
    for my $p (@people) {
        $csv->print($fh, [ map { $p->{$_} } @headers ]);
    }
    close $fh;
    print "CSV written\n";
}

# Read CSV
{
    my $csv = Text::CSV->new({ binary => 1, auto_diag => 1 });
    open(my $fh, '<:encoding(utf8)', '/tmp/people.csv') or die $!;
    
    my $header_row = $csv->getline($fh);
    $csv->column_names(@$header_row);
    
    my @rows;
    while (my $row = $csv->getline_hr($fh)) {
        push @rows, $row;
    }
    close $fh;
    
    printf "CSV: %d rows\n", scalar @rows;
    printf "  %s: %s, %s, %s\n", 
        $_{name}//"", $_{age}//"", $_{city}//"", $_{job}//""
        for @rows;
}

# =====================
# JSON
# =====================

# Write JSON
my $json = JSON::PP->new->utf8->pretty->canonical;

my %config = (
    database => { host => "localhost", port => 5432, name => "myapp" },
    cache    => { ttl => 300, max_size => 1000 },
    features => ["auth", "api", "admin"],
    debug    => 0,
);

my $json_str = $json->encode(\%config);
open(my $fh, '>', '/tmp/config.json') or die $!;
print $fh $json_str;
close $fh;
print "JSON written\n";

# Read JSON  
open($fh, '<', '/tmp/config.json') or die $!;
local $/;
my $raw = <$fh>;
close $fh;

my $cfg = $json->decode($raw);
printf "DB host: %s:%d\n", $cfg->{database}{host}, $cfg->{database}{port};
printf "Features: %s\n", join(", ", @{$cfg->{features}});

# =====================
# JSON streaming (large files)
# =====================

sub write_json_lines {
    my ($file, @records) = @_;
    open(my $fh, '>', $file) or die $!;
    my $json = JSON::PP->new->utf8->canonical;
    print $fh $json->encode($_), "\n" for @records;
    close $fh;
}

sub read_json_lines {
    my $file = shift;
    my $json = JSON::PP->new->utf8;
    open(my $fh, '<', $file) or die $!;
    my @records;
    while (my $line = <$fh>) {
        chomp $line;
        push @records, $json->decode($line) if length $line;
    }
    close $fh;
    return @records;
}

write_json_lines('/tmp/people.jsonl', @people);
my @loaded = read_json_lines('/tmp/people.jsonl');
printf "Loaded %d people from JSONL\n", scalar @loaded;
```

---

## Step 128: DBM Files

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DB_File;
use Fcntl;

# =====================
# DB_File (Berkeley DB)
# =====================

# Create/open DB
my %db;
tie %db, 'DB_File', '/tmp/test.db', O_RDWR|O_CREAT, 0666, $DB_HASH
    or die "Cannot tie: $!";

# Write
$db{alice}  = "alice\@example.com";
$db{bob}    = "bob\@example.com";
$db{carol}  = "carol\@example.com";

# Read
print "Alice: $db{alice}\n";
print "Exists: ", (exists $db{bob} ? "yes" : "no"), "\n";

# Iterate
print "\nAll entries:\n";
my ($key, $val);
my $cursor = (tied %db)->db_cursor;
while ($cursor->c_get($key, $val, DB_NEXT) == 0) {
    print "  $key => $val\n";
}

# Delete
delete $db{carol};

# Count
printf "Entries: %d\n", scalar keys %db;

untie %db;

# =====================
# GDBM_File (simpler)
# =====================

use GDBM_File;

my %cache;
tie %cache, 'GDBM_File', '/tmp/cache.gdbm', GDBM_WRCREAT, 0666
    or die "Cannot tie: $!";

# Use like a hash
$cache{"user:1"} = "Alice";
$cache{"user:2"} = "Bob";
$cache{"config:timeout"} = "30";

printf "cache entries: %d\n", scalar keys %cache;
printf "user:1 = %s\n", $cache{"user:1"};

untie %cache;

# =====================
# Storable — structured data
# =====================

use Storable qw(store retrieve freeze thaw);

my %data = (
    users => [
        { id => 1, name => "Alice", scores => [95, 87, 92] },
        { id => 2, name => "Bob",   scores => [78, 82, 90] },
    ],
    settings => { debug => 0, version => "1.0" },
);

# Store to file
store \%data, '/tmp/data.store';
print "Stored to file\n";

# Retrieve from file
my $loaded = retrieve '/tmp/data.store';
printf "Loaded %d users\n", scalar @{$loaded->{users}};
printf "First user: %s\n", $loaded->{users}[0]{name};

# In-memory serialization
my $frozen = freeze \%data;
my $thawed = thaw $frozen;
print "Frozen size: ", length $frozen, " bytes\n";
```

---

## Step 129: Log Files

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(strftime);
use Fcntl qw(:flock);

# =====================
# Simple logger
# =====================

package Logger;

sub new {
    my ($class, %opts) = @_;
    return bless {
        file    => $opts{file}    // '/tmp/app.log',
        level   => $opts{level}   // 'INFO',
        prefix  => $opts{prefix}  // '',
        rotate  => $opts{rotate}  // 0,
        max_size => $opts{max_size} // 10 * 1024 * 1024,  # 10MB
    }, $class;
}

my %LEVELS = (DEBUG => 0, INFO => 1, WARN => 2, ERROR => 3, FATAL => 4);

sub _should_log {
    my ($self, $level) = @_;
    return ($LEVELS{$level} // 0) >= ($LEVELS{$self->{level}} // 0);
}

sub log {
    my ($self, $level, $msg) = @_;
    return unless $self->_should_log($level);
    
    $self->_rotate if $self->{rotate} && -e $self->{file} && -s $self->{file} > $self->{max_size};
    
    open(my $fh, '>>', $self->{file}) or die "Cannot open log: $!";
    flock($fh, LOCK_EX) or die "Cannot lock log: $!";
    seek($fh, 0, 2);
    
    my $ts = strftime "%Y-%m-%d %H:%M:%S", localtime;
    my $prefix = $self->{prefix} ? "[$self->{prefix}] " : "";
    printf $fh "[%s] [%-5s] %s%s\n", $ts, $level, $prefix, $msg;
    
    flock($fh, LOCK_UN);
    close $fh;
}

sub _rotate {
    my $self = shift;
    my $rotated = $self->{file} . '.' . strftime("%Y%m%d%H%M%S", localtime);
    rename $self->{file}, $rotated;
}

sub debug { $_[0]->log('DEBUG', $_[1]) }
sub info  { $_[0]->log('INFO',  $_[1]) }
sub warn  { $_[0]->log('WARN',  $_[1]) }
sub error { $_[0]->log('ERROR', $_[1]) }
sub fatal { $_[0]->log('FATAL', $_[1]) }

package main;

my $log = Logger->new(file => '/tmp/myapp.log', level => 'DEBUG', prefix => 'app');

$log->debug("Starting up");
$log->info("Server started on port 8080");
$log->warn("Config file not found, using defaults");
$log->error("Database connection failed: timeout");

# =====================
# Log rotation and analysis
# =====================

sub analyze_log {
    my $file = shift;
    
    my %stats = (total => 0);
    
    open(my $fh, '<', $file) or return %stats;
    while (my $line = <$fh>) {
        next unless $line =~ /\[(\w+)\s*\]/;
        my $level = $1;
        $stats{total}++;
        $stats{$level}++;
    }
    close $fh;
    
    return %stats;
}

my %log_stats = analyze_log('/tmp/myapp.log');
print "\nLog statistics:\n";
printf "  %-8s: %d\n", $_, $log_stats{$_} // 0 
    for qw(total DEBUG INFO WARN ERROR FATAL);
```

---

## Step 130: โปรแกรมสรุป — File Manager

```perl
#!/usr/bin/perl
#
# file_manager.pl — จัดการไฟล์และโฟลเดอร์
#

use strict;
use warnings;
use File::Path qw(make_path remove_tree);
use File::Copy qw(copy move);
use File::Basename qw(basename dirname);
use File::Find;
use POSIX qw(strftime);
use Fcntl qw(:flock);

# =====================
# Utility functions
# =====================

sub format_size {
    my $bytes = shift;
    return "${bytes}B" if $bytes < 1024;
    return sprintf "%.1fK", $bytes/1024 if $bytes < 1024**2;
    return sprintf "%.1fM", $bytes/1024**2 if $bytes < 1024**3;
    return sprintf "%.1fG", $bytes/1024**3;
}

sub format_time {
    my $ts = shift;
    return strftime "%Y-%m-%d %H:%M", localtime $ts;
}

# =====================
# Directory listing
# =====================

sub list_directory {
    my ($dir, %opts) = @_;
    $opts{show_hidden} //= 0;
    $opts{sort_by}     //= 'name';
    
    opendir(my $dh, $dir) or die "Cannot open $dir: $!";
    my @entries = readdir $dh;
    closedir $dh;
    
    @entries = grep { !/^\.\.?$/ } @entries;
    @entries = grep { !/^\./ }    @entries unless $opts{show_hidden};
    
    my @items;
    for my $name (@entries) {
        my $path = "$dir/$name";
        my @stat = stat $path;
        
        push @items, {
            name  => $name,
            path  => $path,
            size  => $stat[7] // 0,
            mtime => $stat[9] // 0,
            is_dir => -d $path ? 1 : 0,
        };
    }
    
    # Sort
    if ($opts{sort_by} eq 'size') {
        @items = sort { $b->{size} <=> $a->{size} } @items;
    } elsif ($opts{sort_by} eq 'mtime') {
        @items = sort { $b->{mtime} <=> $a->{mtime} } @items;
    } else {  # name
        @items = sort {
            ($b->{is_dir} <=> $a->{is_dir}) ||
            ($a->{name} cmp $b->{name})
        } @items;
    }
    
    return @items;
}

sub print_listing {
    my ($dir, @items) = @_;
    
    printf "\n  Directory: %s\n", $dir;
    printf "  %-40s %8s  %s  %s\n", "Name", "Size", "Modified", "Type";
    print "  " . "-" x 70 . "\n";
    
    for my $item (@items) {
        printf "  %-40s %8s  %s  %s\n",
            ($item->{is_dir} ? "[" . $item->{name} . "]" : $item->{name}),
            ($item->{is_dir} ? "<DIR>" : format_size($item->{size})),
            format_time($item->{mtime}),
            ($item->{is_dir} ? "directory" : "file");
    }
    
    my $dirs  = grep { $_->{is_dir} }  @items;
    my $files = grep { !$_->{is_dir} } @items;
    my $total = 0;
    $total += $_->{size} for @items;
    
    printf "\n  %d directories, %d files, %s total\n",
        $dirs, $files, format_size($total);
}

# =====================
# File operations
# =====================

sub copy_file {
    my ($src, $dst) = @_;
    
    die "Source not found: $src\n" unless -e $src;
    
    if (-d $dst) {
        $dst = "$dst/" . basename($src);
    }
    
    copy($src, $dst) or die "Cannot copy: $!\n";
    return $dst;
}

sub move_file {
    my ($src, $dst) = @_;
    
    die "Source not found: $src\n" unless -e $src;
    
    if (-d $dst) {
        $dst = "$dst/" . basename($src);
    }
    
    move($src, $dst) or die "Cannot move: $!\n";
    return $dst;
}

sub delete_path {
    my $path = shift;
    die "Path not found: $path\n" unless -e $path;
    
    if (-d $path) {
        remove_tree($path) or die "Cannot remove dir: $!\n";
    } else {
        unlink $path or die "Cannot delete: $!\n";
    }
}

# =====================
# Search files
# =====================

sub search {
    my ($dir, %opts) = @_;
    $opts{name}    //= qr/.*/;
    $opts{content} //= undef;
    $opts{size_gt} //= 0;
    
    my @results;
    
    find(
        {
            wanted => sub {
                return unless -f $_;
                my $name = basename($File::Find::name);
                return unless $name =~ $opts{name};
                return if -s $_ < $opts{size_gt};
                
                if ($opts{content}) {
                    open(my $fh, '<', $_) or return;
                    local $/;
                    my $text = <$fh>;
                    close $fh;
                    return unless $text =~ $opts{content};
                }
                
                push @results, {
                    path  => $File::Find::name,
                    name  => $name,
                    size  => -s $_,
                    mtime => (stat $_)[9],
                };
            },
            no_chdir => 1,
        },
        $dir
    );
    
    return @results;
}

# =====================
# Run demo
# =====================

my $base = '/home/user/perl_course';

print "=== File Manager Demo ===\n";

# List course directory
my @items = list_directory($base);
print_listing($base, @items);

# Search for large md files
print "\n=== Large .md files ===\n";
my @large = search($base, name => qr/\.md$/, size_gt => 10000);
@large = sort { $b->{size} <=> $a->{size} } @large;

printf "Found %d files > 10KB:\n", scalar @large;
printf "  %-30s %8s\n", basename($_->{path}), format_size($_->{size})
    for @large[0..9];

# Disk usage summary
print "\n=== Disk Usage ===\n";
my $total_size = 0;
my $total_files = 0;
find(
    {
        wanted => sub {
            return unless -f $_;
            $total_size += -s $_;
            $total_files++;
        },
        no_chdir => 1,
    },
    $base
);
printf "Total: %d files, %s\n", $total_files, format_size($total_size);
```

---

## สรุป Part 13

ใน Part นี้คุณได้เรียนรู้:
- ✅ File open modes และ filehandles
- ✅ Encoding และ Unicode
- ✅ Directory operations (opendir, readdir)
- ✅ File::Find
- ✅ File locking (flock)
- ✅ Temporary files (File::Temp)
- ✅ CSV, JSON file handling
- ✅ DBM/Berkeley DB
- ✅ Log files และ rotation
- ✅ โปรแกรม File Manager

**ถัดไป: [Part 14 — Modules และ Packages](part_14.md)**
