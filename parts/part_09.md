# Part 09: Input / Output พื้นฐาน
## Steps 81-90: การจัดการ Input และ Output

---

## Step 81: Standard I/O

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# STDIN, STDOUT, STDERR
# =====================

# STDOUT (standard output)
print STDOUT "Hello, STDOUT!\n";
print "Same as above (STDOUT is default)\n";

# STDERR (standard error)
print STDERR "This is an error message\n";
warn "Warning message\n";  # goes to STDERR + newline

# STDIN (standard input)
print "Enter your name: ";
my $name = <STDIN>;
chomp $name;
print "Hello, $name!\n";

# =====================
# Reading multiple lines
# =====================

print "Enter lines (empty line to stop):\n";
my @lines;
while (my $line = <STDIN>) {
    chomp $line;
    last unless $line;  # stop on empty line
    push @lines, $line;
}

print "You entered ", scalar @lines, " lines:\n";
print "  $_\n" for @lines;

# =====================
# Reading with prompt
# =====================

sub prompt {
    my ($msg, $default) = @_;
    print $msg;
    print " [$default]" if defined $default;
    print ": ";
    
    my $input = <STDIN>;
    chomp $input;
    
    return length($input) ? $input : ($default // '');
}

my $host = prompt("Database host", "localhost");
my $port = prompt("Database port", "3306");
my $db   = prompt("Database name");

print "\nConnecting to $host:$port/$db\n";

# =====================
# Input validation loop
# =====================

sub ask {
    my (%opts) = @_;
    my $prompt    = $opts{prompt} or die "need prompt";
    my $validate  = $opts{validate};
    my $error_msg = $opts{error} // "Invalid input";
    my $default   = $opts{default};
    
    while (1) {
        print $prompt;
        print " [$default]" if defined $default;
        print ": ";
        
        my $input = <STDIN>;
        chomp $input;
        $input = $default // '' unless length $input;
        
        if (!$validate || $validate->($input)) {
            return $input;
        }
        print "$error_msg\n";
    }
}

my $age = ask(
    prompt   => "Your age",
    validate => sub { $_[0] =~ /^\d+$/ && $_[0] >= 0 && $_[0] <= 150 },
    error    => "Please enter a valid age (0-150)",
);
print "Age: $age\n";
```

---

## Step 82: File Handling — Open, Read, Close

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Creating a test file
# =====================

# เขียนไฟล์ทดสอบ
open(my $wfh, '>', '/tmp/test.txt') or die "Cannot write: $!";
print $wfh "Line 1: Hello\n";
print $wfh "Line 2: World\n";
print $wfh "Line 3: Perl\n";
close $wfh;

# =====================
# Reading a file
# =====================

# Open for reading
open(my $rfh, '<', '/tmp/test.txt') or die "Cannot read: $!";

# Read line by line
while (my $line = <$rfh>) {
    chomp $line;
    print "Got: $line\n";
}
close $rfh;

# =====================
# Slurp — read entire file
# =====================

open($rfh, '<', '/tmp/test.txt') or die $!;
my @all_lines = <$rfh>;
close $rfh;
chomp @all_lines;
print "\nAll lines: @all_lines\n";

# Slurp into single string
open($rfh, '<', '/tmp/test.txt') or die $!;
my $content;
{
    local $/;  # undefine input record separator
    $content = <$rfh>;
}
close $rfh;
print "Content:\n$content";

# File::Slurp module (easier)
# use File::Slurp;
# my $content = read_file('file.txt');
# my @lines   = read_file('file.txt');

# =====================
# Efficient line-by-line
# =====================

open($rfh, '<', '/tmp/test.txt') or die $!;
my $line_num = 0;
while (<$rfh>) {
    $line_num++;
    chomp;
    printf "%3d: %s\n", $line_num, $_;
}
close $rfh;

# =====================
# Open modes
# =====================

# '<'  — read (default)
# '>'  — write (create/truncate)
# '>>' — append
# '+<' — read/write (existing file)
# '+>' — read/write (create/truncate)
# '-|' — pipe from command
# '|-' — pipe to command

# Append
open(my $afh, '>>', '/tmp/test.txt') or die $!;
print $afh "Line 4: Appended\n";
close $afh;

# =====================
# File test operators
# =====================

my $file = '/tmp/test.txt';

print "exists:      ", (-e $file ? "yes" : "no"), "\n";
print "is file:     ", (-f $file ? "yes" : "no"), "\n";
print "is dir:      ", (-d $file ? "yes" : "no"), "\n";
print "is readable: ", (-r $file ? "yes" : "no"), "\n";
print "is writable: ", (-w $file ? "yes" : "no"), "\n";
print "is executable:", (-x $file ? "yes" : "no"), "\n";
print "size:         ", (-s $file), " bytes\n";
print "last modified:", scalar(localtime(-M $file + time)), "\n";
```

---

## Step 83: File Writing

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Writing different content types
# =====================

# Write text file
sub write_text {
    my ($filename, $content) = @_;
    open(my $fh, '>', $filename) or die "Cannot write $filename: $!";
    print $fh $content;
    close $fh;
    return 1;
}

# Write lines
sub write_lines {
    my ($filename, @lines) = @_;
    open(my $fh, '>', $filename) or die "Cannot write $filename: $!";
    print $fh "$_\n" for @lines;
    close $fh;
    return 1;
}

# Append to file
sub append_text {
    my ($filename, $content) = @_;
    open(my $fh, '>>', $filename) or die "Cannot append $filename: $!";
    print $fh $content;
    close $fh;
    return 1;
}

# Test
write_text('/tmp/output.txt', "Hello, World!\n");
write_lines('/tmp/list.txt', "Apple", "Banana", "Cherry");
append_text('/tmp/output.txt', "More content\n");

# =====================
# CSV writing
# =====================

sub write_csv {
    my ($filename, $headers, @rows) = @_;
    
    open(my $fh, '>', $filename) or die "Cannot write $filename: $!";
    
    # Quote a field if it contains comma, quote, or newline
    my $quote = sub {
        my $f = shift;
        if ($f =~ /[,"\n]/) {
            $f =~ s/"/""/g;
            return qq("$f");
        }
        return $f;
    };
    
    # Write header
    print $fh join(',', map { $quote->($_) } @$headers) . "\n";
    
    # Write rows
    for my $row (@rows) {
        print $fh join(',', map { $quote->($_) } @$row) . "\n";
    }
    
    close $fh;
}

write_csv('/tmp/data.csv',
    [qw(Name Age City)],
    ["Alice", 25, "Bangkok"],
    ["Bob", 30, "New York"],
    ["Charlie", 28, 'London, UK'],
);

# Read back and verify
open(my $fh, '<', '/tmp/data.csv') or die $!;
print "\nCSV content:\n";
while (<$fh>) { print "  $_" }
close $fh;

# =====================
# Binary file writing
# =====================

sub write_binary {
    my ($filename, $data) = @_;
    open(my $fh, '>', $filename) or die $!;
    binmode $fh;  # important for binary data!
    print $fh $data;
    close $fh;
}

# Write a simple PNG-like header (just for demo)
my $binary_data = pack("C*", 0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A);
write_binary('/tmp/test.bin', $binary_data);

print "\nBinary file written (8 bytes)\n";

# =====================
# Atomic write (safe write)
# =====================

sub atomic_write {
    my ($filename, $content) = @_;
    my $tmp = "$filename.tmp.$$";
    
    eval {
        open(my $fh, '>', $tmp) or die "Cannot write temp: $!";
        print $fh $content;
        close $fh;
        rename($tmp, $filename) or die "Cannot rename: $!";
    };
    
    if ($@) {
        unlink $tmp if -e $tmp;
        die $@;
    }
    return 1;
}

atomic_write('/tmp/safe_output.txt', "Safely written content\n");
print "Atomic write successful\n";
```

---

## Step 84: File Navigation

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Directory operations
# =====================

use Cwd qw(cwd abs_path);
use File::Basename qw(basename dirname);
use File::Path qw(make_path remove_tree);
use File::Copy qw(copy move);
use File::Spec;

# Current directory
print "Current dir: ", cwd(), "\n";

# =====================
# Path manipulation
# =====================

my $path = "/home/user/documents/report.pdf";

print "basename: ", basename($path), "\n";          # report.pdf
print "dirname:  ", dirname($path), "\n";           # /home/user/documents
print "basename no ext: ", basename($path, '.pdf'), "\n";  # report

# File::Spec — cross-platform paths
my $joined = File::Spec->catfile('/home', 'user', 'docs', 'file.txt');
print "joined: $joined\n";

my @parts = File::Spec->splitpath($joined);
print "volume: $parts[0]\n";  # (empty on Unix)
print "dirs:   $parts[1]\n";
print "file:   $parts[2]\n";

# =====================
# Glob — list files
# =====================

# List all .pl files
my @pl_files = glob("/tmp/*.txt");
print "\n.txt files in /tmp:\n";
print "  $_\n" for @pl_files;

# Wildcard patterns
my @any_files = glob("/tmp/test*");
print "\ntest* files:\n";
print "  $_\n" for @any_files;

# =====================
# Directory reading
# =====================

sub list_dir {
    my ($dir, %opts) = @_;
    my $recursive = $opts{recursive} // 0;
    my $pattern   = $opts{pattern};
    my $type      = $opts{type} // 'all';  # 'all', 'file', 'dir'
    
    opendir(my $dh, $dir) or die "Cannot open dir $dir: $!";
    my @entries = grep { !/^\.\.?$/ } readdir $dh;
    closedir $dh;
    
    my @result;
    for my $entry (sort @entries) {
        my $full = File::Spec->catfile($dir, $entry);
        
        next if $pattern && $entry !~ /$pattern/;
        next if $type eq 'file' && -d $full;
        next if $type eq 'dir'  && !-d $full;
        
        push @result, $full;
        
        if ($recursive && -d $full) {
            push @result, list_dir($full, %opts);
        }
    }
    return @result;
}

print "\n/tmp contents:\n";
print "  $_\n" for list_dir('/tmp');

# =====================
# File stats
# =====================

sub file_info {
    my $path = shift;
    return unless -e $path;
    
    my @stat = stat($path);
    return {
        path    => $path,
        size    => $stat[7],
        mode    => $stat[2],
        mtime   => $stat[9],
        is_file => -f $path,
        is_dir  => -d $path,
    };
}

my $info = file_info('/tmp/test.txt');
if ($info) {
    printf "Size:     %d bytes\n", $info->{size};
    printf "Modified: %s\n", scalar localtime($info->{mtime});
    printf "Type:     %s\n", $info->{is_dir} ? "directory" : "file";
}
```

---

## Step 85: File Processing Patterns

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# In-place file editing
# =====================

# สร้างไฟล์ทดสอบ
open(my $fh, '>', '/tmp/edit_test.txt') or die $!;
print $fh "Hello World\n";
print $fh "foo bar baz\n";
print $fh "test line 3\n";
close $fh;

# Edit in-place (safe version)
sub edit_file {
    my ($filename, $transform) = @_;
    
    open(my $in, '<', $filename)           or die "Cannot read $filename: $!";
    open(my $out, '>', "$filename.new")    or die "Cannot write temp: $!";
    
    while (<$in>) {
        $_ = $transform->($_);
        print $out $_;
    }
    
    close $in;
    close $out;
    
    rename("$filename.new", $filename) or die "Cannot rename: $!";
}

edit_file('/tmp/edit_test.txt', sub {
    my $line = shift;
    $line =~ s/\b(\w)/\u$1/g;  # Title case each word
    return $line;
});

# Read and show result
open($fh, '<', '/tmp/edit_test.txt') or die $!;
print "After edit:\n";
print while <$fh>;
close $fh;

# =====================
# Two-file merge
# =====================

sub merge_files {
    my ($file1, $file2, $output) = @_;
    
    open(my $f1, '<', $file1) or die "Cannot read $file1: $!";
    open(my $f2, '<', $file2) or die "Cannot read $file2: $!";
    open(my $out, '>', $output) or die "Cannot write $output: $!";
    
    while (defined(my $l1 = <$f1>) || defined(my $l2 = <$f2>)) {
        print $out $l1 if defined $l1;
        print $out $l2 if defined $l2;
    }
    
    close $_ for $f1, $f2, $out;
}

# =====================
# Count matching lines
# =====================

sub grep_file {
    my ($pattern, $filename) = @_;
    my (@matches, $line_num);
    
    open(my $fh2, '<', $filename) or die "Cannot read $filename: $!";
    while (<$fh2>) {
        $line_num++;
        if (/$pattern/) {
            push @matches, { line => $line_num, text => $_ };
        }
    }
    close $fh2;
    
    return @matches;
}

my @found = grep_file(qr/\w+/, '/tmp/edit_test.txt');
print "\nLines matching \\w+:\n";
printf "  %3d: %s", $_->{line}, $_->{text} for @found;

# =====================
# File comparison
# =====================

sub files_equal {
    my ($f1, $f2) = @_;
    
    return 0 unless -f $f1 && -f $f2;
    return 0 unless -s $f1 == -s $f2;
    
    use Digest::MD5;
    
    my $md5 = Digest::MD5->new;
    
    open my $fh1, '<', $f1 or die $!;
    binmode $fh1;
    $md5->addfile($fh1);
    my $hash1 = $md5->hexdigest;
    close $fh1;
    
    $md5->reset;
    open my $fh2, '<', $f2 or die $!;
    binmode $fh2;
    $md5->addfile($fh2);
    my $hash2 = $md5->hexdigest;
    close $fh2;
    
    return $hash1 eq $hash2;
}
```

---

## Step 86: I/O Redirection

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Redirect STDOUT to file
# =====================

# Save and redirect STDOUT
open(my $old_stdout, '>&', STDOUT) or die "Cannot dup STDOUT: $!";
open(STDOUT, '>', '/tmp/captured.txt') or die "Cannot redirect: $!";

print "This goes to file\n";
print "So does this\n";

# Restore STDOUT
open(STDOUT, '>&', $old_stdout) or die "Cannot restore: $!";
print "This goes to terminal again\n";

# Read back
open(my $fh, '<', '/tmp/captured.txt') or die $!;
print "Captured:\n";
print while <$fh>;
close $fh;

# =====================
# Capture output
# =====================

sub capture_output {
    my $code = shift;
    
    open(my $old, '>&', STDOUT) or die $!;
    
    my $captured = '';
    open(STDOUT, '>', \$captured) or die "Cannot redirect to scalar: $!";
    
    $code->();
    
    open(STDOUT, '>&', $old) or die $!;
    
    return $captured;
}

my $output = capture_output(sub {
    print "Line 1\n";
    print "Line 2\n";
    printf "%05d\n", 42;
});

print "Captured output:\n$output";

# =====================
# Pipe operations
# =====================

# Open pipe to command
open(my $cmd_fh, '-|', 'ls -la /tmp') or die "Cannot run ls: $!";
my @ls_output = <$cmd_fh>;
close $cmd_fh;
print "\nFiles in /tmp:\n";
print for @ls_output[0..4];  # first 5 lines

# Open pipe from command
open(my $grep_fh, '-|', 'echo "hello world"') or die $!;
my $echo_out = <$grep_fh>;
close $grep_fh;
chomp $echo_out;
print "\nEcho: $echo_out\n";

# Bidirectional pipe
use IPC::Open2;
my ($child_out, $child_in);
my $pid = open2($child_out, $child_in, 'cat');
print $child_in "Hello from Perl\n";
close $child_in;
my $response = <$child_out>;
chomp $response;
print "Response: $response\n";
waitpid($pid, 0);

# =====================
# In-memory files
# =====================

# Open filehandle to a scalar variable
my $buffer = '';
open(my $mem_fh, '>', \$buffer) or die $!;
print $mem_fh "Line 1\n";
print $mem_fh "Line 2\n";
printf $mem_fh "Value: %d\n", 42;
close $mem_fh;

print "\nIn-memory buffer:\n$buffer";

# Read from scalar
open(my $mem_read, '<', \$buffer) or die $!;
while (<$mem_read>) {
    chomp;
    print "  Read: $_\n";
}
close $mem_read;
```

---

## Step 87: Format Output (Perl Formats)

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# printf tables
# =====================

my @employees = (
    ["Alice",   "Engineering", 90000, "Senior"],
    ["Bob",     "Marketing",   65000, "Junior"],
    ["Charlie", "Engineering", 85000, "Senior"],
    ["Diana",   "HR",          70000, "Manager"],
    ["Eve",     "Finance",     78000, "Senior"],
);

# Header
printf "%-12s %-15s %10s %-10s\n", "Name", "Department", "Salary", "Level";
printf "%s\n", "-" x 52;

# Rows
for my $emp (sort { $a->[0] cmp $b->[0] } @employees) {
    printf "%-12s %-15s %10s %-10s\n", @$emp[0..2], 
        "\$" . format_num($emp->[2]), $emp->[3];
}

printf "%s\n", "-" x 52;

# Totals
my $total = 0;
$total += $_->[2] for @employees;
printf "%-12s %-15s %10s\n", "Total", "", "\$" . format_num($total);
printf "%-12s %-15s %10s\n", "Average", "", 
    "\$" . format_num(int($total/@employees));

sub format_num {
    my $n = shift;
    1 while $n =~ s/(\d+)(\d{3})/$1,$2/;
    return $n;
}

# =====================
# Report with sections
# =====================

sub print_report {
    my ($title, $data, $columns) = @_;
    
    my $width = 0;
    $width += $_->[1] + 2 for @$columns;
    
    print "\n" . "=" x $width . "\n";
    printf "%-${width}s\n", " $title";
    print "=" x $width . "\n";
    
    # Header
    for my $col (@$columns) {
        printf "%-$col->[1]s  ", $col->[0];
    }
    print "\n";
    print "-" x $width . "\n";
    
    # Rows
    for my $row (@$data) {
        for my $i (0..$#$columns) {
            my $val = $row->[$i] // '';
            my $w = $columns->[$i][1];
            if ($columns->[$i][2] eq 'right') {
                printf "%${w}s  ", $val;
            } else {
                printf "%-${w}s  ", $val;
            }
        }
        print "\n";
    }
    
    print "=" x $width . "\n";
}

print_report(
    "EMPLOYEE REPORT",
    [
        ["Alice",   25, "Engineering", 90000],
        ["Bob",     30, "Marketing",   65000],
        ["Charlie", 28, "Engineering", 85000],
    ],
    [
        ["Name",       10, 'left'],
        ["Age",         5, 'right'],
        ["Department", 15, 'left'],
        ["Salary",     10, 'right'],
    ]
);
```

---

## Step 88: Error Handling ใน I/O

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Basic error handling
# =====================

# die on error
sub safe_open {
    my ($file, $mode) = @_;
    $mode //= '<';
    
    open(my $fh, $mode, $file)
        or die "Cannot open '$file' (mode '$mode'): $!\n";
    
    return $fh;
}

# warn on error
sub try_open {
    my ($file, $mode) = @_;
    $mode //= '<';
    
    open(my $fh, $mode, $file)
        or do {
            warn "Cannot open '$file': $!\n";
            return undef;
        };
    
    return $fh;
}

# eval for catching die
eval {
    my $fh = safe_open('/nonexistent/file.txt');
};
if ($@) {
    print "Caught error: $@";
}

# =====================
# Retry logic
# =====================

sub open_with_retry {
    my ($file, %opts) = @_;
    my $mode     = $opts{mode}    // '<';
    my $retries  = $opts{retries} // 3;
    my $delay    = $opts{delay}   // 1;
    
    for my $attempt (1..$retries) {
        if (open(my $fh, $mode, $file)) {
            return $fh;
        }
        
        if ($attempt < $retries) {
            warn "Attempt $attempt failed, retrying in ${delay}s...\n";
            sleep $delay;
        }
    }
    
    die "Failed to open '$file' after $retries attempts: $!\n";
}

# =====================
# Cleanup on failure
# =====================

sub process_file {
    my ($input, $output) = @_;
    
    my ($in_fh, $out_fh);
    
    eval {
        open($in_fh, '<', $input)  or die "Cannot read $input: $!";
        open($out_fh, '>', $output) or die "Cannot write $output: $!";
        
        while (<$in_fh>) {
            s/old/new/g;
            print $out_fh $_;
        }
    };
    
    # Always close files
    close $in_fh  if $in_fh;
    close $out_fh if $out_fh;
    
    if ($@) {
        unlink $output if -e $output;
        die $@;
    }
}

# Create test file
open(my $fh, '>', '/tmp/process_test.txt') or die $!;
print $fh "old text here\n";
close $fh;

eval { process_file('/tmp/process_test.txt', '/tmp/process_out.txt') };
if ($@) {
    print "Error: $@";
} else {
    print "Processed successfully\n";
    open($fh, '<', '/tmp/process_out.txt') or die $!;
    print while <$fh>;
    close $fh;
}

# =====================
# $! and error codes
# =====================

use POSIX qw(:errno_h);

open(my $fh2, '<', '/nonexistent') or do {
    print "Error: $!\n";           # human-readable
    print "Error num: $!\n";       # same (stringified)
    print "Error code: ${!}\n";    # number in numeric context
    
    if ($! == ENOENT) {
        print "File not found\n";
    } elsif ($! == EACCES) {
        print "Permission denied\n";
    }
};
```

---

## Step 89: Command Line Arguments

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# @ARGV
# =====================

# รัน: perl script.pl arg1 arg2 arg3
# @ARGV = ("arg1", "arg2", "arg3")

print "Arguments: @ARGV\n";
print "Count: ", scalar @ARGV, "\n";

# =====================
# Simple argument parsing
# =====================

my %opts;
my @args;

while (@ARGV) {
    my $arg = shift @ARGV;
    
    if ($arg =~ /^--(\w+)=(.*)$/) {
        $opts{$1} = $2;         # --key=value
    } elsif ($arg =~ /^--(\w+)$/) {
        $opts{$1} = shift @ARGV; # --key value
    } elsif ($arg =~ /^-(\w+)$/) {
        $opts{$1} = 1;          # -flag
    } else {
        push @args, $arg;        # positional arg
    }
}

print "Options:\n";
for my $k (sort keys %opts) {
    print "  --$k = $opts{$k}\n";
}
print "Args: @args\n";

# =====================
# Getopt::Long
# =====================

use Getopt::Long;

my ($verbose, $help, $output, $input, $count);

GetOptions(
    'verbose|v' => \$verbose,
    'help|h'    => \$help,
    'output=s'  => \$output,
    'input=s'   => \$input,
    'count=i'   => \$count,
) or die "Usage error\n";

if ($help) {
    print <<'HELP';
Usage: script.pl [options]

Options:
  --verbose, -v    Verbose output
  --help, -h       Show this help
  --output=FILE    Output file
  --input=FILE     Input file
  --count=N        Count
HELP
    exit;
}

print "verbose: ", $verbose ? "yes" : "no", "\n" if defined $verbose;
print "output: $output\n" if $output;

# =====================
# STDIN or file arguments
# =====================

# Standard Unix pattern: accept files or stdin
my @input_lines;

if (@ARGV) {
    for my $file (@ARGV) {
        open(my $fh, '<', $file) or die "Cannot read $file: $!";
        push @input_lines, <$fh>;
        close $fh;
    }
} else {
    @input_lines = <STDIN>;
}

print "Total lines: ", scalar @input_lines, "\n";
```

---

## Step 90: โปรแกรมสรุป — File Manager

```perl
#!/usr/bin/perl
#
# โปรแกรม: file_manager.pl
# ระบบจัดการไฟล์พื้นฐาน
#

use strict;
use warnings;
use File::Basename qw(basename dirname);
use File::Copy qw(copy move);
use File::Path qw(make_path remove_tree);
use File::Find qw(find);
use Cwd qw(cwd abs_path);
use POSIX qw(strftime);

# =====================
# File info
# =====================

sub get_file_info {
    my $path = shift;
    return undef unless -e $path;
    
    my @stat = stat($path);
    my $size = $stat[7];
    
    my $size_str;
    if    ($size >= 1024**3) { $size_str = sprintf "%.1fG", $size/1024**3 }
    elsif ($size >= 1024**2) { $size_str = sprintf "%.1fM", $size/1024**2 }
    elsif ($size >= 1024)    { $size_str = sprintf "%.1fK", $size/1024 }
    else                     { $size_str = "${size}B" }
    
    return {
        path     => abs_path($path),
        name     => basename($path),
        dir      => dirname($path),
        size     => $stat[7],
        size_str => $size_str,
        mtime    => $stat[9],
        is_file  => -f $path,
        is_dir   => -d $path,
        readable => -r $path,
        writable => -w $path,
    };
}

# =====================
# List directory
# =====================

sub list_directory {
    my ($dir, %opts) = @_;
    my $show_hidden = $opts{hidden} // 0;
    my $sort_by     = $opts{sort}   // 'name';
    
    opendir(my $dh, $dir) or die "Cannot open dir: $!";
    my @entries = readdir $dh;
    closedir $dh;
    
    @entries = grep { !/^\./ } @entries unless $show_hidden;
    
    my @items;
    for my $entry (sort @entries) {
        my $full = "$dir/$entry";
        my $info = get_file_info($full);
        push @items, $info if $info;
    }
    
    # Sort
    if ($sort_by eq 'size') {
        @items = sort { $a->{size} <=> $b->{size} } @items;
    } elsif ($sort_by eq 'time') {
        @items = sort { $b->{mtime} <=> $a->{mtime} } @items;
    } else {
        @items = sort { $a->{name} cmp $b->{name} } @items;
    }
    
    return @items;
}

# =====================
# Find files
# =====================

sub find_files {
    my ($dir, %opts) = @_;
    my $pattern = $opts{pattern};
    my $min_size = $opts{min_size};
    my $max_size = $opts{max_size};
    my $ext     = $opts{ext};
    
    my @found;
    
    find(sub {
        return unless -f $_;
        
        my $name = $File::Find::name;
        
        return if $pattern && $name !~ /$pattern/i;
        return if $ext && $name !~ /\.$ext$/i;
        
        my $size = -s $_;
        return if $min_size && $size < $min_size;
        return if $max_size && $size > $max_size;
        
        push @found, $name;
    }, $dir);
    
    return @found;
}

# =====================
# Main program
# =====================

my $dir = '/tmp';

print "=" x 60 . "\n";
print "FILE MANAGER - $dir\n";
print "=" x 60 . "\n\n";

# List directory
printf "%-5s %-30s %8s %s\n", "Type", "Name", "Size", "Modified";
print "-" x 60 . "\n";

for my $item (list_directory($dir)) {
    my $type = $item->{is_dir} ? "[DIR]" : "[FIL]";
    my $time = strftime("%Y-%m-%d %H:%M", localtime($item->{mtime}));
    
    printf "%-5s %-30s %8s %s\n",
        $type,
        substr($item->{name}, 0, 30),
        $item->{size_str},
        $time;
}

print "\n";

# Statistics
my @items = list_directory($dir);
my @files = grep { $_->{is_file} } @items;
my @dirs  = grep { $_->{is_dir} }  @items;

printf "Total: %d items (%d files, %d dirs)\n",
    scalar @items, scalar @files, scalar @dirs;

if (@files) {
    my $total_size = 0;
    $total_size += $_->{size} for @files;
    printf "Total file size: %d bytes\n", $total_size;
}
```

---

## แบบฝึกหัด Part 09

### แบบฝึกหัดที่ 1: CSV Processor
อ่าน CSV ไฟล์, ประมวลผล, เขียนผลลัพธ์

### แบบฝึกหัดที่ 2: Log Rotator
สร้างโปรแกรมที่หมุน log files

### แบบฝึกหัดที่ 3: Config File
อ่านและเขียน config file แบบ INI

---

## สรุป Part 09

ใน Part นี้คุณได้เรียนรู้:
- ✅ STDIN, STDOUT, STDERR
- ✅ การเปิด/อ่าน/เขียน/ปิดไฟล์
- ✅ File modes: read, write, append
- ✅ File test operators
- ✅ Directory operations
- ✅ I/O redirection
- ✅ Pipe operations
- ✅ Error handling ใน I/O
- ✅ Command line arguments

**ถัดไป: [Part 10 — Control Flow](part_10.md)**
