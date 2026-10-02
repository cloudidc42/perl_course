# Part 16: Advanced Data Structures
## Steps 151-160: โครงสร้างข้อมูลขั้นสูง

---

## Step 151: References ขั้นสูง

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Weak references
# =====================

use Scalar::Util qw(weaken isweak);

# Circular reference problem
{
    my $a = { name => "A" };
    my $b = { name => "B" };
    
    # Without weaken — memory leak
    # $a->{next} = $b;
    # $b->{prev} = $a;   # circular ref!
    
    # With weaken — safe
    $a->{next} = $b;
    $b->{prev} = $a;
    weaken($b->{prev});   # break circular ref
    
    printf "a: %s, b: %s\n", $a->{name}, $b->{name};
    printf "is weak: %s\n", isweak($b->{prev}) ? "yes" : "no";
}

# =====================
# Nested data structures
# =====================

my %company = (
    name       => "TechCorp",
    founded    => 2000,
    departments => {
        engineering => {
            head  => "Alice",
            teams => ["frontend", "backend", "devops"],
            count => 50,
        },
        marketing => {
            head  => "Bob",
            teams => ["brand", "digital", "pr"],
            count => 20,
        },
        sales => {
            head  => "Carol",
            teams => ["enterprise", "smb"],
            count => 30,
        },
    },
    employees  => [
        { id => 1, name => "Alice",  dept => "engineering", salary => 120000 },
        { id => 2, name => "Bob",    dept => "marketing",   salary => 90000 },
        { id => 3, name => "Carol",  dept => "sales",       salary => 95000 },
        { id => 4, name => "Dave",   dept => "engineering", salary => 105000 },
        { id => 5, name => "Eve",    dept => "engineering", salary => 115000 },
    ],
);

# Navigate nested structure
my $eng = $company{departments}{engineering};
printf "Engineering: %d people, head: %s\n", $eng->{count}, $eng->{head};
printf "Teams: %s\n", join(", ", @{$eng->{teams}});

# Find highest paid
my @sorted = sort { $b->{salary} <=> $a->{salary} } @{$company{employees}};
printf "Highest paid: %s (\$%d)\n", $sorted[0]{name}, $sorted[0]{salary};

# Group by dept
my %by_dept;
for my $emp (@{$company{employees}}) {
    push @{$by_dept{$emp->{dept}}}, $emp->{name};
}
printf "Engineering: %s\n", join(", ", @{$by_dept{engineering}});

# =====================
# Ref counting tricks
# =====================

sub deep_clone {
    my $data = shift;
    
    return $data unless ref $data;
    
    if (ref $data eq 'ARRAY') {
        return [ map { deep_clone($_) } @$data ];
    }
    if (ref $data eq 'HASH') {
        return { map { $_ => deep_clone($data->{$_}) } keys %$data };
    }
    return $data;  # scalar ref, code ref, etc.
}

my $orig = { a => [1,2,3], b => { x => 1 } };
my $copy = deep_clone($orig);
$copy->{a}[0] = 99;
printf "orig: %d, copy: %d\n", $orig->{a}[0], $copy->{a}[0];   # 1, 99
```

---

## Step 152: Stack, Queue, Deque

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Stack (LIFO)
# =====================

package Stack;

sub new    { bless [], shift }
sub push   { push @{$_[0]}, $_[1] }
sub pop    { pop @{$_[0]} }
sub peek   { $_[0][-1] }
sub is_empty { @{$_[0]} == 0 }
sub size   { scalar @{$_[0]} }
sub clear  { @{$_[0]} = () }

package main;

my $stack = Stack->new;
$stack->push(1);
$stack->push(2);
$stack->push(3);

printf "size=%d, peek=%d\n", $stack->size, $stack->peek;
printf "pop: %d\n", $stack->pop;
printf "pop: %d\n", $stack->pop;
printf "size=%d\n", $stack->size;

# Use case: balanced parentheses
sub is_balanced {
    my $str = shift;
    my $stack = Stack->new;
    my %pairs = (')' => '(', ']' => '[', '}' => '{');
    
    for my $ch (split //, $str) {
        if ($ch =~ /[\(\[\{]/) {
            $stack->push($ch);
        } elsif ($ch =~ /[\)\]\}]/) {
            return 0 if $stack->is_empty;
            return 0 if $stack->pop ne $pairs{$ch};
        }
    }
    
    return $stack->is_empty;
}

for my $expr ("(a + b) * (c - d)", "((x + y))", "([{nested}])", "(unclosed") {
    printf "%-30s %s\n", $expr, is_balanced($expr) ? "balanced" : "unbalanced";
}

# =====================
# Queue (FIFO)
# =====================

package Queue;

sub new     { bless [], shift }
sub enqueue { push @{$_[0]}, $_[1] }
sub dequeue { shift @{$_[0]} }
sub front   { $_[0][0] }
sub is_empty { @{$_[0]} == 0 }
sub size    { scalar @{$_[0]} }

package main;

my $q = Queue->new;
$q->enqueue("first");
$q->enqueue("second");
$q->enqueue("third");

printf "front=%s, size=%d\n", $q->front, $q->size;
printf "dequeue: %s\n", $q->dequeue;
printf "dequeue: %s\n", $q->dequeue;

# =====================
# Priority Queue
# =====================

package PriorityQueue;

sub new { bless { heap => [] }, shift }

sub insert {
    my ($self, $priority, $value) = @_;
    push @{$self->{heap}}, [$priority, $value];
    $self->_sift_up($#{$self->{heap}});
}

sub extract_max {
    my $self = shift;
    return undef unless @{$self->{heap}};
    
    my $max = $self->{heap}[0];
    my $last = pop @{$self->{heap}};
    
    if (@{$self->{heap}}) {
        $self->{heap}[0] = $last;
        $self->_sift_down(0);
    }
    
    return $max->[1];
}

sub _sift_up {
    my ($self, $i) = @_;
    my $heap = $self->{heap};
    while ($i > 0) {
        my $parent = int(($i-1)/2);
        last if $heap->[$parent][0] >= $heap->[$i][0];
        @{$heap}[$parent, $i] = @{$heap}[$i, $parent];
        $i = $parent;
    }
}

sub _sift_down {
    my ($self, $i) = @_;
    my $heap = $self->{heap};
    my $n = scalar @$heap;
    while (1) {
        my ($left, $right) = (2*$i+1, 2*$i+2);
        my $largest = $i;
        $largest = $left  if $left  < $n && $heap->[$left][0]  > $heap->[$largest][0];
        $largest = $right if $right < $n && $heap->[$right][0] > $heap->[$largest][0];
        last if $largest == $i;
        @{$heap}[$i, $largest] = @{$heap}[$largest, $i];
        $i = $largest;
    }
}

sub is_empty { @{$_[0]->{heap}} == 0 }
sub size     { scalar @{$_[0]->{heap}} }

package main;

my $pq = PriorityQueue->new;
$pq->insert(3, "medium");
$pq->insert(1, "low");
$pq->insert(5, "high");
$pq->insert(2, "low2");
$pq->insert(4, "medium2");

print "\nPriority queue (max first):\n";
while (!$pq->is_empty) {
    print "  ", $pq->extract_max, "\n";
}
```

---

## Step 153: Linked List

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Singly linked list
# =====================

package Node;
sub new {
    my ($class, $val) = @_;
    return bless { val => $val, next => undef }, $class;
}

package LinkedList;

sub new { bless { head => undef, size => 0 }, shift }

sub prepend {
    my ($self, $val) = @_;
    my $node = Node->new($val);
    $node->{next} = $self->{head};
    $self->{head} = $node;
    $self->{size}++;
}

sub append {
    my ($self, $val) = @_;
    my $node = Node->new($val);
    if (!$self->{head}) {
        $self->{head} = $node;
    } else {
        my $curr = $self->{head};
        $curr = $curr->{next} while $curr->{next};
        $curr->{next} = $node;
    }
    $self->{size}++;
}

sub delete_val {
    my ($self, $val) = @_;
    return unless $self->{head};
    
    if ($self->{head}{val} == $val) {
        $self->{head} = $self->{head}{next};
        $self->{size}--;
        return 1;
    }
    
    my $curr = $self->{head};
    while ($curr->{next}) {
        if ($curr->{next}{val} == $val) {
            $curr->{next} = $curr->{next}{next};
            $self->{size}--;
            return 1;
        }
        $curr = $curr->{next};
    }
    return 0;
}

sub to_array {
    my $self = shift;
    my @result;
    my $curr = $self->{head};
    while ($curr) {
        push @result, $curr->{val};
        $curr = $curr->{next};
    }
    return @result;
}

sub size { $_[0]->{size} }

sub reverse_list {
    my $self = shift;
    my $prev = undef;
    my $curr = $self->{head};
    while ($curr) {
        my $next = $curr->{next};
        $curr->{next} = $prev;
        $prev = $curr;
        $curr = $next;
    }
    $self->{head} = $prev;
}

package main;

my $list = LinkedList->new;
$list->append(1);
$list->append(2);
$list->append(3);
$list->prepend(0);
$list->append(4);

print "List: ", join(" -> ", $list->to_array), "\n";
printf "Size: %d\n", $list->size;

$list->delete_val(2);
print "After delete(2): ", join(" -> ", $list->to_array), "\n";

$list->reverse_list;
print "Reversed: ", join(" -> ", $list->to_array), "\n";

# =====================
# Doubly linked list
# =====================

package DNode;
sub new {
    my ($class, $val) = @_;
    return bless { val => $val, next => undef, prev => undef }, $class;
}

package DoubleList;

sub new { bless { head => undef, tail => undef, size => 0 }, shift }

sub append {
    my ($self, $val) = @_;
    my $node = DNode->new($val);
    if (!$self->{head}) {
        $self->{head} = $self->{tail} = $node;
    } else {
        $node->{prev} = $self->{tail};
        $self->{tail}{next} = $node;
        $self->{tail} = $node;
    }
    $self->{size}++;
}

sub to_array_forward {
    my $self = shift;
    my @r;
    my $n = $self->{head};
    while ($n) { push @r, $n->{val}; $n = $n->{next} }
    return @r;
}

sub to_array_backward {
    my $self = shift;
    my @r;
    my $n = $self->{tail};
    while ($n) { push @r, $n->{val}; $n = $n->{prev} }
    return @r;
}

package main;

my $dl = DoubleList->new;
$dl->append($_) for 1..5;
print "Forward:  ", join(" <-> ", $dl->to_array_forward), "\n";
print "Backward: ", join(" <-> ", $dl->to_array_backward), "\n";
```

---

## Step 154: Trees

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Binary Search Tree
# =====================

package BST;

sub new { bless { root => undef }, shift }

sub insert {
    my ($self, $val) = @_;
    $self->{root} = _insert($self->{root}, $val);
}

sub _insert {
    my ($node, $val) = @_;
    unless ($node) {
        return { val => $val, left => undef, right => undef };
    }
    if ($val < $node->{val}) {
        $node->{left} = _insert($node->{left}, $val);
    } elsif ($val > $node->{val}) {
        $node->{right} = _insert($node->{right}, $val);
    }
    return $node;
}

sub search {
    my ($self, $val) = @_;
    return _search($self->{root}, $val);
}

sub _search {
    my ($node, $val) = @_;
    return 0 unless $node;
    return 1 if $node->{val} == $val;
    return $val < $node->{val}
        ? _search($node->{left}, $val)
        : _search($node->{right}, $val);
}

sub inorder {
    my $self = shift;
    my @result;
    _inorder($self->{root}, \@result);
    return @result;
}

sub _inorder {
    my ($node, $result) = @_;
    return unless $node;
    _inorder($node->{left}, $result);
    push @$result, $node->{val};
    _inorder($node->{right}, $result);
}

sub height {
    my $self = shift;
    return _height($self->{root});
}

sub _height {
    my $node = shift;
    return 0 unless $node;
    my $left  = _height($node->{left});
    my $right = _height($node->{right});
    return 1 + ($left > $right ? $left : $right);
}

package main;

my $bst = BST->new;
$bst->insert($_) for (5, 3, 7, 1, 4, 6, 8, 2);

print "Inorder: ", join(", ", $bst->inorder), "\n";
printf "Height: %d\n", $bst->height;
printf "Search 4: %s\n", $bst->search(4) ? "found" : "not found";
printf "Search 9: %s\n", $bst->search(9) ? "found" : "not found";

# =====================
# N-ary Tree with traversal
# =====================

package NTree;

sub new {
    my ($class, $val) = @_;
    return bless { val => $val, children => [] }, $class;
}

sub add_child {
    my ($self, $child) = @_;
    push @{$self->{children}}, $child;
    return $child;
}

sub bfs {  # breadth-first
    my $root = shift;
    my @queue = ($root);
    my @result;
    while (@queue) {
        my $node = shift @queue;
        push @result, $node->{val};
        push @queue, @{$node->{children}};
    }
    return @result;
}

sub dfs {  # depth-first (preorder)
    my $root = shift;
    my @result;
    _dfs($root, \@result);
    return @result;
}

sub _dfs {
    my ($node, $result) = @_;
    push @$result, $node->{val};
    _dfs($_, $result) for @{$node->{children}};
}

package main;

my $root = NTree->new("root");
my $a = $root->add_child(NTree->new("A"));
my $b = $root->add_child(NTree->new("B"));
my $c = $root->add_child(NTree->new("C"));
$a->add_child(NTree->new("A1"));
$a->add_child(NTree->new("A2"));
$b->add_child(NTree->new("B1"));
$c->add_child(NTree->new("C1"));
$c->add_child(NTree->new("C2"));
$c->add_child(NTree->new("C3"));

printf "BFS: %s\n", join(" ", NTree::bfs($root));
printf "DFS: %s\n", join(" ", NTree::dfs($root));
```

---

## Step 155: Graphs

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Graph (Adjacency List)
# =====================

package Graph;

sub new {
    my ($class, %opts) = @_;
    return bless {
        adj      => {},
        directed => $opts{directed} // 0,
        weighted => $opts{weighted} // 0,
    }, $class;
}

sub add_vertex {
    my ($self, $v) = @_;
    $self->{adj}{$v} //= [];
}

sub add_edge {
    my ($self, $from, $to, $weight) = @_;
    $weight //= 1;
    
    $self->add_vertex($from);
    $self->add_vertex($to);
    
    push @{$self->{adj}{$from}}, { to => $to, w => $weight };
    unless ($self->{directed}) {
        push @{$self->{adj}{$to}}, { to => $from, w => $weight };
    }
}

sub vertices { sort keys %{$_[0]->{adj}} }

sub neighbors {
    my ($self, $v) = @_;
    return map { $_->{to} } @{$self->{adj}{$v} // []};
}

# BFS
sub bfs {
    my ($self, $start) = @_;
    my %visited;
    my @queue = ($start);
    my @order;
    
    $visited{$start} = 1;
    
    while (@queue) {
        my $v = shift @queue;
        push @order, $v;
        
        for my $n ($self->neighbors($v)) {
            next if $visited{$n};
            $visited{$n} = 1;
            push @queue, $n;
        }
    }
    
    return @order;
}

# DFS
sub dfs {
    my ($self, $start) = @_;
    my %visited;
    my @order;
    
    $self->_dfs($start, \%visited, \@order);
    
    return @order;
}

sub _dfs {
    my ($self, $v, $visited, $order) = @_;
    $visited->{$v} = 1;
    push @$order, $v;
    
    for my $n (sort $self->neighbors($v)) {
        $self->_dfs($n, $visited, $order) unless $visited->{$n};
    }
}

# Dijkstra's shortest path
sub dijkstra {
    my ($self, $start) = @_;
    
    my %dist = map { $_ => 9**9 } $self->vertices;
    my %prev;
    my %visited;
    $dist{$start} = 0;
    
    while (1) {
        # Find unvisited vertex with minimum distance
        my $u = undef;
        for my $v ($self->vertices) {
            next if $visited{$v};
            if (!defined $u || $dist{$v} < $dist{$u}) {
                $u = $v;
            }
        }
        last unless defined $u && $dist{$u} < 9**9;
        
        $visited{$u} = 1;
        
        for my $edge (@{$self->{adj}{$u}}) {
            my ($v, $w) = @{$edge}{qw(to w)};
            my $alt = $dist{$u} + $w;
            if ($alt < $dist{$v}) {
                $dist{$v} = $alt;
                $prev{$v} = $u;
            }
        }
    }
    
    return (\%dist, \%prev);
}

sub path {
    my ($self, $start, $end, $prev) = @_;
    my @path;
    my $curr = $end;
    while (defined $curr) {
        unshift @path, $curr;
        $curr = $prev->{$curr};
    }
    return @path if $path[0] eq $start;
    return ();
}

package main;

# Undirected graph
my $g = Graph->new;
$g->add_edge("A", "B");
$g->add_edge("A", "C");
$g->add_edge("B", "D");
$g->add_edge("C", "D");
$g->add_edge("D", "E");
$g->add_edge("C", "F");

printf "Vertices: %s\n", join(", ", $g->vertices);
printf "BFS from A: %s\n", join(", ", $g->bfs("A"));
printf "DFS from A: %s\n", join(", ", $g->dfs("A"));

# Weighted graph (Dijkstra)
my $wg = Graph->new(weighted => 1);
$wg->add_edge("A", "B", 4);
$wg->add_edge("A", "C", 2);
$wg->add_edge("B", "C", 1);
$wg->add_edge("B", "D", 5);
$wg->add_edge("C", "D", 8);
$wg->add_edge("C", "E", 10);
$wg->add_edge("D", "E", 2);

my ($dist, $prev) = $wg->dijkstra("A");
print "\nShortest paths from A:\n";
for my $v (sort keys %$dist) {
    my @path = $wg->path("A", $v, $prev);
    printf "  A -> %s: %d (path: %s)\n", $v, $dist->{$v}, join(" -> ", @path);
}
```

---

## Step 156: Hash-based Algorithms

```perl
#!/usr/bin/perl
use strict;
use warnings;
use List::Util qw(sum);

# =====================
# LRU Cache
# =====================

package LRUCache;

sub new {
    my ($class, $capacity) = @_;
    return bless {
        capacity => $capacity,
        cache    => {},
        order    => [],
    }, $class;
}

sub get {
    my ($self, $key) = @_;
    return undef unless exists $self->{cache}{$key};
    
    # Move to front (most recently used)
    $self->{order} = [grep { $_ ne $key } @{$self->{order}}];
    unshift @{$self->{order}}, $key;
    
    return $self->{cache}{$key};
}

sub put {
    my ($self, $key, $value) = @_;
    
    if (exists $self->{cache}{$key}) {
        # Update existing
        $self->{order} = [grep { $_ ne $key } @{$self->{order}}];
    } elsif (@{$self->{order}} >= $self->{capacity}) {
        # Evict LRU (last)
        my $evict = pop @{$self->{order}};
        delete $self->{cache}{$evict};
        printf "  [LRU evict] %s\n", $evict;
    }
    
    $self->{cache}{$key} = $value;
    unshift @{$self->{order}}, $key;
}

sub size { scalar keys %{$_[0]->{cache}} }

package main;

my $lru = LRUCache->new(3);
$lru->put("a", 1);
$lru->put("b", 2);
$lru->put("c", 3);
printf "a=%s\n", $lru->get("a") // "undef";
$lru->put("d", 4);   # evicts b (LRU)
printf "b=%s\n", $lru->get("b") // "undef";

# =====================
# Trie
# =====================

package Trie;

sub new { bless { children => {}, is_end => 0 }, shift }

sub insert {
    my ($self, $word) = @_;
    my $node = $self;
    for my $ch (split //, $word) {
        $node->{children}{$ch} //= Trie->new;
        $node = $node->{children}{$ch};
    }
    $node->{is_end} = 1;
}

sub search {
    my ($self, $word) = @_;
    my $node = $self;
    for my $ch (split //, $word) {
        return 0 unless exists $node->{children}{$ch};
        $node = $node->{children}{$ch};
    }
    return $node->{is_end};
}

sub starts_with {
    my ($self, $prefix) = @_;
    my $node = $self;
    for my $ch (split //, $prefix) {
        return 0 unless exists $node->{children}{$ch};
        $node = $node->{children}{$ch};
    }
    return 1;
}

sub all_words {
    my ($self, $prefix) = @_;
    $prefix //= "";
    my @words;
    $self->_all_words($prefix, \@words);
    return @words;
}

sub _all_words {
    my ($self, $prefix, $words) = @_;
    push @$words, $prefix if $self->{is_end};
    for my $ch (sort keys %{$self->{children}}) {
        $self->{children}{$ch}->_all_words($prefix . $ch, $words);
    }
}

package main;

my $trie = Trie->new;
$trie->insert($_) for qw(perl python php ruby rust racket go java javascript);

print "\nTrie search:\n";
printf "  perl:   %s\n", $trie->search("perl") ? "found" : "not found";
printf "  c++:    %s\n", $trie->search("c++") ? "found" : "not found";
printf "  starts 'r': %s\n", $trie->starts_with("r") ? "yes" : "no";

my @r_words = $trie->all_words("r");
printf "  words starting with r: %s\n", join(", ", @r_words);

# =====================
# Bloom Filter (probabilistic)
# =====================

package BloomFilter;

sub new {
    my ($class, %opts) = @_;
    my $size = $opts{size} // 1000;
    return bless {
        size   => $size,
        bits   => [(0) x $size],
        hashes => $opts{hashes} // 3,
    }, $class;
}

sub _hash {
    my ($self, $item, $seed) = @_;
    my $h = $seed;
    $h = ($h * 31 + ord) % $self->{size} for split //, $item;
    return $h;
}

sub add {
    my ($self, $item) = @_;
    for my $i (1..$self->{hashes}) {
        my $pos = $self->_hash($item, $i * 7919);
        $self->{bits}[$pos] = 1;
    }
}

sub might_contain {
    my ($self, $item) = @_;
    for my $i (1..$self->{hashes}) {
        my $pos = $self->_hash($item, $i * 7919);
        return 0 unless $self->{bits}[$pos];
    }
    return 1;
}

package main;

my $bloom = BloomFilter->new(size => 2000, hashes => 5);
$bloom->add($_) for qw(perl python ruby go rust);

printf "\nBloom filter:\n";
for my $word (qw(perl javascript ruby c++ python)) {
    printf "  %-15s might_contain: %s\n",
        $word, $bloom->might_contain($word) ? "yes" : "no";
}
```

---

## Step 157: Functional Data Structures

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# Immutable-style operations
# =====================

# อย่า mutate — return new value แทน

sub array_push {
    my ($arr, @items) = @_;
    return [@$arr, @items];   # new array
}

sub array_filter {
    my ($arr, $pred) = @_;
    return [grep { $pred->($_) } @$arr];
}

sub array_map {
    my ($arr, $fn) = @_;
    return [map { $fn->($_) } @$arr];
}

sub array_reduce {
    my ($arr, $fn, $init) = @_;
    my $acc = $init;
    $acc = $fn->($acc, $_) for @$arr;
    return $acc;
}

my $nums = [1..10];
my $evens   = array_filter($nums, sub { $_[0] % 2 == 0 });
my $doubled = array_map($evens, sub { $_[0] * 2 });
my $total   = array_reduce($doubled, sub { $_[0] + $_[1] }, 0);

printf "evens: %s\n", join(", ", @$evens);
printf "doubled: %s\n", join(", ", @$doubled);
printf "total: %d\n", $total;

# =====================
# Lazy evaluation / generators
# =====================

package LazyList;

sub new {
    my ($class, $head, $tail_fn) = @_;
    return bless {
        head    => $head,
        tail_fn => $tail_fn,
        _tail   => undef,
    }, $class;
}

sub head { $_[0]->{head} }

sub tail {
    my $self = shift;
    unless ($self->{_tail}) {
        $self->{_tail} = $self->{tail_fn}->();
    }
    return $self->{_tail};
}

sub take {
    my ($self, $n) = @_;
    my @result;
    my $curr = $self;
    while ($n-- > 0 && defined $curr) {
        push @result, $curr->head;
        $curr = $curr->tail;
    }
    return @result;
}

package main;

# Infinite list of natural numbers
sub naturals_from {
    my $n = shift;
    return LazyList->new($n, sub { naturals_from($n + 1) });
}

# Infinite list of Fibonacci
sub fibs_from {
    my ($a, $b) = @_;
    return LazyList->new($a, sub { fibs_from($b, $a + $b) });
}

my $nats = naturals_from(1);
print "First 10 naturals: ", join(", ", $nats->take(10)), "\n";

my $fibs = fibs_from(0, 1);
print "First 15 fibs:     ", join(", ", $fibs->take(15)), "\n";

# =====================
# Persistent (immutable) data
# =====================

# Hash update without mutation
sub hash_set {
    my ($hash, $key, $val) = @_;
    return { %$hash, $key => $val };
}

sub hash_delete {
    my ($hash, $key) = @_;
    my %new = %$hash;
    delete $new{$key};
    return \%new;
}

my $state1 = { name => "Alice", age => 28 };
my $state2 = hash_set($state1, "age", 29);
my $state3 = hash_set($state2, "city", "Bangkok");

printf "state1: %s\n", join(", ", map { "$_=$state1->{$_}" } sort keys %$state1);
printf "state2: %s\n", join(", ", map { "$_=$state2->{$_}" } sort keys %$state2);
printf "state3: %s\n", join(", ", map { "$_=$state3->{$_}" } sort keys %$state3);
```

---

## Step 158: Set Operations

```perl
#!/usr/bin/perl
use strict;
use warnings;
use List::Util qw(uniq);

# =====================
# Set implementation
# =====================

package Set;

sub new {
    my ($class, @items) = @_;
    my %h;
    $h{$_} = 1 for @items;
    return bless \%h, $class;
}

sub add      { $_[0]->{$_[1]} = 1 }
sub remove   { delete $_[0]->{$_[1]} }
sub contains { exists $_[0]->{$_[1]} }
sub size     { scalar keys %{$_[0]} }
sub members  { sort keys %{$_[0]} }
sub is_empty { !%{$_[0]} }

sub union {
    my ($a, $b) = @_;
    return Set->new($a->members, $b->members);
}

sub intersection {
    my ($a, $b) = @_;
    return Set->new(grep { $b->contains($_) } $a->members);
}

sub difference {
    my ($a, $b) = @_;
    return Set->new(grep { !$b->contains($_) } $a->members);
}

sub symmetric_difference {
    my ($a, $b) = @_;
    return $a->union($b)->difference($a->intersection($b));
}

sub is_subset {
    my ($a, $b) = @_;   # is $a subset of $b?
    return !grep { !$b->contains($_) } $a->members;
}

sub is_equal {
    my ($a, $b) = @_;
    return $a->size == $b->size && $a->is_subset($b);
}

sub to_string {
    my $self = shift;
    return "{" . join(", ", $self->members) . "}";
}

package main;

my $A = Set->new(qw(1 2 3 4 5));
my $B = Set->new(qw(3 4 5 6 7));
my $C = Set->new(qw(1 2 3));

printf "A = %s\n", $A->to_string;
printf "B = %s\n", $B->to_string;
printf "C = %s\n", $C->to_string;
printf "\nA ∪ B = %s\n", $A->union($B)->to_string;
printf "A ∩ B = %s\n", $A->intersection($B)->to_string;
printf "A - B = %s\n", $A->difference($B)->to_string;
printf "A △ B = %s\n", $A->symmetric_difference($B)->to_string;
printf "\nC ⊆ A? %s\n", $C->is_subset($A) ? "yes" : "no";
printf "A ⊆ C? %s\n", $A->is_subset($C) ? "yes" : "no";

# =====================
# Multiset (bag)
# =====================

package Multiset;

sub new {
    my ($class, @items) = @_;
    my %counts;
    $counts{$_}++ for @items;
    return bless \%counts, $class;
}

sub add      { $_[0]->{$_[1]}++ }
sub remove   { $_[0]->{$_[1]}-- if $_[0]->{$_[1]}; delete $_[0]->{$_[1]} unless $_[0]->{$_[1]} }
sub count    { $_[0]->{$_[1]} // 0 }
sub members  { sort keys %{$_[0]} }
sub distinct { scalar keys %{$_[0]} }
sub total    { my $t=0; $t += $_ for values %{$_[0]}; $t }

package main;

my $bag = Multiset->new(qw(apple banana apple cherry apple banana));
printf "\nMultiset:\n";
printf "  %s: %d\n", $_, $bag->count($_) for $bag->members;
printf "distinct=%d, total=%d\n", $bag->distinct, $bag->total;
```

---

## Step 159: Sorting Algorithms

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Time::HiRes qw(gettimeofday tv_interval);

# =====================
# Various sort algorithms
# =====================

sub quicksort {
    my @arr = @_;
    return @arr if @arr <= 1;
    
    my $pivot = $arr[int(@arr/2)];
    my @less    = grep { $_ < $pivot } @arr;
    my @equal   = grep { $_ == $pivot } @arr;
    my @greater = grep { $_ > $pivot } @arr;
    
    return (quicksort(@less), @equal, quicksort(@greater));
}

sub mergesort {
    my @arr = @_;
    return @arr if @arr <= 1;
    
    my $mid = int @arr / 2;
    my @left  = mergesort(@arr[0..$mid-1]);
    my @right = mergesort(@arr[$mid..$#arr]);
    
    return merge(\@left, \@right);
}

sub merge {
    my ($left, $right) = @_;
    my @result;
    
    while (@$left && @$right) {
        if ($left->[0] <= $right->[0]) {
            push @result, shift @$left;
        } else {
            push @result, shift @$right;
        }
    }
    
    return (@result, @$left, @$right);
}

sub heapsort {
    my @arr = @_;
    my $n = @arr;
    
    # Build heap
    for (my $i = int($n/2) - 1; $i >= 0; $i--) {
        heapify(\@arr, $n, $i);
    }
    
    # Extract elements
    for (my $i = $n-1; $i > 0; $i--) {
        @arr[0, $i] = @arr[$i, 0];
        heapify(\@arr, $i, 0);
    }
    
    return @arr;
}

sub heapify {
    my ($arr, $n, $i) = @_;
    my $largest = $i;
    my $left  = 2 * $i + 1;
    my $right = 2 * $i + 2;
    
    $largest = $left  if $left  < $n && $arr->[$left]  > $arr->[$largest];
    $largest = $right if $right < $n && $arr->[$right] > $arr->[$largest];
    
    if ($largest != $i) {
        @{$arr}[$i, $largest] = @{$arr}[$largest, $i];
        heapify($arr, $n, $largest);
    }
}

# =====================
# Benchmark
# =====================

my @data = map { int rand 1000 } 1..200;

my %algorithms = (
    perl_sort  => sub { sort { $a <=> $b } @_ },
    quicksort  => \&quicksort,
    mergesort  => \&mergesort,
    heapsort   => \&heapsort,
);

printf "%-15s %10s\n", "Algorithm", "Time(ms)";
print "-" x 30 . "\n";

for my $name (sort keys %algorithms) {
    my $t0 = [gettimeofday];
    for (1..100) {
        $algorithms{$name}->(@data);
    }
    my $elapsed = tv_interval($t0);
    printf "%-15s %10.2f\n", $name, $elapsed * 1000 / 100;
}

# Verify all produce same result
my @expected = sort { $a <=> $b } @data;
for my $name (keys %algorithms) {
    my @sorted = $algorithms{$name}->(@data);
    print "$name: ", (@sorted ~~ @expected ? "correct" : "WRONG!"), "\n";
}
```

---

## Step 160: โปรแกรมสรุป — Data Structure Library

```perl
#!/usr/bin/perl
#
# ds_demo.pl — ตัวอย่าง Data Structures
#

use strict;
use warnings;

# =====================
# Queue-based BFS maze solver
# =====================

sub solve_maze {
    my ($maze) = @_;
    
    my $rows = scalar @$maze;
    my $cols = scalar @{$maze->[0]};
    
    my ($start, $end);
    for my $r (0..$rows-1) {
        for my $c (0..$cols-1) {
            $start = [$r, $c] if $maze->[$r][$c] eq 'S';
            $end   = [$r, $c] if $maze->[$r][$c] eq 'E';
        }
    }
    
    return undef unless $start && $end;
    
    my @queue = ([$start, []]);
    my %visited;
    $visited{"$start->[0],$start->[1]"} = 1;
    
    my @dirs = ([-1,0],[1,0],[0,-1],[0,1]);
    
    while (@queue) {
        my ($pos, $path) = @{shift @queue};
        my ($r, $c) = @$pos;
        
        if ($r == $end->[0] && $c == $end->[1]) {
            return [@$path, [$r, $c]];
        }
        
        for my $dir (@dirs) {
            my ($nr, $nc) = ($r + $dir->[0], $c + $dir->[1]);
            next if $nr < 0 || $nr >= $rows || $nc < 0 || $nc >= $cols;
            next if $maze->[$nr][$nc] eq '#';
            next if $visited{"$nr,$nc"}++;
            push @queue, [[$nr, $nc], [@$path, [$r, $c]]];
        }
    }
    
    return undef;  # no path found
}

my @maze = (
    [qw(. . . # . . .)],
    [qw(. # . # . # .)],
    [qw(S # . . . # E)],
    [qw(. # # # . # .)],
    [qw(. . . . . . .)],
);

my $path = solve_maze(\@maze);
if ($path) {
    print "Maze solution found! Length: ", scalar @$path, " steps\n";
    
    # Draw solution
    my %path_set = map { "$_->[0],$_->[1]" => 1 } @$path;
    for my $r (0..$#maze) {
        for my $c (0..$#{$maze[$r]}) {
            my $cell = $maze[$r][$c];
            if ($path_set{"$r,$c"} && $cell eq '.') {
                print "*";
            } else {
                print $cell;
            }
        }
        print "\n";
    }
} else {
    print "No path found!\n";
}

# =====================
# Priority Queue job scheduler
# =====================

print "\n=== Job Scheduler ===\n";

my $scheduler = PriorityQueue->new;

sub PriorityQueue::new { bless { heap => [] }, shift }

sub PriorityQueue::submit {
    my ($self, $priority, $job) = @_;
    push @{$self->{heap}}, [$priority, $job];
    $self->_sift_up($#{$self->{heap}});
}

sub PriorityQueue::next_job {
    my $self = shift;
    return undef unless @{$self->{heap}};
    my $top = $self->{heap}[0];
    my $last = pop @{$self->{heap}};
    if (@{$self->{heap}}) {
        $self->{heap}[0] = $last;
        $self->_sift_down(0);
    }
    return $top;
}

sub PriorityQueue::_sift_up {
    my ($self, $i) = @_;
    my $h = $self->{heap};
    while ($i > 0) {
        my $p = int(($i-1)/2);
        last if $h->[$p][0] >= $h->[$i][0];
        @{$h}[$p,$i] = @{$h}[$i,$p];
        $i = $p;
    }
}

sub PriorityQueue::_sift_down {
    my ($self, $i) = @_;
    my $h = $self->{heap};
    my $n = scalar @$h;
    while (1) {
        my ($l, $r) = (2*$i+1, 2*$i+2);
        my $max = $i;
        $max = $l if $l < $n && $h->[$l][0] > $h->[$max][0];
        $max = $r if $r < $n && $h->[$r][0] > $h->[$max][0];
        last if $max == $i;
        @{$h}[$i,$max] = @{$h}[$max,$i];
        $i = $max;
    }
}

sub PriorityQueue::is_empty { !@{$_[0]->{heap}} }

$scheduler->submit(3, "Send email notification");
$scheduler->submit(10, "CRITICAL: System alert");
$scheduler->submit(5, "Generate report");
$scheduler->submit(1, "Cleanup temp files");
$scheduler->submit(8, "Backup database");

print "Processing jobs by priority:\n";
my $step = 1;
while (!$scheduler->is_empty) {
    my ($pri, $job) = @{$scheduler->next_job};
    printf "  %d. [priority=%2d] %s\n", $step++, $pri, $job;
}
```

---

## สรุป Part 16

ใน Part นี้คุณได้เรียนรู้:
- ✅ References ขั้นสูงและ weak references
- ✅ Stack, Queue, Deque
- ✅ Priority Queue (Heap)
- ✅ Linked List (singly, doubly)
- ✅ Binary Search Tree
- ✅ N-ary Tree
- ✅ Graph และ BFS/DFS
- ✅ Dijkstra's algorithm
- ✅ LRU Cache
- ✅ Trie
- ✅ Bloom Filter
- ✅ Set operations
- ✅ Sorting algorithms
- ✅ Functional data structures
- ✅ Lazy evaluation

**ถัดไป: [Part 17 — CGI Programming](part_17.md)**
