# Part 21: References และ Complex Data Structures
## Steps 201-210: ระดับ Intermediate — References เชิงลึก

---

## Step 201: Reference พื้นฐาน

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Scalar references
# =====================

my $x   = 42;
my $ref = \$x;          # สร้าง reference

printf "Value:     %d\n",   $x;
printf "Ref addr:  %s\n",   $ref;     # SCALAR(0x...)
printf "Deref:     %d\n",   $$ref;    # dereference
printf "Deref alt: %d\n",   ${$ref};  # แบบชัดเจน

$$ref = 100;            # แก้ผ่าน reference
printf "Updated x: %d\n",   $x;       # x เปลี่ยนด้วย!

# ref() ตรวจ type
printf "Type: %s\n", ref($ref);       # SCALAR

# =====================
# Array references
# =====================

my @arr    = (1, 2, 3, 4, 5);
my $aref   = \@arr;

printf "\nArray ref: %s\n", $aref;    # ARRAY(0x...)
printf "Length: %d\n",  scalar @$aref;
printf "First:  %d\n",  $aref->[0];  # arrow notation
printf "Last:   %d\n",  $aref->[-1];

# Loop
printf "Elements: %s\n", join(", ", @$aref);

# Modify
push @$aref, 6;
printf "After push: %s\n", join(", ", @arr);

# Anonymous array
my $anon = [10, 20, 30];
printf "Anon: %s\n", ref($anon);      # ARRAY
printf "Anon[1]: %d\n", $anon->[1];

# =====================
# Hash references
# =====================

my %h    = (name => "Alice", age => 28, city => "Bangkok");
my $href = \%h;

printf "\nHash ref: %s\n", $href;      # HASH(0x...)
printf "Name: %s\n", $href->{name};
printf "Age:  %d\n", $$href{age};

# Modify
$href->{email} = "alice\@example.com";
printf "Keys: %s\n", join(", ", sort keys %$href);

# Anonymous hash
my $person = { name => "Bob", age => 35 };
printf "Anon hash: %s => %d\n", $person->{name}, $person->{age};

# =====================
# Code references
# =====================

my $add = sub { $_[0] + $_[1] };
my $mul = sub { $_[0] * $_[1] };

printf "\nCode ref: %s\n", ref($add);  # CODE
printf "add(3,4): %d\n", $add->(3, 4);
printf "mul(3,4): %d\n", $mul->(3, 4);

my @ops = ($add, $mul, sub { $_[0] - $_[1] });
printf "ops[0](5,2): %d\n", $ops[0]->(5, 2);

# =====================
# Nested structures
# =====================

my %config = (
    database => {
        host => "localhost",
        port => 5432,
        name => "mydb",
    },
    features => ["auth", "api", "admin"],
    limits => {
        max_connections => 100,
        timeout         => 30,
    },
);

printf "\nDB host: %s\n",   $config{database}{host};
printf "DB port: %d\n",     $config{database}{port};
printf "Feature: %s\n",     $config{features}[0];
printf "Max conn: %d\n",    $config{limits}{max_connections};

# Deep access
my $all = \%config;
printf "Via ref: %s\n", $all->{database}{name};
printf "Feature2: %s\n", $all->{features}[1];

# =====================
# ref() type checking
# =====================

my @tests = (
    \42,
    [1,2,3],
    {a => 1},
    sub { 1 },
    \*STDOUT,
);

for my $r (@tests) {
    printf "ref(%s) = %s\n", "$r", ref($r);
}
```

---

## Step 202: Array of Hashes

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Array of Hashes — the most common pattern
# =====================

my @students = (
    { name => "Alice",   score => 92, grade => "A", subject => "Perl"   },
    { name => "Bob",     score => 78, grade => "B", subject => "Python" },
    { name => "Carol",   score => 85, grade => "B", subject => "Perl"   },
    { name => "Dave",    score => 95, grade => "A", subject => "Python" },
    { name => "Eve",     score => 70, grade => "C", subject => "Perl"   },
    { name => "Frank",   score => 88, grade => "B", subject => "Perl"   },
    { name => "Grace",   score => 96, grade => "A", subject => "Python" },
);

# =====================
# Accessing
# =====================

printf "First: %s (score=%d)\n", $students[0]{name}, $students[0]{score};
printf "Last:  %s (grade=%s)\n", $students[-1]{name}, $students[-1]{grade};

# Dereference alternative
my $s = $students[0];
printf "Same:  %s\n", $s->{name};

# =====================
# Sorting
# =====================

my @by_score = sort { $b->{score} <=> $a->{score} } @students;
print "\nTop students by score:\n";
printf "  %d. %-8s %3d (%s)\n", $i+1, $by_score[$i]{name}, $by_score[$i]{score}, $by_score[$i]{grade}
    for (0..$#by_score);

my @by_name  = sort { $a->{name} cmp $b->{name} } @students;
printf "\nAlphabetical: %s\n", join(", ", map { $_->{name} } @by_name);

# Multi-key sort: subject then score desc
my @multi = sort {
    $a->{subject} cmp $b->{subject} || $b->{score} <=> $a->{score}
} @students;
print "\nBy subject, then score:\n";
printf "  %-8s %-8s %d\n", $_->{subject}, $_->{name}, $_->{score} for @multi;

# =====================
# Filtering
# =====================

my @perl_students = grep { $_->{subject} eq "Perl" } @students;
printf "\nPerl students: %s\n", join(", ", map { $_->{name} } @perl_students);

my @a_students = grep { $_->{grade} eq "A" } @students;
printf "A students:   %s\n", join(", ", map { $_->{name} } @a_students);

my @high_scorers = grep { $_->{score} >= 85 } @students;
printf "Score >= 85:  %d students\n", scalar @high_scorers;

# =====================
# Mapping / Transforming
# =====================

my @names  = map { $_->{name} } @students;
my @scores = map { $_->{score} } @students;

use List::Util qw(sum min max);
my $avg = sum(@scores) / @scores;
printf "\nScore stats: min=%d max=%d avg=%.1f\n", min(@scores), max(@scores), $avg;

# Transform — add 'pass' field
my @with_pass = map {
    { %$_, pass => $_->{score} >= 75 ? 1 : 0 }
} @students;

printf "\nPassing students: %d/%d\n",
    scalar(grep { $_->{pass} } @with_pass),
    scalar @with_pass;

# =====================
# Grouping
# =====================

my %by_grade;
push @{$by_grade{$_->{grade}}}, $_ for @students;

for my $grade (sort keys %by_grade) {
    my @group = @{$by_grade{$grade}};
    printf "Grade %s: %s\n", $grade, join(", ", map { $_->{name} } @group);
}

my %by_subject;
push @{$by_subject{$_->{subject}}}, $_ for @students;

for my $subj (sort keys %by_subject) {
    my @group = @{$by_subject{$subj}};
    my $avg_s = sum(map { $_->{score} } @group) / @group;
    printf "%-8s: %d students, avg score %.1f\n", $subj, scalar @group, $avg_s;
}

# =====================
# Adding / Removing fields
# =====================

for my $s (@students) {
    $s->{rank}  = $s->{score} >= 90 ? "Honor"  :
                  $s->{score} >= 80 ? "Merit"  : "Pass";
    $s->{bonus} = $s->{score} - 70;
}

printf "\nWith ranks:\n";
printf "  %-8s %3d %-6s +%d\n", $_->{name}, $_->{score}, $_->{rank}, $_->{bonus}
    for @students;
```

---

## Step 203: Hash of Arrays

```perl
#!/usr/bin/perl
use strict;
use warnings;
use List::Util qw(sum max min);

# =====================
# Hash of Arrays
# =====================

my %school = (
    "Perl"    => ["Alice", "Carol", "Eve", "Frank"],
    "Python"  => ["Bob", "Dave", "Grace"],
    "Ruby"    => ["Hank", "Ivy"],
    "Go"      => ["Jack", "Kate", "Leo", "Mona"],
);

# Access
printf "Perl class: %s\n", join(", ", @{$school{Perl}});
printf "First Python: %s\n", $school{Python}[0];
printf "All Go students: %d\n", scalar @{$school{Go}};

# Iterate
for my $course (sort keys %school) {
    printf "%-10s (%d): %s\n",
        $course,
        scalar @{$school{$course}},
        join(", ", @{$school{$course}});
}

# =====================
# Building up
# =====================

my %tags;
my @articles = (
    { title => "Perl CGI Guide",    tags => ["perl", "web", "cgi"] },
    { title => "Database with DBI", tags => ["perl", "database", "dbi"] },
    { title => "Python Web",        tags => ["python", "web"] },
    { title => "Perl Regex",        tags => ["perl", "regex"] },
    { title => "SQL Basics",        tags => ["database", "sql"] },
);

for my $article (@articles) {
    push @{$tags{$_}}, $article->{title} for @{$article->{tags}};
}

print "\nArticles by tag:\n";
for my $tag (sort keys %tags) {
    printf "  [%s]: %s\n", $tag, join(" | ", @{$tags{$tag}});
}

# =====================
# Multi-week schedule
# =====================

my %schedule = (
    Mon => ["Stand-up 9am", "Code review 2pm"],
    Tue => ["Sprint planning 10am"],
    Wed => ["Stand-up 9am", "1-on-1 3pm", "Demo 4pm"],
    Thu => ["Stand-up 9am"],
    Fri => ["Stand-up 9am", "Retrospective 3pm", "Team lunch 12pm"],
);

print "\nWeek schedule:\n";
for my $day (qw(Mon Tue Wed Thu Fri)) {
    my @events = @{$schedule{$day} // []};
    if (@events) {
        printf "  %s: %s\n", $day, join(", ", @events);
    } else {
        printf "  %s: (free)\n", $day;
    }
}

# Count events
my $total_events = sum(map { scalar @{$schedule{$_}} } keys %schedule);
printf "Total events this week: %d\n", $total_events;

# Find busiest day
my ($busiest) = sort { scalar @{$schedule{$b}} <=> scalar @{$schedule{$a}} } keys %schedule;
printf "Busiest day: %s (%d events)\n", $busiest, scalar @{$schedule{$busiest}};

# =====================
# Inverted index
# =====================

my @documents = (
    "perl is a powerful programming language",
    "python is easy to learn for beginners",
    "perl and python are both scripting languages",
    "web development with perl cgi is classic",
    "perl has great text processing capabilities",
);

my %index;
for my $i (0..$#documents) {
    for my $word (split /\s+/, $documents[$i]) {
        $word =~ s/[^a-z]//g;
        push @{$index{$word}}, $i unless grep { $_ == $i } @{$index{$word} // []};
    }
}

my @search_terms = ("perl", "python", "scripting");
print "\nSearch index:\n";
for my $term (@search_terms) {
    my @docs = @{$index{$term} // []};
    printf "  '%s': found in %d docs\n", $term, scalar @docs;
    printf "    -> \"%s\"\n", $documents[$_] for @docs;
}
```

---

## Step 204: Hash of Hashes

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Hash of Hashes
# =====================

my %employees = (
    E001 => { name => "Alice Smith",   dept => "Engineering", salary => 85000, level => "Senior" },
    E002 => { name => "Bob Jones",     dept => "Marketing",   salary => 72000, level => "Mid"    },
    E003 => { name => "Carol Davis",   dept => "Engineering", salary => 92000, level => "Lead"   },
    E004 => { name => "Dave Wilson",   dept => "Finance",     salary => 68000, level => "Junior" },
    E005 => { name => "Eve Brown",     dept => "Engineering", salary => 78000, level => "Mid"    },
);

# Access
printf "E001 name: %s\n", $employees{E001}{name};
printf "E003 dept: %s\n", $employees{E003}{dept};

# Add field dynamically
for my $id (keys %employees) {
    $employees{$id}{annual_bonus} = int($employees{$id}{salary} * 0.10);
}

# Print table
printf "\n%-5s %-20s %-14s %8s %8s\n", "ID", "Name", "Department", "Salary", "Bonus";
print "-" x 60 . "\n";
for my $id (sort keys %employees) {
    my $e = $employees{$id};
    printf "%-5s %-20s %-14s %8.0f %8.0f\n",
        $id, $e->{name}, $e->{dept}, $e->{salary}, $e->{annual_bonus};
}

# =====================
# Nested modifications
# =====================

$employees{E006} = {
    name   => "Frank Miller",
    dept   => "Marketing",
    salary => 74000,
    level  => "Mid",
};
$employees{E006}{annual_bonus} = int($employees{E006}{salary} * 0.10);

printf "\nAdded E006: %s\n", $employees{E006}{name};

# Department statistics
my %dept_stats;
for my $id (keys %employees) {
    my $e = $employees{$id};
    $dept_stats{$e->{dept}}{count}++;
    $dept_stats{$e->{dept}}{total_salary} += $e->{salary};
    push @{$dept_stats{$e->{dept}}{employees}}, $e->{name};
}

print "\nDepartment Statistics:\n";
for my $dept (sort keys %dept_stats) {
    my $d = $dept_stats{$dept};
    printf "  %-14s: %d employees, avg salary \$%.0f\n",
        $dept, $d->{count}, $d->{total_salary} / $d->{count};
    printf "    Members: %s\n", join(", ", @{$d->{employees}});
}

# =====================
# 3-level hash
# =====================

my %company = (
    Engineering => {
        Frontend  => { head => "Alice", size => 5, budget => 500000 },
        Backend   => { head => "Carol", size => 8, budget => 800000 },
        DevOps    => { head => "Eve",   size => 3, budget => 300000 },
    },
    Marketing => {
        Digital  => { head => "Bob",   size => 4, budget => 400000 },
        Content  => { head => "Frank", size => 3, budget => 250000 },
    },
    Finance => {
        Accounting => { head => "Dave", size => 2, budget => 200000 },
    },
);

print "\n3-level hierarchy:\n";
for my $div (sort keys %company) {
    printf "  %s:\n", $div;
    for my $team (sort keys %{$company{$div}}) {
        my $t = $company{$div}{$team};
        printf "    %-12s head=%-6s size=%d budget=\$%d\n",
            $team, $t->{head}, $t->{size}, $t->{budget};
    }
}

# Total by division
for my $div (sort keys %company) {
    my $total_size   = 0;
    my $total_budget = 0;
    $total_size   += $company{$div}{$_}{size}   for keys %{$company{$div}};
    $total_budget += $company{$div}{$_}{budget} for keys %{$company{$div}};
    printf "  %s total: %d people, \$%d budget\n", $div, $total_size, $total_budget;
}
```

---

## Step 205: Deep Copy และ Clone

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Storable qw(dclone);
use Scalar::Util qw(reftype blessed);

# =====================
# Shallow copy problem
# =====================

my $orig = {
    name  => "Alice",
    scores => [90, 85, 92],
    meta  => { active => 1 },
};

# Shallow copy — สร้าง hash ใหม่ แต่ nested refs ยังชี้ที่เดิม
my $shallow = { %$orig };
$shallow->{name}       = "Bob";        # ไม่กระทบ $orig
push @{$shallow->{scores}}, 100;        # กระทบ $orig ด้วย!
$shallow->{meta}{active} = 0;           # กระทบ $orig ด้วย!

printf "orig name:   %s\n", $orig->{name};    # Alice
printf "shallow name: %s\n", $shallow->{name}; # Bob
printf "orig scores: %s\n",  join(",", @{$orig->{scores}});    # 90,85,92,100 ← changed!
printf "orig active: %d\n",  $orig->{meta}{active};            # 0 ← changed!

# =====================
# Deep copy — dclone (Storable)
# =====================

my $orig2 = {
    name  => "Carol",
    data  => { x => 1, y => [1,2,3] },
};

my $deep = dclone($orig2);   # completely independent copy
$deep->{name}          = "Dave";
push @{$deep->{data}{y}}, 99;
$deep->{data}{x}       = 999;

printf "\norig2 name: %s\n",    $orig2->{name};        # Carol
printf "deep name:  %s\n",      $deep->{name};          # Dave
printf "orig2 y: %s\n",         join(",", @{$orig2->{data}{y}}); # 1,2,3
printf "deep  y: %s\n",         join(",", @{$deep->{data}{y}});  # 1,2,3,99

# =====================
# Manual deep copy
# =====================

sub deep_copy {
    my $thing = shift;
    my $type = ref $thing;
    
    return $thing unless $type;
    
    if ($type eq 'HASH') {
        return { map { $_ => deep_copy($thing->{$_}) } keys %$thing };
    } elsif ($type eq 'ARRAY') {
        return [ map { deep_copy($_) } @$thing ];
    } elsif ($type eq 'SCALAR') {
        return \do { my $x = $$thing };
    } elsif ($type eq 'CODE') {
        return $thing;   # code refs can't be deep-copied
    } else {
        return dclone($thing);
    }
}

my $orig3 = { a => [1, [2, 3], { c => 4 }], b => "hello" };
my $copy3 = deep_copy($orig3);

$copy3->{a}[1][0] = 999;
$copy3->{b}       = "world";

printf "\norig3 a[1][0]: %d\n", $orig3->{a}[1][0];   # 2 — unchanged
printf "copy3 a[1][0]: %d\n",   $copy3->{a}[1][0];   # 999

# =====================
# Compare structures
# =====================

sub deep_equal {
    my ($a, $b) = @_;
    
    return 1 if !defined($a) && !defined($b);
    return 0 if !defined($a) || !defined($b);
    
    my $ta = ref($a);
    my $tb = ref($b);
    
    return 0 unless $ta eq $tb;
    
    unless ($ta) {
        return $a eq $b;
    }
    
    if ($ta eq 'HASH') {
        my @ka = sort keys %$a;
        my @kb = sort keys %$b;
        return 0 unless @ka ~~ @kb;
        for my $k (@ka) {
            return 0 unless deep_equal($a->{$k}, $b->{$k});
        }
        return 1;
    } elsif ($ta eq 'ARRAY') {
        return 0 unless @$a == @$b;
        for my $i (0..$#$a) {
            return 0 unless deep_equal($a->[$i], $b->[$i]);
        }
        return 1;
    }
    
    return "$a" eq "$b";
}

my $x = { a => [1,2,3], b => { c => "hello" } };
my $y = dclone($x);
my $z = { a => [1,2,4], b => { c => "hello" } };

printf "\ndeep_equal(x,y): %s\n", deep_equal($x,$y) ? "equal" : "different";
printf "deep_equal(x,z): %s\n",   deep_equal($x,$z) ? "equal" : "different";
```

---

## Step 206: Circular References

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Scalar::Util qw(weaken isweak refcount);

# =====================
# Circular reference — memory leak!
# =====================

{
    my $a = { name => "A" };
    my $b = { name => "B" };
    
    $a->{partner} = $b;   # A → B
    $b->{partner} = $a;   # B → A  ← circular!
    
    printf "A's partner: %s\n", $a->{partner}{name};
    printf "B's partner: %s\n", $b->{partner}{name};
    
    # When $a and $b go out of scope,
    # ref count never reaches 0 → memory leak!
}

# =====================
# Weak references — เพื่อแก้ปัญหา
# =====================

{
    my $a = { name => "Alice" };
    my $b = { name => "Bob"   };
    
    $a->{friend} = $b;   # strong ref A → B
    $b->{friend} = $a;   # strong ref B → A
    
    # Weaken one side
    weaken($b->{friend});
    
    printf "\nisweak(a→b): %s\n", isweak($a->{friend}) ? "yes" : "no";   # no
    printf "isweak(b→a): %s\n",   isweak($b->{friend}) ? "yes" : "no";   # yes
    
    printf "Alice's friend: %s\n", $a->{friend}{name};
    printf "Bob's friend:   %s\n", $b->{friend}{name} // "(gone)";
}

# =====================
# Tree with parent pointer (weak)
# =====================

sub new_node {
    my ($val, $parent) = @_;
    my $node = { value => $val, parent => $parent, children => [] };
    weaken($node->{parent}) if defined $node->{parent};
    return $node;
}

sub add_child {
    my ($parent, $val) = @_;
    my $child = new_node($val, $parent);
    push @{$parent->{children}}, $child;
    return $child;
}

sub print_tree {
    my ($node, $indent) = @_;
    $indent //= 0;
    printf "%s%s (parent: %s)\n",
        "  " x $indent,
        $node->{value},
        $node->{parent} ? $node->{parent}{value} : "(root)";
    print_tree($_, $indent + 1) for @{$node->{children}};
}

my $root  = new_node("root", undef);
my $child1 = add_child($root, "child1");
my $child2 = add_child($root, "child2");
my $gc1    = add_child($child1, "grandchild1");
my $gc2    = add_child($child1, "grandchild2");

print "\nTree:\n";
print_tree($root);

# Navigate up tree
printf "\ngc1 parent: %s\n", $gc1->{parent}{value};
printf "gc1 grandparent: %s\n", $gc1->{parent}{parent}{value};

# =====================
# Observer pattern with weak refs
# =====================

package Event::Emitter;

sub new { bless { listeners => {} }, shift }

sub on {
    my ($self, $event, $cb) = @_;
    push @{$self->{listeners}{$event}}, $cb;
}

sub on_weak {
    my ($self, $event, $obj, $method) = @_;
    my $weak_obj = $obj;
    weaken($weak_obj);
    push @{$self->{listeners}{$event}}, sub {
        $weak_obj->$method(@_) if defined $weak_obj;
    };
}

sub emit {
    my ($self, $event, @args) = @_;
    for my $cb (@{$self->{listeners}{$event} // []}) {
        $cb->(@args);
    }
    # Clean up dead weak refs
    $self->{listeners}{$event} = [
        grep { defined $_ } @{$self->{listeners}{$event}}
    ];
}

package main;

my $emitter = Event::Emitter->new;
$emitter->on('data', sub { printf "Received: %s\n", $_[0] });

$emitter->emit('data', "hello");
$emitter->emit('data', "world");

print "\nCircular reference demo complete.\n";
```

---

## Step 207: Data Manipulation Patterns

```perl
#!/usr/bin/perl
use strict;
use warnings;
use List::Util qw(sum min max first reduce);
use Scalar::Util qw(looks_like_number);

# =====================
# Transform pipeline
# =====================

my @raw_data = (
    { name => "  Alice  ", age => "28", salary => "85000.5" },
    { name => "BOB",       age => "35", salary => "72000.0" },
    { name => "carol",     age => "22", salary => "58000.0" },
    { name => "Dave",      age => "41", salary => "95000.0" },
    { name => "",          age => "25", salary => "bad"     },   # invalid
);

# Step 1: Clean
my @cleaned = map {
    {
        name   => do { (my $n = $_->{name}) =~ s/^\s+|\s+$//g; ucfirst(lc $n) },
        age    => int($_->{age}),
        salary => looks_like_number($_->{salary}) ? $_->{salary}+0 : undef,
    }
} @raw_data;

# Step 2: Filter invalid
my @valid = grep { $_->{name} && defined $_->{salary} } @cleaned;

printf "Valid records: %d/%d\n", scalar @valid, scalar @raw_data;

# Step 3: Enrich
my @enriched = map {
    my $r = $_;
    {
        %$r,
        level => $r->{salary} >= 90000 ? "Senior"  :
                 $r->{salary} >= 70000 ? "Mid"      : "Junior",
        tax   => $r->{salary} * 0.20,
        net   => $r->{salary} * 0.80,
    }
} @valid;

# Step 4: Sort and display
my @sorted = sort { $b->{salary} <=> $a->{salary} } @enriched;

printf "\n%-10s %3s %10s %6s %10s\n", "Name","Age","Salary","Level","Net";
print "-" x 44 . "\n";
printf "%-10s %3d %10.0f %-6s %10.0f\n",
    $_->{name}, $_->{age}, $_->{salary}, $_->{level}, $_->{net}
    for @sorted;

my $total = sum(map { $_->{salary} } @enriched);
printf "\nTotal payroll: \$%.0f\n", $total;
printf "Average: \$%.0f\n", $total / @enriched;

# =====================
# reduce patterns
# =====================

my @nums = (1..10);

my $sum     = reduce { $a + $b } @nums;
my $product = reduce { $a * $b } @nums;
my $max_val = reduce { $a > $b ? $a : $b } @nums;

printf "\nSum:     %d\n", $sum;
printf "Product: %d\n", $product;
printf "Max:     %d\n", $max_val;

# Flatten array of arrays
my @nested = ([1,2,3], [4,5], [6,7,8,9]);
my @flat   = map { @$_ } @nested;
printf "Flat: %s\n", join(",", @flat);

# Group by
sub group_by {
    my ($key_fn, @items) = @_;
    my %groups;
    push @{$groups{$key_fn->($_)}}, $_ for @items;
    return %groups;
}

my %by_level = group_by(sub { $_[0]{level} }, @enriched);
for my $level (sort keys %by_level) {
    printf "  %s: %s\n", $level,
        join(", ", map { $_->{name} } @{$by_level{$level}});
}

# =====================
# Chaining with closures
# =====================

package Pipeline;

sub new {
    my ($class, @data) = @_;
    return bless { data => \@data }, $class;
}

sub filter {
    my ($self, $pred) = @_;
    return Pipeline->new(grep { $pred->($_) } @{$self->{data}});
}

sub map_data {
    my ($self, $fn) = @_;
    return Pipeline->new(map { $fn->($_) } @{$self->{data}});
}

sub sort_by {
    my ($self, $fn) = @_;
    return Pipeline->new(sort { $fn->($a) cmp $fn->($b) } @{$self->{data}});
}

sub take {
    my ($self, $n) = @_;
    return Pipeline->new(@{$self->{data}}[0..($n-1)]);
}

sub result { @{$_[0]->{data}} }

package main;

my @result = Pipeline->new(@sorted)
    ->filter(sub { $_[0]{salary} > 70000 })
    ->map_data(sub { { name => $_[0]{name}, net => $_[0]{net} } })
    ->sort_by(sub { $_[0]{name} })
    ->result;

printf "\nPipeline result:\n";
printf "  %s: \$%.0f net\n", $_->{name}, $_->{net} for @result;
```

---

## Step 208: JSON Processing

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

my $json = JSON::PP->new->utf8->canonical->pretty;

# =====================
# Encode Perl → JSON
# =====================

my $data = {
    name    => "Alice",
    age     => 28,
    active  => JSON::PP::true,
    scores  => [95, 87, 92],
    address => {
        street => "123 Main St",
        city   => "Bangkok",
        zip    => "10100",
    },
    notes   => undef,
};

my $json_str = $json->encode($data);
print "Encoded JSON:\n$json_str\n";

# =====================
# Decode JSON → Perl
# =====================

my $json_input = <<'JSON';
{
  "users": [
    {"id": 1, "name": "Alice", "role": "admin", "active": true},
    {"id": 2, "name": "Bob",   "role": "user",  "active": false},
    {"id": 3, "name": "Carol", "role": "user",  "active": true}
  ],
  "total": 3,
  "page": 1
}
JSON

my $decoded = $json->decode($json_input);

printf "Total users: %d\n", $decoded->{total};
printf "Page: %d\n",        $decoded->{page};

for my $user (@{$decoded->{users}}) {
    printf "  #%d %s (%s) %s\n",
        $user->{id}, $user->{name}, $user->{role},
        $user->{active} ? "active" : "inactive";
}

# =====================
# JSON file I/O
# =====================

my $config = {
    version => "1.0",
    debug   => JSON::PP::false,
    database => {
        host => "localhost",
        port => 5432,
        name => "mydb",
    },
    allowed_origins => ["http://localhost:3000", "https://myapp.com"],
};

# Write
my $config_file = "/tmp/config.json";
open(my $fh_w, '>:utf8', $config_file) or die "Cannot write: $!";
print $fh_w $json->encode($config);
close $fh_w;

print "Config saved to $config_file\n";

# Read back
open(my $fh_r, '<:utf8', $config_file) or die "Cannot read: $!";
local $/;
my $loaded = $json->decode(<$fh_r>);
close $fh_r;

printf "Loaded config: version=%s, db=%s\n",
    $loaded->{version}, $loaded->{database}{name};

# =====================
# JSON API response builder
# =====================

sub json_ok {
    my ($data, %opts) = @_;
    return {
        success => JSON::PP::true,
        data    => $data,
        meta    => {
            count     => ref $data eq 'ARRAY' ? scalar @$data : 1,
            timestamp => time(),
        },
    };
}

sub json_error {
    my ($code, $message) = @_;
    return {
        success => JSON::PP::false,
        error   => {
            code    => $code,
            message => $message,
        },
    };
}

my $j2 = JSON::PP->new->utf8->canonical;

my $ok_resp = json_ok([
    { id => 1, title => "Item 1" },
    { id => 2, title => "Item 2" },
]);

my $err_resp = json_error(404, "Resource not found");

printf "\nOK response:\n%s\n",    $json->encode($ok_resp);
printf "Error response:\n%s\n",   $json->encode($err_resp);

# =====================
# Deep merge
# =====================

sub merge_hashes {
    my ($base, $override) = @_;
    my %result = %$base;
    
    for my $key (keys %$override) {
        if (ref $override->{$key} eq 'HASH' && ref $result{$key} eq 'HASH') {
            $result{$key} = merge_hashes($result{$key}, $override->{$key});
        } else {
            $result{$key} = $override->{$key};
        }
    }
    
    return \%result;
}

my $default_cfg = { a => 1, b => { c => 2, d => 3 }, e => [1,2,3] };
my $user_cfg    = { a => 9, b => { c => 99 } };
my $merged      = merge_hashes($default_cfg, $user_cfg);

printf "\nMerged: %s\n", $j2->encode($merged);
```

---

## Step 209: Functional Programming Patterns

```perl
#!/usr/bin/perl
use strict;
use warnings;
use List::Util qw(sum reduce first any all none);

# =====================
# Higher-order functions
# =====================

sub compose {
    my @fns = reverse @_;
    return sub {
        my @args = @_;
        my $result = $fns[0]->(@args);
        $result = $_->($result) for @fns[1..$#fns];
        return $result;
    };
}

sub pipe {
    my @fns = @_;
    return sub {
        my @args = @_;
        my $result = $fns[0]->(@args);
        $result = $_->($result) for @fns[1..$#fns];
        return $result;
    };
}

my $double  = sub { $_[0] * 2 };
my $inc     = sub { $_[0] + 1 };
my $square  = sub { $_[0] ** 2 };

my $double_then_inc = compose($inc, $double);
printf "double_then_inc(5): %d\n", $double_then_inc->(5);  # 11

my $pipeline = pipe($double, $inc, $square);
printf "pipeline(5): %d\n", $pipeline->(5);  # (5*2+1)^2 = 121

# =====================
# Curry
# =====================

sub curry {
    my ($fn, @partial) = @_;
    return sub { $fn->(@partial, @_) };
}

my $add   = sub { $_[0] + $_[1] };
my $add5  = curry($add, 5);
my $add10 = curry($add, 10);

printf "\nadd5(3): %d\n",   $add5->(3);    # 8
printf "add10(3): %d\n",    $add10->(3);   # 13

my $multiply = sub { $_[0] * $_[1] };
my $double2  = curry($multiply, 2);
my @doubled  = map { $double2->($_) } 1..5;
printf "doubled: %s\n", join(",", @doubled);   # 2,4,6,8,10

# =====================
# Memoize
# =====================

sub memoize {
    my $fn = shift;
    my %cache;
    return sub {
        my $key = join("\0", @_);
        unless (exists $cache{$key}) {
            $cache{$key} = $fn->(@_);
        }
        return $cache{$key};
    };
}

my $fib;
$fib = memoize(sub {
    my $n = shift;
    return $n if $n <= 1;
    return $fib->($n-1) + $fib->($n-2);
});

printf "\nFib(10): %d\n", $fib->(10);
printf "Fib(20): %d\n",   $fib->(20);
printf "Fib(30): %d\n",   $fib->(30);

# =====================
# Lazy evaluation
# =====================

sub lazy {
    my $fn = shift;
    my $computed = 0;
    my $value;
    return sub {
        unless ($computed) {
            $value    = $fn->();
            $computed = 1;
        }
        return $value;
    };
}

my $expensive = lazy(sub {
    printf "(computing expensive value)\n";
    return 42;
});

printf "\nLazy value: %d\n", $expensive->();  # computes
printf "Lazy value: %d\n",   $expensive->();  # cached

# =====================
# Partial application
# =====================

sub partial {
    my ($fn, @partial_args) = @_;
    return sub { $fn->(@partial_args, @_) };
}

sub filter_by { my ($pred, @arr) = @_; grep { $pred->($_) } @arr }
sub map_by    { my ($fn,   @arr) = @_; map  { $fn->($_)   } @arr }

my $is_even    = sub { $_[0] % 2 == 0 };
my $is_positive = sub { $_[0] > 0 };

my @numbers = (-3, -1, 0, 1, 2, 3, 4, 5, 6);
my @evens   = filter_by($is_even, @numbers);
my @pos     = filter_by($is_positive, @numbers);

printf "\nEvens:    %s\n", join(",", @evens);
printf "Positives: %s\n",  join(",", @pos);

# any/all/none (List::Util)
printf "\nany even: %s\n", (any { $_ % 2 == 0 } @numbers) ? "yes" : "no";
printf "all pos:  %s\n",  (all { $_ > 0 } @numbers) ? "yes" : "no";
printf "none neg: %s\n",  (none { $_ < 0 } @numbers) ? "yes" : "no";

# =====================
# Trampolining (for tail-call simulation)
# =====================

sub trampoline {
    my $fn = shift;
    my @args = @_;
    my $result;
    while (ref $fn eq 'CODE') {
        $result = $fn->(@args);
        if (ref $result eq 'CODE') {
            ($fn, @args) = ($result);
        } else {
            last;
        }
    }
    return ref $result eq 'CODE' ? $result->() : $result;
}

sub count_down {
    my $n = shift;
    return $n if $n <= 0;
    return sub { count_down($n - 1) };
}

printf "\ntrampoline count_down(10000): %d\n",
    trampoline(\&count_down, 10000);
```

---

## Step 210: โปรแกรมสรุป — Data Processing Engine

```perl
#!/usr/bin/perl
# data_engine.pl — Complex data processing
use strict;
use warnings;
use JSON::PP;
use List::Util qw(sum min max reduce first);
use Scalar::Util qw(looks_like_number);
use Storable qw(dclone);
use POSIX qw(strftime);

# =====================
# Data Engine
# =====================

package DataEngine;

sub new {
    my ($class, @data) = @_;
    return bless { data => \@data, ops => [] }, $class;
}

sub _apply {
    my ($self, @data) = @_;
    my @result = @data;
    $result[0] = $_->($result[0]) for @{$self->{ops}};
    return @{$result[0]};
}

sub where    { my ($self,$fn) = @_; push @{$self->{ops}}, sub { [grep { $fn->($_) }  @{$_[0]}] }; $self }
sub select   { my ($self,$fn) = @_; push @{$self->{ops}}, sub { [map  { $fn->($_) }  @{$_[0]}] }; $self }
sub order_by { my ($self,$fn,$d) = @_; push @{$self->{ops}}, sub { [sort { $d&&$d eq'desc' ? $fn->($b)<=>$fn->($a)||$fn->($b)cmp$fn->($a) : $fn->($a)<=>$fn->($b)||$fn->($a)cmp$fn->($b) } @{$_[0]}] }; $self }
sub take     { my ($self,$n) = @_;   push @{$self->{ops}}, sub { [@{$_[0]}[0..($n<@{$_[0]}?$n-1:$#@{$_[0]})]] }; $self }
sub skip     { my ($self,$n) = @_;   push @{$self->{ops}}, sub { [@{$_[0]}[$n..$#{$_[0]}]] }; $self }

sub execute {
    my $self = shift;
    my @result = @{$self->{data}};
    for my $op (@{$self->{ops}}) {
        @result = @{$op->(\@result)};
    }
    return @result;
}

sub count { my $self = shift; return scalar $self->execute }
sub first { my $self = shift; return ($self->take(1)->execute)[0] }

sub aggregate {
    my ($self, %agg) = @_;
    my @data = $self->execute;
    my %result;
    for my $key (keys %agg) {
        $result{$key} = $agg{$key}->(\@data);
    }
    return %result;
}

sub group_by {
    my ($self, $key_fn) = @_;
    my @data = $self->execute;
    my %groups;
    push @{$groups{$key_fn->($_)}}, $_ for @data;
    return %groups;
}

# =====================
# Main demo
# =====================

package main;

# Sales data
my @sales = map {
    my $month = 1 + int(rand 12);
    my $region = ("North","South","East","West")[rand 4];
    my $product = ("Widget A","Widget B","Gadget X","Gadget Y","Tool Z")[rand 5];
    my $qty    = 1 + int rand 50;
    my $price  = 10 + rand 100;
    {
        id      => $_,
        month   => $month,
        region  => $region,
        product => $product,
        qty     => $qty,
        price   => sprintf("%.2f", $price),
        revenue => sprintf("%.2f", $qty * $price),
    }
} 1..500;

printf "Dataset: %d sales records\n\n", scalar @sales;

# Query 1: Top revenue products
my $engine1 = DataEngine->new(@sales);

my %by_product = $engine1->group_by(sub { $_[0]{product} });

my @product_summary = map {
    my $name  = $_;
    my @group = @{$by_product{$name}};
    my $rev   = sum(map { $_->{revenue} } @group);
    my $units = sum(map { $_->{qty} } @group);
    { product => $name, revenue => $rev, units => $units, orders => scalar @group }
} keys %by_product;

my @top_products = sort { $b->{revenue} <=> $a->{revenue} } @product_summary;

print "=== TOP PRODUCTS BY REVENUE ===\n";
printf "%-12s %8s %8s %6s\n", "Product","Revenue","Units","Orders";
print "-" x 38 . "\n";
printf "%-12s %8.0f %8d %6d\n",
    $_->{product}, $_->{revenue}, $_->{units}, $_->{orders}
    for @top_products;

# Query 2: Regional performance
my $engine2 = DataEngine->new(@sales);
my %by_region = $engine2->group_by(sub { $_[0]{region} });

print "\n=== REGIONAL PERFORMANCE ===\n";
for my $region (sort keys %by_region) {
    my @group = @{$by_region{$region}};
    my $rev   = sum(map { $_->{revenue} } @group);
    my $avg   = $rev / @group;
    printf "  %-8s: %4d orders, \$%.0f total, \$%.0f avg\n",
        $region, scalar @group, $rev, $avg;
}

# Query 3: Monthly trend
my $engine3 = DataEngine->new(@sales);
my %by_month = $engine3->group_by(sub { sprintf("%02d", $_[0]{month}) });

print "\n=== MONTHLY REVENUE TREND ===\n";
for my $month (sort keys %by_month) {
    my @group = @{$by_month{$month}};
    my $rev   = sum(map { $_->{revenue} } @group);
    my $bar   = "█" x int($rev / 5000);
    printf "  Month %s: \$%6.0f %s\n", $month, $rev, substr($bar, 0, 30);
}

# Query 4: High-value orders
my $engine4 = DataEngine->new(@sales);
my @big_orders = DataEngine->new(@sales)
    ->where(sub { $_[0]{revenue} >= 2000 })
    ->order_by(sub { -$_[0]{revenue} })
    ->take(5)
    ->execute;

print "\n=== TOP 5 ORDERS ===\n";
printf "  #%-4d %-12s %-8s \$%.2f\n",
    $_->{id}, $_->{product}, $_->{region}, $_->{revenue}
    for @big_orders;

# Summary stats
my @all_revenues = map { $_->{revenue} } @sales;
printf "\n=== SUMMARY ===\n";
printf "Total revenue: \$%.0f\n", sum(@all_revenues);
printf "Average order: \$%.2f\n", sum(@all_revenues) / @all_revenues;
printf "Min order:     \$%.2f\n", min(@all_revenues);
printf "Max order:     \$%.2f\n", max(@all_revenues);

print "\nData engine demo complete.\n";
```

---

## สรุป Part 21

ใน Part นี้คุณได้เรียนรู้:
- ✅ Scalar, Array, Hash, Code references
- ✅ Array of Hashes และ Hash of Hashes
- ✅ Hash of Arrays
- ✅ Deep copy vs shallow copy
- ✅ Circular references และ weak references
- ✅ Data manipulation patterns (filter/map/reduce)
- ✅ JSON processing
- ✅ Functional programming patterns (compose, curry, memoize)
- ✅ Data processing engine

**ถัดไป: [Part 22 — Advanced OOP](part_22.md)**
