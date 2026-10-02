# Part 10: Control Flow — if, unless, while
## Steps 91-100: การควบคุมการทำงานของโปรแกรม

---

## Step 91: if / elsif / else

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Basic if
# =====================

my $age = 25;

if ($age >= 18) {
    print "ผู้ใหญ่\n";
}

# =====================
# if-else
# =====================

if ($age >= 18) {
    print "ผู้ใหญ่\n";
} else {
    print "ผู้เยาว์\n";
}

# =====================
# if-elsif-else
# =====================

my $score = 75;

if ($score >= 90) {
    print "เกรด A\n";
} elsif ($score >= 80) {
    print "เกรด B\n";
} elsif ($score >= 70) {
    print "เกรด C\n";
} elsif ($score >= 60) {
    print "เกรด D\n";
} else {
    print "เกรด F\n";
}

# =====================
# Postfix if (statement modifier)
# =====================

print "ผู้ใหญ่\n" if $age >= 18;
print "ผู้เยาว์\n" if $age < 18;

my $x = 10;
$x *= 2 if $x < 20;
print "x = $x\n";

# =====================
# Nested if
# =====================

my $temp = 30;
my $raining = 1;

if ($temp > 25) {
    if ($raining) {
        print "ร้อนและฝนตก\n";
    } else {
        print "ร้อนและแดดออก\n";
    }
} else {
    print "ไม่ร้อน\n";
}

# =====================
# Complex conditions
# =====================

my $username = "admin";
my $password = "secret123";
my $is_active = 1;

if ($username eq "admin" && $password eq "secret123" && $is_active) {
    print "เข้าสู่ระบบสำเร็จ\n";
} elsif (!$is_active) {
    print "บัญชีถูกระงับ\n";
} else {
    print "ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง\n";
}

# =====================
# Short-circuit in if
# =====================

my %user;
# ตรวจสอบ key ก่อน เพื่อหลีกเลี่ยง undef warning
if (exists $user{name} && defined $user{name} && length $user{name}) {
    print "Name: $user{name}\n";
}

# หรือใช้ // ช่วย
my $name = $user{name} // "Anonymous";
print "Name: $name\n";
```

---

## Step 92: unless

```perl
#!/usr/bin/perl
use strict;
use warnings;

# unless = if not

my $debug = 0;
unless ($debug) {
    print "Production mode\n";
}

# เหมือนกัน
if (!$debug) {
    print "Production mode\n";
}

# Postfix unless
my $logged_in = 0;
print "Please login\n" unless $logged_in;

# unless-else (แต่ไม่ควรใช้บ่อย — อ่านยาก)
my $error = "";
unless ($error) {
    print "Success\n";
} else {
    print "Error: $error\n";
}

# =====================
# Use cases สำหรับ unless
# =====================

# Guard clauses
sub process {
    my $data = shift;
    
    unless (defined $data) {
        warn "No data provided\n";
        return;
    }
    
    unless (length $data) {
        warn "Empty data\n";
        return;
    }
    
    # ประมวลผล
    print "Processing: $data\n";
}

process("Hello");
process();
process("");
```

---

## Step 93: while และ until

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# while loop
# =====================

my $i = 1;
while ($i <= 5) {
    print "$i ";
    $i++;
}
print "\n";

# =====================
# until loop (while not)
# =====================

$i = 1;
until ($i > 5) {
    print "$i ";
    $i++;
}
print "\n";

# =====================
# do-while
# =====================

$i = 1;
do {
    print "$i ";
    $i++;
} while ($i <= 5);
print "\n";

# =====================
# do-until
# =====================

$i = 1;
do {
    print "$i ";
    $i++;
} until ($i > 5);
print "\n";

# =====================
# Infinite loop
# =====================

my $count = 0;
while (1) {
    $count++;
    last if $count >= 5;  # break
    next if $count == 3;  # skip iteration 3
    print "count: $count\n";
}

# =====================
# while กับ file
# =====================

open(my $fh, '<', '/tmp/test.txt') or die $!;
while (my $line = <$fh>) {
    chomp $line;
    print ">> $line\n";
}
close $fh;

# =====================
# Postfix while
# =====================

$i = 1;
print $i++, " " while $i <= 5;
print "\n";

# =====================
# while กับ regex
# =====================

my $text = "abc123def456ghi789";
my @numbers;
while ($text =~ /(\d+)/g) {
    push @numbers, $1;
}
print "Numbers: @numbers\n";  # 123 456 789
```

---

## Step 94: for และ foreach

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# C-style for loop
# =====================

for (my $i = 0; $i < 5; $i++) {
    print "$i ";
}
print "\n";

# =====================
# foreach loop
# =====================

my @fruits = qw(apple banana cherry date elderberry);

foreach my $fruit (@fruits) {
    print "$fruit\n";
}

# Short form: for (same as foreach)
for my $fruit (@fruits) {
    print "$fruit\n";
}

# =====================
# Default variable $_
# =====================

for (@fruits) {
    print "$_\n";  # $_ is the current element
}

# Modify in loop
for (@fruits) {
    $_ = uc;  # modifies actual element!
}
print "@fruits\n";  # APPLE BANANA CHERRY DATE ELDERBERRY

# =====================
# Index in foreach
# =====================

my @colors = qw(red green blue yellow);

for my $i (0..$#colors) {
    printf "[%d] %s\n", $i, $colors[$i];
}

# With each (Perl 5.12+)
while (my ($idx, $val) = each @colors) {
    printf "[%d] %s\n", $idx, $val;
}

# =====================
# Nested loops
# =====================

for my $i (1..3) {
    for my $j (1..3) {
        printf "%3d", $i * $j;
    }
    print "\n";
}

# =====================
# Loop labels
# =====================

OUTER: for my $i (1..5) {
    for my $j (1..5) {
        if ($i + $j > 7) {
            next OUTER;  # jump to next iteration of outer loop
        }
        print "($i,$j) ";
    }
}
print "\n";

# =====================
# Postfix for
# =====================

print "$_ " for 1..10;
print "\n";

print "$_ " for reverse @colors;
print "\n";

print uc, "\n" for qw(hello world perl);
```

---

## Step 95: last, next, redo

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# last — break loop
# =====================

print "last demo:\n";
for my $i (1..10) {
    last if $i > 5;
    print "$i ";
}
print "\n";

# =====================
# next — skip iteration
# =====================

print "\nnext demo (skip evens):\n";
for my $i (1..10) {
    next if $i % 2 == 0;
    print "$i ";
}
print "\n";

# =====================
# redo — restart current iteration
# =====================

print "\nredo demo:\n";
my $attempts = 0;
for my $i (1..3) {
    $attempts++;
    if ($attempts <= 6 && $i == 2 && $attempts < 5) {
        print "redo $i (attempt $attempts)\n";
        redo;  # restart iteration without changing $i
    }
    print "done $i\n";
}

# =====================
# Loop labels with control
# =====================

print "\nLabeled loop demo:\n";
OUTER: for my $row (1..4) {
    for my $col (1..4) {
        if ($col == 3) {
            next OUTER;  # next iteration of OUTER
        }
        print "($row,$col) ";
    }
}
print "\n";

# Find first match
my @matrix = (
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9],
);

my $target = 5;
my ($found_i, $found_j);

SEARCH: for my $i (0..$#matrix) {
    for my $j (0..$#{$matrix[$i]}) {
        if ($matrix[$i][$j] == $target) {
            ($found_i, $found_j) = ($i, $j);
            last SEARCH;
        }
    }
}

if (defined $found_i) {
    print "Found $target at [$found_i][$found_j]\n";
}

# =====================
# Loop as expression
# =====================

my @evens = grep { $_ % 2 == 0 } 1..20;
print "Evens: @evens\n";

my @squares = map { $_ ** 2 } 1..10;
print "Squares: @squares\n";
```

---

## Step 96: given/when (Switch) และ Smart Match

```perl
#!/usr/bin/perl
use strict;
use warnings;
use feature 'switch';  # requires Perl 5.10+

# =====================
# given/when (เหมือน switch)
# =====================

# หมายเหตุ: given/when ถูก deprecated ใน Perl 5.38
# แต่ยังสามารถใช้ได้ด้วย 'use feature'
# แนะนำให้ใช้ if/elsif แทน

my $day = "Monday";

# if/elsif แทน switch (แนะนำกว่า)
sub day_type {
    my $d = shift;
    
    my %weekday = map { $_ => 1 } qw(Monday Tuesday Wednesday Thursday Friday);
    my %weekend = map { $_ => 1 } qw(Saturday Sunday);
    
    return "weekday" if $weekday{$d};
    return "weekend" if $weekend{$d};
    return "unknown";
}

for my $d (qw(Monday Saturday Wednesday Sunday)) {
    printf "%-12s: %s\n", $d, day_type($d);
}

# =====================
# Dispatch table (alternative to switch)
# =====================

sub handle_command {
    my $cmd = shift;
    
    my %handlers = (
        'start'   => sub { print "Starting...\n" },
        'stop'    => sub { print "Stopping...\n" },
        'status'  => sub { print "Running OK\n" },
        'restart' => sub { 
            print "Restarting...\n";
        },
    );
    
    my $handler = $handlers{lc $cmd} // sub { print "Unknown: $cmd\n" };
    $handler->();
}

handle_command("start");
handle_command("status");
handle_command("unknown");

# =====================
# Multi-value switch
# =====================

sub classify_char {
    my $c = shift;
    
    return "digit"      if $c =~ /\d/;
    return "uppercase"  if $c =~ /[A-Z]/;
    return "lowercase"  if $c =~ /[a-z]/;
    return "whitespace" if $c =~ /\s/;
    return "punctuation" if $c =~ /[[:punct:]]/;
    return "other";
}

for my $c ('a', 'Z', '5', ' ', '!', chr(128)) {
    printf "char '%s': %s\n", $c, classify_char($c);
}
```

---

## Step 97: Conditional Expressions

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Ternary operator
# =====================

my $x = 10;
my $result = $x > 5 ? "big" : "small";
print "$result\n";

# Nested ternary
my $n = 42;
my $label = $n < 0   ? "negative" :
            $n == 0  ? "zero"     :
            $n < 100 ? "small"    :
                       "large";
print "$label\n";

# =====================
# || and // as conditionals
# =====================

my $user = undef;
my $default_user = $user // "Guest";  # Use Guest if $user is undef
print "User: $default_user\n";

my $setting = 0;
my $display = $setting || "default";   # Use "default" if $setting is falsy
print "Setting: $display\n";

# =====================
# and / or for control flow
# =====================

# Old style (still used)
open(my $fh, '<', '/tmp/test.txt') or die "Cannot open: $!\n";
# Same as: if (!open(...)) { die ... }

# =====================
# Chained expressions
# =====================

sub validate_email {
    my $email = shift;
    return $email =~ /^[^\@]+\@[^\@]+\.[^\@]+$/;
}

my @emails = ("valid\@example.com", "invalid", "also\@valid.org", "bad\@");

for my $email (@emails) {
    print "$email: ", validate_email($email) ? "valid" : "invalid", "\n";
}

# =====================
# unless in complex conditions
# =====================

my @items = (1, 2, 3, 4, 5);

# Print unless empty and unless first element is zero
unless (@items == 0 || $items[0] == 0) {
    print "Items: @items\n";
}

# =====================
# Boolean operators return values
# =====================

# || returns first true value or last value
my $a = undef || 0 || "" || "found";
print "a = $a\n";  # found

# && returns first false or last value
my $b = 1 && 2 && 3;
print "b = $b\n";  # 3

# These are used for default values
my %config;
my $host    = $config{host}    || "localhost";
my $timeout = $config{timeout} // 30;   # // for numeric defaults (avoids 0 issue)
```

---

## Step 98: Loop Patterns

```perl
#!/usr/bin/perl
use strict;
use warnings;
use List::Util qw(sum min max first any all none);

# =====================
# Map and Grep (functional loops)
# =====================

my @nums = 1..10;

# grep — filter
my @evens = grep { $_ % 2 == 0 } @nums;
my @odds  = grep { $_ % 2 != 0 } @nums;
print "evens: @evens\n";
print "odds:  @odds\n";

# map — transform
my @squared  = map { $_ ** 2 } @nums;
my @str_nums = map { "item_$_" } @nums;
print "squared: @squared\n";

# Chained
my @result = sort { $a <=> $b }
             grep { $_ > 20 }
             map  { $_ ** 2 }
             1..10;
print "result: @result\n";  # 25 36 49 64 81 100

# =====================
# Reduce
# =====================

use List::Util qw(reduce);

my $total  = reduce { $a + $b } @nums;
my $product = reduce { $a * $b } 1..5;
print "total: $total, product: $product\n";

# =====================
# Short-circuit loops
# =====================

# first — find first match
my $first_even = first { $_ % 2 == 0 } @nums;
print "first even: $first_even\n";

# any — at least one matches
print "any >5: ", (any { $_ > 5 } @nums) ? "yes" : "no", "\n";

# all — all match
print "all >0: ", (all { $_ > 0 } @nums) ? "yes" : "no", "\n";

# none — none match
print "none <0: ", (none { $_ < 0 } @nums) ? "yes" : "no", "\n";

# =====================
# Iteration patterns
# =====================

# Process pairs
my @pairs = ([1,2], [3,4], [5,6]);
for my $pair (@pairs) {
    printf "(%d, %d) => %d\n", $pair->[0], $pair->[1], 
        $pair->[0] * $pair->[1];
}

# Sliding window
my @data = 1..10;
my $window = 3;
for my $i (0..@data-$window) {
    my @window_data = @data[$i..$i+$window-1];
    my $avg = sum(@window_data) / $window;
    printf "Window %d-%d: avg=%.1f\n", $i+1, $i+$window, $avg;
}

# =====================
# Loop with accumulator
# =====================

my @words = qw(the quick brown fox jumps over the lazy dog);
my %letter_freq;

for my $word (@words) {
    for my $letter (split //, $word) {
        $letter_freq{$letter}++;
    }
}

print "\nTop 5 letters:\n";
my @top5 = (sort { $letter_freq{$b} <=> $letter_freq{$a} } keys %letter_freq)[0..4];
printf "  %s: %d\n", $_, $letter_freq{$_} for @top5;
```

---

## Step 99: Control Flow ขั้นสูง

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# State machine
# =====================

use constant {
    STATE_IDLE     => 'idle',
    STATE_RUNNING  => 'running',
    STATE_PAUSED   => 'paused',
    STATE_STOPPED  => 'stopped',
};

my $state = STATE_IDLE;

my %transitions = (
    STATE_IDLE    , { start  => STATE_RUNNING },
    STATE_RUNNING , { pause  => STATE_PAUSED, stop => STATE_STOPPED },
    STATE_PAUSED  , { resume => STATE_RUNNING, stop => STATE_STOPPED },
    STATE_STOPPED , {},
);

sub transition {
    my ($current, $event) = @_;
    my $next = $transitions{$current}{$event};
    unless (defined $next) {
        warn "Invalid transition: $current -> $event\n";
        return $current;
    }
    print "State: $current -> $next (event: $event)\n";
    return $next;
}

$state = transition($state, 'start');   # idle -> running
$state = transition($state, 'pause');   # running -> paused
$state = transition($state, 'resume');  # paused -> running
$state = transition($state, 'stop');    # running -> stopped
$state = transition($state, 'start');   # invalid! (stopped)

# =====================
# Event-driven pattern
# =====================

my %event_handlers;

sub on {
    my ($event, $handler) = @_;
    push @{$event_handlers{$event}}, $handler;
}

sub emit {
    my ($event, @args) = @_;
    for my $handler (@{$event_handlers{$event} // []}) {
        $handler->(@args);
    }
}

on 'login',  sub { print "Handler 1: user '$_[0]' logged in\n" };
on 'login',  sub { print "Handler 2: logging login event\n" };
on 'logout', sub { print "User '$_[0]' logged out\n" };

emit('login',  'Alice');
emit('login',  'Bob');
emit('logout', 'Alice');

# =====================
# Recursive control
# =====================

sub walk_tree {
    my ($node, $depth) = @_;
    $depth //= 0;
    
    print "  " x $depth, $node->{name}, "\n";
    
    if ($node->{children}) {
        walk_tree($_, $depth + 1) for @{$node->{children}};
    }
}

my $tree = {
    name => "root",
    children => [
        { name => "child1", children => [
            { name => "grandchild1" },
            { name => "grandchild2" },
        ]},
        { name => "child2" },
        { name => "child3", children => [
            { name => "grandchild3" },
        ]},
    ],
};

print "\nTree:\n";
walk_tree($tree);
```

---

## Step 100: โปรแกรมสรุป — Task Manager

```perl
#!/usr/bin/perl
#
# โปรแกรม: task_manager.pl
# ระบบจัดการงาน (To-Do List)
#

use strict;
use warnings;
use POSIX qw(strftime);

# =====================
# Data structure
# =====================
my @tasks;
my $next_id = 1;

# =====================
# Task operations
# =====================

sub add_task {
    my (%opts) = @_;
    my $task = {
        id       => $next_id++,
        title    => $opts{title}    // "Untitled",
        priority => $opts{priority} // "medium",
        status   => "pending",
        created  => time(),
        due      => $opts{due},
        tags     => $opts{tags}     // [],
    };
    push @tasks, $task;
    return $task->{id};
}

sub find_task {
    my $id = shift;
    return (grep { $_->{id} == $id } @tasks)[0];
}

sub complete_task {
    my $id = shift;
    my $task = find_task($id) or return;
    $task->{status}     = "completed";
    $task->{completed}  = time();
}

sub delete_task {
    my $id = shift;
    @tasks = grep { $_->{id} != $id } @tasks;
}

sub filter_tasks {
    my (%opts) = @_;
    my @filtered = @tasks;
    
    if ($opts{status}) {
        @filtered = grep { $_->{status} eq $opts{status} } @filtered;
    }
    if ($opts{priority}) {
        @filtered = grep { $_->{priority} eq $opts{priority} } @filtered;
    }
    if ($opts{tag}) {
        @filtered = grep { grep { $_ eq $opts{tag} } @{$_->{tags}} } @filtered;
    }
    
    return @filtered;
}

sub sort_tasks {
    my ($tasks_ref, $by) = @_;
    $by //= 'priority';
    
    my %priority_order = (high => 1, medium => 2, low => 3);
    
    if ($by eq 'priority') {
        return sort {
            ($priority_order{$a->{priority}} // 99) <=>
            ($priority_order{$b->{priority}} // 99) ||
            $a->{created} <=> $b->{created}
        } @$tasks_ref;
    } elsif ($by eq 'created') {
        return sort { $a->{created} <=> $b->{created} } @$tasks_ref;
    } elsif ($by eq 'id') {
        return sort { $a->{id} <=> $b->{id} } @$tasks_ref;
    }
    return @$tasks_ref;
}

# =====================
# Display
# =====================

my %status_symbol = (
    pending   => "[ ]",
    completed => "[X]",
);

my %priority_label = (
    high   => "!!!",
    medium => "!! ",
    low    => "!  ",
);

sub display_task {
    my $task = shift;
    my $sym  = $status_symbol{$task->{status}} // "[?]";
    my $pri  = $priority_label{$task->{priority}} // "   ";
    my $tags = @{$task->{tags}} ? "[" . join(",", @{$task->{tags}}) . "]" : "";
    
    printf "%s %s %3d. %-40s %s\n",
        $sym, $pri, $task->{id}, 
        substr($task->{title}, 0, 40),
        $tags;
}

sub display_list {
    my ($title, @tasks) = @_;
    
    print "\n" . "=" x 60 . "\n";
    print "$title\n";
    print "=" x 60 . "\n";
    
    unless (@tasks) {
        print "  (ไม่มีรายการ)\n";
        return;
    }
    
    display_task($_) for @tasks;
    print "\nรวม: ", scalar @tasks, " รายการ\n";
}

# =====================
# Populate test data
# =====================

add_task(title => "ส่ง report ประจำเดือน", priority => "high",   
         tags  => [qw(work urgent)]);
add_task(title => "ซื้อของใช้ส่วนตัว",     priority => "medium", 
         tags  => [qw(personal shopping)]);
add_task(title => "ออกกำลังกาย",           priority => "low",    
         tags  => [qw(health)]);
add_task(title => "เรียน Perl Chapter 5",   priority => "high",   
         tags  => [qw(learning perl)]);
add_task(title => "โทรหาแม่",              priority => "medium", 
         tags  => [qw(personal family)]);
add_task(title => "fix bug #1234",          priority => "high",   
         tags  => [qw(work bug)]);
add_task(title => "อ่านหนังสือ 30 นาที",   priority => "low",    
         tags  => [qw(personal reading)]);

# =====================
# Main program
# =====================

# แสดงทั้งหมด
display_list("ทุกงาน", sort_tasks(\@tasks, 'priority'));

# เสร็จงาน 2, 4
complete_task(2);
complete_task(4);

# แสดงงานที่ยังค้างอยู่
my @pending = filter_tasks(status => 'pending');
display_list("งานที่ยังค้างอยู่", sort_tasks(\@pending, 'priority'));

# แสดงงาน priority สูง
my @high_priority = filter_tasks(priority => 'high', status => 'pending');
display_list("งานเร่งด่วน (High Priority)", @high_priority);

# สถิติ
print "\n" . "=" x 60 . "\n";
print "สถิติ\n";
print "=" x 60 . "\n";

my %by_status;
$by_status{$_->{status}}++ for @tasks;
printf "%-15s: %d\n", $_, $by_status{$_} // 0 
    for qw(pending completed);

my %by_priority;
$by_priority{$_->{priority}}++ for grep { $_->{status} eq 'pending' } @tasks;
printf "\nงานที่ค้างแยกตาม priority:\n";
printf "  %-10s: %d\n", $_, $by_priority{$_} // 0
    for qw(high medium low);
```

---

## แบบฝึกหัด Part 10

### แบบฝึกหัดที่ 1: ATM Simulation
สร้าง ATM simulator ที่:
- ตรวจสอบ PIN
- ฝาก/ถอน/โอน
- แสดงยอดเงิน

### แบบฝึกหัดที่ 2: Number Guessing Game
เกมทายตัวเลข 1-100 พร้อม hint

### แบบฝึกหัดที่ 3: Menu-Driven Program
โปรแกรมที่มี menu หลัก, submenu และการตรวจสอบ input

---

## สรุป Part 10

ใน Part นี้คุณได้เรียนรู้:
- ✅ if / elsif / else
- ✅ unless
- ✅ while / until
- ✅ do-while / do-until
- ✅ for / foreach
- ✅ last, next, redo
- ✅ Loop labels
- ✅ Postfix conditionals
- ✅ Dispatch tables
- ✅ State machine pattern
- ✅ Event-driven pattern

**ถัดไป: [Part 11 — Loops ขั้นสูง](part_11.md)**
