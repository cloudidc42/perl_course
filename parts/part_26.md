# Part 26: Advanced File I/O
## Steps 251-260: การจัดการไฟล์ขั้นสูง

---

## Step 251: File Handle ขั้นสูง

```perl
#!/usr/bin/perl
use strict;
use warnings;
use autodie;

# =====================
# Modern file handling
# =====================

my $tmpdir = "/tmp/perl_io_$$";
mkdir $tmpdir;

# Write with say
open my $fh, '>', "$tmpdir/test.txt";
say $fh "Line 1";
say $fh "Line 2";
say $fh "Line 3";
close $fh;

# Read line by line
open my $in, '<', "$tmpdir/test.txt";
while (my $line = <$in>) {
    chomp $line;
    printf "Read: '%s'\n", $line;
}
close $in;

# Append
open my $app, '>>', "$tmpdir/test.txt";
say $app "Line 4";
say $app "Line 5";
close $app;

# Read all lines into array
open $in, '<', "$tmpdir/test.txt";
my @lines = <$in>;
close $in;
chomp @lines;
printf "Total lines: %d\n", scalar @lines;

# Read entire file into scalar
open $in, '<', "$tmpdir/test.txt";
my $content = do { local $/; <$in> };
close $in;
printf "File size: %d bytes\n", length($content);

# =====================
# In-memory file handle
# =====================

my $buffer = "";
open my $mem, '>', \$buffer;
print $mem "In-memory content\n";
print $mem "Second line\n";
close $mem;

printf "\nIn-memory buffer:\n%s", $buffer;

# Read from in-memory
open my $mem_in, '<', \$buffer;
my @mem_lines = <$mem_in>;
close $mem_in;
chomp @mem_lines;
printf "Lines in buffer: %d\n", scalar @mem_lines;

# =====================
# Binmode
# =====================

# Write binary data
open my $bin, '>:raw', "$tmpdir/binary.bin";
print $bin pack('N', 0xDEADBEEF);  # 4 bytes big-endian
print $bin pack('n', 1234);         # 2 bytes big-endian
close $bin;

open $bin, '<:raw', "$tmpdir/binary.bin";
my $data;
read $bin, $data, 6;
close $bin;

my ($magic, $val) = unpack('Nn', $data);
printf "\nBinary: magic=0x%08X val=%d\n", $magic, $val;

# UTF-8
open my $utf, '>:utf8', "$tmpdir/utf8.txt";
print $utf "สวัสดีครับ\n";
print $utf "Hello World\n";
close $utf;

open $utf, '<:utf8', "$tmpdir/utf8.txt";
while (<$utf>) { chomp; printf "UTF-8: %s\n", $_ }
close $utf;

# Cleanup
unlink "$tmpdir/$_" for qw(test.txt binary.bin utf8.txt);
rmdir $tmpdir;
```

---

## Step 252: File::Spec and Path Operations

```perl
#!/usr/bin/perl
use strict;
use warnings;
use File::Spec;
use File::Basename;
use Cwd qw(getcwd abs_path);

# =====================
# File::Spec — portable path operations
# =====================

my $path = File::Spec->catfile("home", "user", "documents", "file.txt");
printf "Path: %s\n", $path;

my ($vol, $dir, $file) = File::Spec->splitpath("/home/user/docs/file.txt");
printf "Volume: '%s'\n", $vol;
printf "Dir: '%s'\n", $dir;
printf "File: '%s'\n", $file;

my @dirs = File::Spec->splitdir("/home/user/documents");
printf "Dirs: %s\n", join(", ", @dirs);

printf "\nAbs path: %s\n", File::Spec->rel2abs("../test.txt");
printf "Root: %s\n",       File::Spec->rootdir;
printf "Curdir: %s\n",     File::Spec->curdir;
printf "Updir: %s\n",      File::Spec->updir;

# =====================
# File::Basename
# =====================

my @files = (
    "/home/user/script.pl",
    "/var/log/app.log.gz",
    "just_name",
    "/path/to/dir/",
);

for my $f (@files) {
    printf "\nFile: %s\n", $f;
    printf "  basename: %s\n",  basename($f);
    printf "  dirname:  %s\n",  dirname($f);
    
    if ($f =~ /\./) {
        my ($name, $dir, $suffix) = fileparse($f, qr/\.[^.]*/);
        printf "  name:     %s\n", $name;
        printf "  suffix:   %s\n", $suffix;
    }
}

# =====================
# Path building utility
# =====================

{
package PathBuilder;

sub new {
    my ($class, $base) = @_;
    return bless { parts => [$base // "."] }, $class;
}

sub join_path {
    my ($self, @parts) = @_;
    return bless { parts => [@{$self->{parts}}, @parts] }, ref $self;
}

sub to_string {
    File::Spec->catfile(@{$_[0]->{parts}});
}

sub exists  { -e $_[0]->to_string }
sub is_file { -f $_[0]->to_string }
sub is_dir  { -d $_[0]->to_string }

use overload '""' => \&to_string, fallback => 1;
}

my $base = PathBuilder->new("/tmp");
my $logs = $base->join_path("logs");
my $file = $logs->join_path("app.log");

printf "\nPath: %s\n", $file;
printf "Exists: %s\n", $file->exists ? "yes" : "no";
printf "Is dir: %s\n", $logs->is_dir ? "yes" : "no";

# CWD
printf "\nCWD: %s\n", getcwd();
```

---

## Step 253: Directory Operations

```perl
#!/usr/bin/perl
use strict;
use warnings;
use File::Path qw(make_path remove_tree);
use File::Find;
use File::Copy;

my $base = "/tmp/perl_dir_test_$$";

# =====================
# Create directory tree
# =====================

make_path(
    "$base/src/lib",
    "$base/src/bin",
    "$base/tests",
    "$base/docs",
    { mode => 0755, verbose => 0 },
);

# Create some test files
for my $dir ("$base/src/lib", "$base/src/bin", "$base/tests") {
    open my $fh, '>', "$dir/sample.txt";
    print $fh "Sample content in $dir\n";
    close $fh;
}

open my $fh, '>', "$base/README.md";
print $fh "# Test project\n";
close $fh;

# =====================
# Read directory
# =====================

printf "Contents of $base:\n";
opendir my $dh, $base;
while (my $entry = readdir $dh) {
    next if $entry =~ /^\./;
    printf "  %s %s\n", -d "$base/$entry" ? "DIR" : "FILE", $entry;
}
closedir $dh;

# =====================
# File::Find
# =====================

printf "\nAll .txt files:\n";
File::Find::find(sub {
    return unless -f && /\.txt$/;
    printf "  %s\n", $File::Find::name;
}, $base);

# =====================
# Recursive listing (manual)
# =====================

sub list_tree {
    my ($dir, $indent) = @_;
    $indent //= 0;
    
    opendir my $dh, $dir;
    my @entries = sort grep { !/^\./ } readdir $dh;
    closedir $dh;
    
    for my $e (@entries) {
        my $path = "$dir/$e";
        printf "%s%s %s\n", "  " x $indent, -d $path ? "D" : "F", $e;
        list_tree($path, $indent + 1) if -d $path;
    }
}

printf "\nDirectory tree:\n";
list_tree($base);

# =====================
# File statistics
# =====================

sub dir_stats {
    my $dir = shift;
    my (%stats);
    $stats{files} = $stats{dirs} = $stats{total_size} = 0;
    
    File::Find::find(sub {
        if (-f) {
            $stats{files}++;
            $stats{total_size} += -s;
        } elsif (-d && $File::Find::name ne $dir) {
            $stats{dirs}++;
        }
    }, $dir);
    
    return %stats;
}

my %stats = dir_stats($base);
printf "\nDirectory stats for %s:\n", $base;
printf "  Files: %d\n", $stats{files};
printf "  Dirs:  %d\n", $stats{dirs};
printf "  Size:  %d bytes\n", $stats{total_size};

# Cleanup
remove_tree($base, { verbose => 0 });
printf "\nCleaned up.\n";
```

---

## Step 254: File Locking

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Fcntl qw(:flock SEEK_SET SEEK_END);

# =====================
# File locking with flock
# =====================

my $lockfile = "/tmp/perl_lock_$$";
my $datafile = "/tmp/perl_data_$$";

# Write initial data
open my $init, '>', $datafile;
print $init "0\n";
close $init;

# Counter with file locking (simulated concurrent access)
sub increment_counter {
    my ($id) = @_;
    
    open my $fh, '+<', $datafile or die "Can't open: $!";
    
    # Exclusive lock
    flock($fh, LOCK_EX) or die "Can't lock: $!";
    
    # Read current value
    seek($fh, 0, SEEK_SET);
    my $val = <$fh>;
    chomp $val;
    
    # Increment
    $val++;
    
    # Write back
    seek($fh, 0, SEEK_SET);
    print $fh "$val\n";
    truncate($fh, tell($fh));
    
    # Unlock (happens on close too)
    flock($fh, LOCK_UN);
    close $fh;
    
    return $val;
}

# Simulate concurrent increments
my @results;
for my $i (1..5) {
    push @results, increment_counter($i);
}

printf "Counter values: %s\n", join(", ", @results);
printf "Final value: %d\n", $results[-1];

# =====================
# Shared lock for reading
# =====================

sub read_with_shared_lock {
    open my $fh, '<', $datafile;
    flock($fh, LOCK_SH);  # shared lock — multiple readers OK
    my $val = do { local $/; <$fh> };
    chomp $val;
    flock($fh, LOCK_UN);
    close $fh;
    return $val;
}

my $current = read_with_shared_lock();
printf "Current value (shared read): %d\n", $current;

# =====================
# Lock file pattern
# =====================

sub with_lock {
    my ($lockpath, $code) = @_;
    
    open my $lock_fh, '>', $lockpath or die "Can't create lock: $!";
    
    if (!flock($lock_fh, LOCK_EX | LOCK_NB)) {
        close $lock_fh;
        die "Another process holds the lock\n";
    }
    
    print $lock_fh "$$\n";  # write our PID
    
    eval { $code->() };
    my $err = $@;
    
    flock($lock_fh, LOCK_UN);
    close $lock_fh;
    unlink $lockpath;
    
    die $err if $err;
}

with_lock("/tmp/myapp_$$.lock", sub {
    printf "Running with exclusive lock\n";
    # Critical section
    my $v = read_with_shared_lock();
    printf "Value in locked section: %d\n", $v;
});

# Cleanup
unlink $datafile;
```

---

## Step 255: CSV and Structured Files

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# CSV parser (without Text::CSV)
# =====================

sub parse_csv_line {
    my $line = shift;
    my @fields;
    my $pos = 0;
    
    while ($pos < length($line)) {
        if (substr($line, $pos, 1) eq '"') {
            # Quoted field
            my $start = $pos + 1;
            my $end   = $start;
            while ($end < length($line)) {
                if (substr($line, $end, 2) eq '""') {
                    $end += 2;
                } elsif (substr($line, $end, 1) eq '"') {
                    last;
                } else {
                    $end++;
                }
            }
            my $field = substr($line, $start, $end - $start);
            $field =~ s/""/"/g;
            push @fields, $field;
            $pos = $end + 2;  # skip closing " and comma
        } elsif (substr($line, $pos, 1) eq ',') {
            push @fields, "" if $pos == 0 || substr($line, $pos-1, 1) eq ',';
            $pos++;
        } else {
            my $end = index($line, ',', $pos);
            $end = length($line) if $end == -1;
            push @fields, substr($line, $pos, $end - $pos);
            $pos = $end + 1;
        }
    }
    
    return @fields;
}

sub csv_to_array {
    my $text = shift;
    my @rows;
    for my $line (split /\n/, $text) {
        push @rows, [parse_csv_line($line)];
    }
    return @rows;
}

sub array_to_csv {
    my @rows = @_;
    my @lines;
    for my $row (@rows) {
        my @escaped = map {
            /[,"\n]/ ? do { (my $s = $_) =~ s/"/""/g; "\"$s\"" } : $_
        } @$row;
        push @lines, join(',', @escaped);
    }
    return join("\n", @lines) . "\n";
}

# Demo
my $csv_data = qq{name,age,email,city
"Alice Smith",28,alice\@example.com,Bangkok
"Bob, Jr.",35,bob\@example.com,"New York, NY"
"Charlie ""Chuck""",42,chuck\@example.com,London
};

my @rows = csv_to_array($csv_data);
my ($header, @data) = @rows;

printf "Headers: %s\n\n", join(", ", @$header);
for my $row (@data) {
    printf "Name: %-20s Age: %-4s Email: %-25s City: %s\n",
        $row->[0], $row->[1], $row->[2], $row->[3];
}

# Roundtrip test
my $roundtrip = array_to_csv(\@rows);
my @rt_rows = csv_to_array($roundtrip);
printf "\nRoundtrip check: %s\n",
    ($rt_rows[1][0] eq $rows[1][0] ? "PASS" : "FAIL");

# =====================
# TSV (Tab-Separated Values)
# =====================

my @table = (
    [qw(Product Price Stock Category)],
    ["Widget A",  9.99,  100, "Electronics"],
    ["Gadget B",  24.99,  50, "Home"],
    ["Doohickey", 4.99,  200, "Misc"],
);

my $tsv = join("\n", map { join("\t", @$_) } @table) . "\n";

printf "\nTSV:\n%s", $tsv;

# Parse TSV
my @parsed = map { [split /\t/, $_] } split /\n/, $tsv;
printf "\nParsed rows: %d\n", scalar @parsed;
printf "Product: %s, Price: %s\n", $parsed[1][0], $parsed[1][1];
```

---

## Step 256: JSON File Operations

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

my $tmpdir = "/tmp/perl_json_$$";
mkdir $tmpdir;

# =====================
# JSON read/write
# =====================

my $json = JSON::PP->new->utf8->canonical->pretty;

# Write JSON
my $config = {
    app     => "MyApp",
    version => "1.0.0",
    database => {
        host => "localhost",
        port => 5432,
        name => "myapp",
    },
    features => ["auth", "api", "websocket"],
    debug    => JSON::PP::true,
};

open my $fh, '>', "$tmpdir/config.json";
print $fh $json->encode($config);
close $fh;

# Read JSON
open $fh, '<', "$tmpdir/config.json";
my $loaded = $json->decode(do { local $/; <$fh> });
close $fh;

printf "App: %s v%s\n", $loaded->{app}, $loaded->{version};
printf "DB: %s:%d/%s\n", $loaded->{database}{host}, $loaded->{database}{port}, $loaded->{database}{name};
printf "Features: %s\n", join(", ", @{$loaded->{features}});

# =====================
# JSON Lines (JSONL) — one JSON per line
# =====================

my @events = (
    { type => "login",  user => "alice", ts => time() },
    { type => "view",   user => "alice", path => "/home", ts => time() },
    { type => "action", user => "bob",   action => "buy", ts => time() },
    { type => "logout", user => "alice", ts => time() },
);

# Write JSONL
my $jl = JSON::PP->new->utf8;
open $fh, '>', "$tmpdir/events.jsonl";
print $fh $jl->encode($_) . "\n" for @events;
close $fh;

# Read and process JSONL
my @loaded_events;
open $fh, '<', "$tmpdir/events.jsonl";
while (my $line = <$fh>) {
    chomp $line;
    push @loaded_events, $jl->decode($line);
}
close $fh;

printf "\nLoaded %d events\n", scalar @loaded_events;
my @alice_events = grep { $_->{user} eq "alice" } @loaded_events;
printf "Alice's events: %d\n", scalar @alice_events;

# =====================
# Atomic JSON update
# =====================

sub update_json_file {
    my ($path, $updater) = @_;
    
    # Read
    open my $fh, '<', $path;
    my $data = $json->decode(do { local $/; <$fh> });
    close $fh;
    
    # Update
    $updater->($data);
    
    # Write atomically (write to temp, then rename)
    my $tmp = "$path.tmp.$$";
    open $fh, '>', $tmp;
    print $fh $json->encode($data);
    close $fh;
    
    rename $tmp, $path;
}

update_json_file("$tmpdir/config.json", sub {
    my $d = shift;
    $d->{version} = "1.1.0";
    push @{$d->{features}}, "logging";
});

open $fh, '<', "$tmpdir/config.json";
my $updated = $json->decode(do { local $/; <$fh> });
close $fh;

printf "Updated version: %s\n", $updated->{version};
printf "Features: %s\n", join(", ", @{$updated->{features}});

# Cleanup
unlink "$tmpdir/$_" for qw(config.json events.jsonl);
rmdir $tmpdir;
```

---

## Step 257: Log File Processing

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(strftime);
use List::Util qw(sum max min);

# =====================
# Log file analyzer
# =====================

# Apache Combined Log Format
my @sample_logs = (
    '192.168.1.1 - alice [01/Jan/2024:10:00:01 +0700] "GET /index.html HTTP/1.1" 200 2048 "-" "Mozilla/5.0"',
    '192.168.1.2 - bob   [01/Jan/2024:10:00:02 +0700] "POST /login HTTP/1.1" 302 0 "/login" "Chrome/120"',
    '10.0.0.1   - -     [01/Jan/2024:10:00:03 +0700] "GET /api/data HTTP/1.1" 200 512 "/home" "curl/7.8"',
    '192.168.1.1 - alice [01/Jan/2024:10:00:05 +0700] "GET /dashboard HTTP/1.1" 200 4096 "/home" "Mozilla/5.0"',
    '192.168.1.3 - carol [01/Jan/2024:10:00:07 +0700] "GET /secret HTTP/1.1" 403 256 "-" "Mozilla/5.0"',
    '192.168.1.1 - alice [01/Jan/2024:10:00:10 +0700] "GET /notfound HTTP/1.1" 404 128 "-" "Mozilla/5.0"',
    '10.0.0.2   - -     [01/Jan/2024:10:00:15 +0700] "GET /health HTTP/1.1" 200 32 "-" "HealthCheck/1.0"',
    '192.168.1.4 - -     [01/Jan/2024:10:00:20 +0700] "GET /admin HTTP/1.1" 401 64 "-" "Nikto/2.0"',
);

my $log_re = qr/
    ^(\S+)\s+          # IP
    \S+\s+             # ident
    (\S+)\s+           # user
    \[([^\]]+)\]\s+    # datetime
    "([A-Z]+)\s+       # method
    (\S+)\s+[^"]*"\s+  # path
    (\d+)\s+           # status
    (\d+)\s+           # bytes
    "([^"]*)"\s+       # referer
    "([^"]*)"          # user agent
/x;

my (@entries, %stats);

for my $line (@sample_logs) {
    if ($line =~ $log_re) {
        my $entry = {
            ip      => $1,
            user    => $2,
            dt      => $3,
            method  => $4,
            path    => $5,
            status  => $6,
            bytes   => $7,
            referer => $8,
            ua      => $9,
        };
        push @entries, $entry;
        
        $stats{status}{$entry->{status}}++;
        $stats{method}{$entry->{method}}++;
        $stats{user}{$entry->{user}}++ unless $entry->{user} eq '-';
        $stats{total_bytes} += $entry->{bytes};
        
        # Classify
        push @{$stats{errors}},  $entry if $entry->{status} >= 400;
        push @{$stats{success}}, $entry if $entry->{status} == 200;
    }
}

printf "=== Log Analysis ===\n";
printf "Total requests: %d\n", scalar @entries;
printf "Total bytes:    %d\n", $stats{total_bytes};

printf "\nStatus codes:\n";
printf "  %s: %d\n", $_, $stats{status}{$_} for sort keys %{$stats{status}};

printf "\nMethods:\n";
printf "  %s: %d\n", $_, $stats{method}{$_} for sort keys %{$stats{method}};

printf "\nTop users:\n";
my @sorted_users = sort { $stats{user}{$b} <=> $stats{user}{$a} } keys %{$stats{user}};
printf "  %-10s %d requests\n", $_, $stats{user}{$_} for @sorted_users[0..1];

printf "\nErrors (%d):\n", scalar @{$stats{errors}//[]};
for my $e (@{$stats{errors}//[]}) {
    printf "  %s %s %s %s\n", $e->{ip}, $e->{user}, $e->{status}, $e->{path};
}

# =====================
# Log rotation simulation
# =====================

sub rotate_logs {
    my ($logdir, $max_files) = @_;
    $max_files //= 5;
    
    opendir my $dh, $logdir;
    my @log_files = sort grep { /^app\.\d+\.log$/ } readdir $dh;
    closedir $dh;
    
    # Remove old files beyond max
    while (@log_files > $max_files) {
        my $old = shift @log_files;
        unlink "$logdir/$old";
        printf "Removed old log: $old\n";
    }
    
    # Rename existing
    for my $f (reverse @log_files) {
        $f =~ /app\.(\d+)\.log/;
        my $new_n = $1 + 1;
        rename "$logdir/$f", "$logdir/app.$new_n.log";
    }
    
    # Current becomes .1
    rename "$logdir/app.log", "$logdir/app.1.log" if -f "$logdir/app.log";
}

printf "\nLog rotation simulation done.\n";
```

---

## Step 258: Temp Files and Atomic Writes

```perl
#!/usr/bin/perl
use strict;
use warnings;
use File::Temp qw(tempfile tempdir);
use File::Copy qw(move copy);

# =====================
# Temporary files
# =====================

# Named temp file
my ($tmp_fh, $tmp_path) = tempfile("perl_XXXXXXXX", DIR => "/tmp", SUFFIX => ".txt");
print $tmp_fh "Temporary content\n";
close $tmp_fh;

printf "Temp file: %s\n", $tmp_path;
printf "Exists: %s\n", -f $tmp_path ? "yes" : "no";

# Read back
open my $in, '<', $tmp_path;
print while <$in>;
close $in;

# Auto-cleanup
my ($auto_fh, $auto_path) = tempfile(UNLINK => 1);
printf "Auto cleanup: %s\n", $auto_path;

# Temp dir
my $tmpdir = tempdir("perl_XXXXXXXX", DIR => "/tmp", CLEANUP => 1);
printf "Temp dir: %s\n", $tmpdir;

# Create files in temp dir
for my $i (1..3) {
    open my $fh, '>', "$tmpdir/file$i.txt";
    print $fh "Content $i\n";
    close $fh;
}

opendir my $dh, $tmpdir;
my @tmp_files = grep { !/^\./ } readdir $dh;
closedir $dh;
printf "Files in temp dir: %d\n", scalar @tmp_files;

# =====================
# Atomic file write
# =====================

sub atomic_write {
    my ($path, $content_or_coderef) = @_;
    
    my $dir = do {
        require File::Basename;
        File::Basename::dirname($path);
    };
    
    my ($tmp_fh, $tmp_path) = tempfile(".tmp_XXXXXXXX", DIR => $dir);
    
    eval {
        if (ref $content_or_coderef eq 'CODE') {
            $content_or_coderef->($tmp_fh);
        } else {
            print $tmp_fh $content_or_coderef;
        }
        close $tmp_fh;
        move($tmp_path, $path) or die "Move failed: $!";
    };
    
    if ($@) {
        close $tmp_fh;
        unlink $tmp_path;
        die $@;
    }
}

my $final = "/tmp/perl_atomic_$$.txt";

# Write atomically
atomic_write($final, "Hello, atomic world!\n");
printf "Atomic write: %s\n", -f $final ? "success" : "failed";

# Write with callback
atomic_write($final, sub {
    my $fh = shift;
    print $fh "Line 1\n";
    print $fh "Line 2\n";
    print $fh "Line 3\n";
});

open my $check, '<', $final;
my @lines = <$check>;
close $check;
printf "Lines written: %d\n", scalar @lines;

# Cleanup
unlink $tmp_path if -f $tmp_path;
unlink $final;
```

---

## Step 259: File Watching (Polling)

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# File change detection
# =====================

{
package FileWatcher;

sub new {
    my ($class, @paths) = @_;
    my %state;
    for my $p (@paths) {
        $state{$p} = -e $p ? (stat $p)[9] : undef;  # mtime
    }
    return bless { paths => \@paths, state => \%state, callbacks => {} }, $class;
}

sub on_change {
    my ($self, $event, $cb) = @_;
    push @{$self->{callbacks}{$event}}, $cb;
    return $self;
}

sub _emit {
    my ($self, $event, $path) = @_;
    $_->($path) for @{$self->{callbacks}{$event}//[]};
    $_->($path) for @{$self->{callbacks}{any}//[]};
}

sub check {
    my $self = shift;
    my @changed;
    
    for my $path (@{$self->{paths}}) {
        my $old_mtime = $self->{state}{$path};
        
        if (-e $path) {
            my $new_mtime = (stat $path)[9];
            if (!defined $old_mtime) {
                $self->_emit('created', $path);
                push @changed, { path => $path, event => 'created' };
            } elsif ($new_mtime != $old_mtime) {
                $self->_emit('modified', $path);
                push @changed, { path => $path, event => 'modified' };
            }
            $self->{state}{$path} = $new_mtime;
        } else {
            if (defined $old_mtime) {
                $self->_emit('deleted', $path);
                push @changed, { path => $path, event => 'deleted' };
            }
            $self->{state}{$path} = undef;
        }
    }
    
    return @changed;
}

sub poll {
    my ($self, %opts) = @_;
    my $interval = $opts{interval} // 1;
    my $max      = $opts{max_checks} // 10;
    my $checks   = 0;
    
    while ($checks < $max) {
        $self->check;
        $checks++;
        select undef, undef, undef, $interval;  # sleep
    }
}
}

# Demo
my $dir = "/tmp/perl_watch_$$";
mkdir $dir;

my $watcher = FileWatcher->new(
    "$dir/config.json",
    "$dir/data.txt",
    "$dir/temp.log",
);

my @events;
$watcher->on_change('created',  sub { push @events, "CREATED: $_[0]" });
$watcher->on_change('modified', sub { push @events, "MODIFIED: $_[0]" });
$watcher->on_change('deleted',  sub { push @events, "DELETED: $_[0]" });

# Simulate changes
open my $fh, '>', "$dir/config.json"; print $fh "{}"; close $fh;
$watcher->check;  # Should detect 'created'

open $fh, '>', "$dir/data.txt"; print $fh "data"; close $fh;
sleep 1;
open $fh, '>', "$dir/data.txt"; print $fh "updated"; close $fh;
$watcher->check;  # Should detect 'created' (data.txt) + 'modified' may depend on timing

unlink "$dir/config.json";
$watcher->check;  # Should detect 'deleted'

printf "Events detected:\n";
printf "  %s\n", $_ for @events;

# Cleanup
unlink "$dir/data.txt" if -f "$dir/data.txt";
rmdir $dir;
```

---

## Step 260: โปรแกรมสรุป — File Processing Pipeline

```perl
#!/usr/bin/perl
# file_pipeline.pl — ETL pipeline using files
use strict;
use warnings;
use File::Path qw(make_path remove_tree);
use JSON::PP;
use List::Util qw(sum max min);
use POSIX qw(strftime);

my $work_dir = "/tmp/perl_pipeline_$$";
make_path("$work_dir/input", "$work_dir/output", "$work_dir/archive");

# =====================
# Step 1: Generate sample data files
# =====================

my @products = (
    { sku => "P001", name => "Widget A",    price => 9.99,  qty => 100 },
    { sku => "P002", name => "Widget B",    price => 14.99, qty => 50  },
    { sku => "P003", name => "Gadget Pro",  price => 49.99, qty => 25  },
    { sku => "P004", name => "Accessory",   price => 4.99,  qty => 200 },
);

my @orders = (
    { id => "O001", date => "2024-01-15", items => [["P001",2], ["P003",1]] },
    { id => "O002", date => "2024-01-15", items => [["P002",3], ["P004",5]] },
    { id => "O003", date => "2024-01-16", items => [["P001",1], ["P002",1], ["P004",2]] },
    { id => "O004", date => "2024-01-16", items => [["P003",2]] },
);

# Write products as CSV
open my $fh, '>', "$work_dir/input/products.csv";
print $fh "sku,name,price,qty\n";
printf $fh "%s,\"%s\",%.2f,%d\n", $_->{sku}, $_->{name}, $_->{price}, $_->{qty} for @products;
close $fh;

# Write orders as JSONL
my $jl = JSON::PP->new->utf8;
open $fh, '>', "$work_dir/input/orders.jsonl";
print $fh $jl->encode($_) . "\n" for @orders;
close $fh;

printf "Input files created\n";

# =====================
# Step 2: Load and validate
# =====================

# Load products
my %product_catalog;
open $fh, '<', "$work_dir/input/products.csv";
my $header = <$fh>;  # skip header
while (<$fh>) {
    chomp;
    my ($sku, $name, $price, $qty) = /^(\w+),"?([^",]+)"?,(\d+\.\d+),(\d+)$/;
    $product_catalog{$sku} = { sku => $sku, name => $name, price => $price+0, qty => $qty+0 };
}
close $fh;

printf "Products loaded: %d\n", scalar keys %product_catalog;

# Load orders
my @loaded_orders;
open $fh, '<', "$work_dir/input/orders.jsonl";
while (<$fh>) {
    chomp;
    my $order = $jl->decode($_);
    
    # Validate
    my $valid = 1;
    for my $item (@{$order->{items}}) {
        unless ($product_catalog{$item->[0]}) {
            printf "Invalid product %s in order %s\n", $item->[0], $order->{id};
            $valid = 0;
        }
    }
    
    push @loaded_orders, $order if $valid;
}
close $fh;

printf "Orders loaded: %d\n", scalar @loaded_orders;

# =====================
# Step 3: Process and transform
# =====================

my @processed;
my %daily_totals;

for my $order (@loaded_orders) {
    my $total = 0;
    my @enriched_items;
    
    for my $item (@{$order->{items}}) {
        my ($sku, $qty) = @$item;
        my $prod = $product_catalog{$sku};
        my $line_total = $prod->{price} * $qty;
        $total += $line_total;
        push @enriched_items, {
            sku   => $sku,
            name  => $prod->{name},
            price => $prod->{price},
            qty   => $qty,
            total => $line_total,
        };
    }
    
    push @processed, {
        id    => $order->{id},
        date  => $order->{date},
        items => \@enriched_items,
        total => $total,
        count => scalar @{$order->{items}},
    };
    
    $daily_totals{$order->{date}} += $total;
}

# =====================
# Step 4: Write outputs
# =====================

# Detailed orders JSON
my $json = JSON::PP->new->utf8->canonical->pretty;
open $fh, '>', "$work_dir/output/orders_processed.json";
print $fh $json->encode({ orders => \@processed, generated => strftime("%Y-%m-%dT%H:%M:%S", localtime) });
close $fh;

# Daily summary CSV
open $fh, '>', "$work_dir/output/daily_summary.csv";
print $fh "date,total_orders,total_revenue\n";
my %daily_count;
$daily_count{$_->{date}}++ for @processed;
for my $date (sort keys %daily_totals) {
    printf $fh "%s,%d,%.2f\n", $date, $daily_count{$date}, $daily_totals{$date};
}
close $fh;

# Revenue report
open $fh, '>', "$work_dir/output/report.txt";
printf $fh "=== Sales Report ===\n";
printf $fh "Generated: %s\n\n", strftime("%Y-%m-%d %H:%M:%S", localtime);
printf $fh "Orders processed: %d\n", scalar @processed;
printf $fh "Total revenue: \$%.2f\n\n", sum(map { $_->{total} } @processed);
printf $fh "Daily breakdown:\n";
for my $date (sort keys %daily_totals) {
    printf $fh "  %s: %d orders, \$%.2f\n", $date, $daily_count{$date}, $daily_totals{$date};
}
close $fh;

# =====================
# Step 5: Archive inputs
# =====================

my $ts = strftime("%Y%m%d_%H%M%S", localtime);
for my $f ("products.csv", "orders.jsonl") {
    rename "$work_dir/input/$f", "$work_dir/archive/${ts}_${f}";
}

# =====================
# Verify outputs
# =====================

printf "\n=== Pipeline Complete ===\n";
opendir my $dh, "$work_dir/output";
for my $f (sort grep { !/^\./ } readdir $dh) {
    printf "  Output: %-35s (%d bytes)\n", $f, -s "$work_dir/output/$f";
}
closedir $dh;

printf "\n--- Report preview ---\n";
open $fh, '<', "$work_dir/output/report.txt";
print while <$fh>;
close $fh;

# Cleanup
remove_tree($work_dir, { verbose => 0 });
printf "\nPipeline finished and cleaned up.\n";
```

---

## สรุป Part 26

ใน Part นี้คุณได้เรียนรู้:
- ✅ Modern file handling: open, close, filehandles
- ✅ In-memory filehandles
- ✅ Binary mode and UTF-8 encoding
- ✅ File::Spec and File::Basename
- ✅ Directory operations and File::Find
- ✅ File locking with flock
- ✅ CSV parsing and generation
- ✅ JSON file I/O and JSONL
- ✅ Log file analysis
- ✅ Temporary files and atomic writes
- ✅ File change watching
- ✅ Complete ETL pipeline

**ถัดไป: [Part 27 — Perl Networking](part_27.md)**
