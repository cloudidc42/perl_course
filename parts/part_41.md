# Part 41: Caching Strategies & Performance
## Steps 401-410: In-memory Cache, Redis-style, Cache Patterns

---

## Step 401: LRU Cache

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Cache::LRU;

sub new {
    my ($class, %opts) = @_;
    return bless {
        capacity => $opts{capacity} // 100,
        ttl      => $opts{ttl}      // 0,
        _hash    => {},
        _head    => undef,   # most recently used
        _tail    => undef,   # least recently used
        _size    => 0,
        stats    => { hits=>0, misses=>0, evictions=>0, sets=>0 },
    }, $class;
}

sub get {
    my ($self, $key) = @_;
    my $node = $self->{_hash}{$key};
    
    unless ($node) {
        $self->{stats}{misses}++;
        return undef;
    }
    
    # Check TTL
    if ($self->{ttl} && $node->{expires} && $node->{expires} < time()) {
        $self->_remove_node($node);
        delete $self->{_hash}{$key};
        $self->{_size}--;
        $self->{stats}{misses}++;
        return undef;
    }
    
    $self->{stats}{hits}++;
    $self->_move_to_head($node);
    return $node->{value};
}

sub set {
    my ($self, $key, $value, %opts) = @_;
    my $ttl = $opts{ttl} // $self->{ttl};
    
    if (my $node = $self->{_hash}{$key}) {
        $node->{value}   = $value;
        $node->{expires} = $ttl ? time() + $ttl : undef;
        $self->_move_to_head($node);
        $self->{stats}{sets}++;
        return $self;
    }
    
    # Evict if full
    if ($self->{_size} >= $self->{capacity}) {
        my $lru = $self->{_tail};
        $self->_remove_node($lru);
        delete $self->{_hash}{$lru->{key}};
        $self->{_size}--;
        $self->{stats}{evictions}++;
    }
    
    my $node = {
        key     => $key,
        value   => $value,
        prev    => undef,
        next    => undef,
        expires => $ttl ? time() + $ttl : undef,
    };
    
    $self->{_hash}{$key} = $node;
    $self->_add_to_head($node);
    $self->{_size}++;
    $self->{stats}{sets}++;
    return $self;
}

sub delete {
    my ($self, $key) = @_;
    my $node = $self->{_hash}{$key} or return 0;
    $self->_remove_node($node);
    delete $self->{_hash}{$key};
    $self->{_size}--;
    return 1;
}

sub exists {
    my ($self, $key) = @_;
    my $node = $self->{_hash}{$key} or return 0;
    return 0 if $self->{ttl} && $node->{expires} && $node->{expires} < time();
    return 1;
}

sub keys_list {
    my $self = shift;
    my @keys;
    my $node = $self->{_head};
    while ($node) {
        push @keys, $node->{key};
        $node = $node->{next};
    }
    return @keys;
}

sub size    { $_[0]->{_size} }
sub stats   { %{$_[0]->{stats}} }
sub hit_rate {
    my $s = $_[0]->{stats};
    my $total = $s->{hits} + $s->{misses};
    return $total ? $s->{hits}/$total : 0;
}

sub get_or_set {
    my ($self, $key, $builder, %opts) = @_;
    my $val = $self->get($key);
    unless (defined $val) {
        $val = $builder->();
        $self->set($key, $val, %opts) if defined $val;
    }
    return $val;
}

sub _add_to_head {
    my ($self, $node) = @_;
    $node->{next} = $self->{_head};
    $node->{prev} = undef;
    $self->{_head}{prev} = $node if $self->{_head};
    $self->{_head} = $node;
    $self->{_tail} //= $node;
}

sub _remove_node {
    my ($self, $node) = @_;
    $node->{prev}{next} = $node->{next} if $node->{prev};
    $node->{next}{prev} = $node->{prev} if $node->{next};
    $self->{_head} = $node->{next} if $self->{_head} == $node;
    $self->{_tail} = $node->{prev} if $self->{_tail} == $node;
}

sub _move_to_head {
    my ($self, $node) = @_;
    $self->_remove_node($node);
    $self->_add_to_head($node);
}
}

package main;

printf "=== LRU Cache ===\n\n";

my $cache = Cache::LRU->new(capacity => 5);

# Fill cache
printf "Adding items:\n";
for my $i (1..5) {
    $cache->set("key$i", "value$i");
    printf "  set key%d | size=%d | order=[%s]\n", $i, $cache->size,
        join(",", $cache->keys_list);
}

# Access key2 (makes it recent)
$cache->get("key2");
printf "\nAfter get(key2): [%s]\n", join(",", $cache->keys_list);

# Add key6 — should evict key1 (LRU)
$cache->set("key6", "value6");
printf "After set(key6): [%s]\n\n", join(",", $cache->keys_list);

# Miss
my $v = $cache->get("key1");
printf "Get key1 (evicted): %s\n", defined $v ? $v : "MISS";

# Hit
$v = $cache->get("key2");
printf "Get key2: %s\n\n", defined $v ? $v : "MISS";

# get_or_set
printf "get_or_set:\n";
my $computed = $cache->get_or_set("expensive_key", sub {
    printf "  (computing...)\n";
    return "computed_result_42";
});
printf "  First call:  %s\n", $computed;

my $cached = $cache->get_or_set("expensive_key", sub { "should-not-run" });
printf "  Second call: %s\n\n", $cached;

# Stats
my %stats = $cache->stats;
printf "Stats:\n";
printf "  hits=%d misses=%d sets=%d evictions=%d\n",
    $stats{hits}, $stats{misses}, $stats{sets}, $stats{evictions};
printf "  hit_rate=%.1f%%\n", $cache->hit_rate * 100;
```

---

## Step 402: TTL Cache with Multiple Eviction Policies

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Cache::Policy;

sub new {
    my ($class, %opts) = @_;
    return bless {
        policy   => $opts{policy}   // "lru",  # lru, lfu, fifo, random
        capacity => $opts{capacity} // 100,
        ttl      => $opts{ttl}      // 0,
        _data    => {},
        _access  => {},
        _freq    => {},
        _insert_order => [],
        stats    => { hits=>0, misses=>0, evictions=>0 },
    }, $class;
}

sub set {
    my ($self, $key, $value, %opts) = @_;
    my $ttl = $opts{ttl} // $self->{ttl};
    
    unless (exists $self->{_data}{$key}) {
        $self->_evict if scalar(keys %{$self->{_data}}) >= $self->{capacity};
        push @{$self->{_insert_order}}, $key;
    }
    
    $self->{_data}{$key}   = $value;
    $self->{_access}{$key} = time();
    $self->{_freq}{$key}   = ($self->{_freq}{$key} // 0) + 1;
    $self->{_expire}{$key} = $ttl ? time() + $ttl : undef;
}

sub get {
    my ($self, $key) = @_;
    
    unless (exists $self->{_data}{$key}) {
        $self->{stats}{misses}++;
        return undef;
    }
    
    if ($self->{_expire}{$key} && $self->{_expire}{$key} < time()) {
        $self->_remove($key);
        $self->{stats}{misses}++;
        return undef;
    }
    
    $self->{stats}{hits}++;
    $self->{_access}{$key} = time();
    $self->{_freq}{$key}++;
    return $self->{_data}{$key};
}

sub _evict {
    my $self = shift;
    my $policy = $self->{policy};
    my @keys = keys %{$self->{_data}};
    return unless @keys;
    
    my $victim;
    if ($policy eq "lru") {
        $victim = (sort { $self->{_access}{$a} <=> $self->{_access}{$b} } @keys)[0];
    } elsif ($policy eq "lfu") {
        $victim = (sort { $self->{_freq}{$a} <=> $self->{_freq}{$b} } @keys)[0];
    } elsif ($policy eq "fifo") {
        $victim = $self->{_insert_order}[0];
    } elsif ($policy eq "random") {
        $victim = $keys[rand @keys];
    }
    
    $self->_remove($victim);
    $self->{stats}{evictions}++;
}

sub _remove {
    my ($self, $key) = @_;
    delete $self->{_data}{$key};
    delete $self->{_access}{$key};
    delete $self->{_freq}{$key};
    delete $self->{_expire}{$key};
    @{$self->{_insert_order}} = grep { $_ ne $key } @{$self->{_insert_order}};
}

sub size  { scalar keys %{$_[0]->{_data}} }
sub stats { %{$_[0]->{stats}} }
sub hit_rate {
    my $s = $_[0]->{stats};
    my $t = $s->{hits}+$s->{misses}; return $t ? sprintf "%.1f%%", 100*$s->{hits}/$t : "n/a";
}
}

package main;

printf "=== Cache Eviction Policies ===\n\n";

# Compare policies
for my $policy (qw(lru lfu fifo random)) {
    my $cache = Cache::Policy->new(policy => $policy, capacity => 3);
    
    # Fill
    $cache->set("a", 1); $cache->set("b", 2); $cache->set("c", 3);
    
    # Access patterns
    $cache->get("a"); $cache->get("a"); $cache->get("b");  # a most accessed, b moderate, c cold
    
    # Trigger eviction with new key
    $cache->set("d", 4);
    
    my @remaining = sort grep { defined $cache->get($_) } qw(a b c d);
    printf "  %-8s after eviction: [%s]\n", $policy, join(",", @remaining);
}

printf "\n--- TTL Demo ---\n";
{
    my $cache = Cache::Policy->new(policy => "lru", capacity => 100, ttl => 5);
    
    # Use fake time via closure
    my $fake_time = 1000;
    
    # Monkey-patch time for demo purposes
    $cache->set("short", "expires-soon");
    $cache->set("long",  "lives-long",   ttl => 100);
    
    printf "  Before expire: short=%s long=%s\n",
        $cache->get("short") // "MISS",
        $cache->get("long")  // "MISS";
    
    # Manually expire by reaching into internals
    $cache->{_expire}{short} = time() - 1;
    
    printf "  After expire:  short=%s long=%s\n",
        $cache->get("short") // "MISS",
        $cache->get("long")  // "MISS";
}
```

---

## Step 403: Cache Warming & Preloading

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Cache::Warm;

sub new {
    my ($class, %opts) = @_;
    return bless {
        cache   => {},
        loader  => $opts{loader},
        ttl     => $opts{ttl} // 300,
        stats   => { warm=>0, cold=>0, reloads=>0 },
    }, $class;
}

sub get {
    my ($self, $key) = @_;
    if (exists $self->{cache}{$key} && $self->{cache}{$key}{exp} > time()) {
        $self->{stats}{warm}++;
        return $self->{cache}{$key}{value};
    }
    $self->{stats}{cold}++;
    return $self->_load($key);
}

sub _load {
    my ($self, $key) = @_;
    my $val = $self->{loader}->($key);
    if (defined $val) {
        $self->{cache}{$key} = { value => $val, exp => time() + $self->{ttl} };
        $self->{stats}{reloads}++;
    }
    return $val;
}

sub warm_all {
    my ($self, @keys) = @_;
    my $loaded = 0;
    for my $key (@keys) {
        my $val = $self->{loader}->($key);
        if (defined $val) {
            $self->{cache}{$key} = { value => $val, exp => time() + $self->{ttl} };
            $loaded++;
        }
    }
    return $loaded;
}

sub warm_batch {
    my ($self, $batch_loader, @keys) = @_;
    my %results = $batch_loader->(@keys);
    my $loaded = 0;
    for my $key (@keys) {
        if (defined $results{$key}) {
            $self->{cache}{$key} = { value => $results{$key}, exp => time()+$self->{ttl} };
            $loaded++;
        }
    }
    return $loaded;
}

sub invalidate        { delete $_[0]->{cache}{$_[1]}; 1 }
sub invalidate_prefix { my ($self,$pfx)=@_; delete $self->{cache}{$_} for grep{/^\Q$pfx/} keys %{$self->{cache}} }
sub stats             { %{$_[0]->{stats}} }
sub hit_rate {
    my $s = $_[0]->{stats};
    my $t = $s->{warm}+$s->{cold}; return $t ? sprintf "%.1f%%", 100*$s->{warm}/$t : "n/a";
}
}

package main;

printf "=== Cache Warming ===\n\n";

# Simulate a database
my %db = (
    "user:1" => { id=>1, name=>"Alice", role=>"admin" },
    "user:2" => { id=>2, name=>"Bob",   role=>"user"  },
    "user:3" => { id=>3, name=>"Carol", role=>"user"  },
    "config:theme" => "dark",
    "config:lang"  => "en",
);

my $db_calls = 0;
my $cache = Cache::Warm->new(
    ttl    => 60,
    loader => sub {
        $db_calls++;
        printf "  [DB LOAD] %s\n", $_[0];
        return $db{$_[0]};
    }
);

# Cold start — all misses
printf "Cold access:\n";
$cache->get("user:1");
$cache->get("user:2");
$cache->get("user:1");  # Should be warm now
printf "DB calls: %d\n\n", $db_calls;

# Warm the cache ahead of time
printf "Warming cache:\n";
$db_calls = 0;
my $warmed = $cache->warm_all("user:1","user:2","user:3","config:theme","config:lang");
printf "Warmed %d keys, DB calls: %d\n\n", $warmed, $db_calls;

# Now all hot
printf "Warm access:\n";
$db_calls = 0;
$cache->get("user:1");
$cache->get("user:2");
$cache->get("user:3");
$cache->get("config:theme");
printf "DB calls after warm: %d\n", $db_calls;

my %stats = $cache->stats;
printf "Stats: warm=%d cold=%d hit_rate=%s\n\n",
    $stats{warm}, $stats{cold}, $cache->hit_rate;

# Invalidation
printf "Invalidation:\n";
$cache->invalidate("user:1");
printf "  After invalidate user:1: %s\n",
    $cache->{cache}{"user:1"} ? "still cached (bad)" : "removed OK";

$cache->invalidate_prefix("config:");
my @remaining = grep { exists $cache->{cache}{$_} } keys %db;
printf "  After invalidate config:*: remaining=%s\n", join(",", sort @remaining);
```

---

## Step 404: Write-through & Write-behind Cache

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Cache::WriteThrough;
# Writes go to cache AND database synchronously

my @db_writes;

sub new {
    my ($class, %opts) = @_;
    return bless {
        cache  => {},
        writer => $opts{writer} // sub {},
        reader => $opts{reader} // sub {},
        stats  => { cache_writes=>0, db_writes=>0, cache_reads=>0, db_reads=>0 },
    }, $class;
}

sub set {
    my ($self, $key, $value) = @_;
    $self->{cache}{$key} = $value;
    $self->{writer}->($key, $value);  # synchronous write
    $self->{stats}{cache_writes}++;
    $self->{stats}{db_writes}++;
}

sub get {
    my ($self, $key) = @_;
    if (exists $self->{cache}{$key}) {
        $self->{stats}{cache_reads}++;
        return $self->{cache}{$key};
    }
    my $val = $self->{reader}->($key);
    $self->{cache}{$key} = $val if defined $val;
    $self->{stats}{db_reads}++;
    return $val;
}

sub stats { %{$_[0]->{stats}} }
}

{
package Cache::WriteBehind;
# Writes go to cache immediately, flushed to DB asynchronously

sub new {
    my ($class, %opts) = @_;
    return bless {
        cache      => {},
        dirty      => {},
        writer     => $opts{writer}  // sub {},
        flush_size => $opts{flush_size} // 10,
        stats      => { cache_writes=>0, db_writes=>0, flushes=>0 },
    }, $class;
}

sub set {
    my ($self, $key, $value) = @_;
    $self->{cache}{$key}  = $value;
    $self->{dirty}{$key}  = { value => $value, ts => time() };
    $self->{stats}{cache_writes}++;
    
    # Auto flush if dirty set is large
    $self->flush if scalar(keys %{$self->{dirty}}) >= $self->{flush_size};
}

sub get {
    my ($self, $key) = @_;
    return $self->{cache}{$key};
}

sub flush {
    my $self = shift;
    my %dirty = %{$self->{dirty}};
    $self->{dirty} = {};
    
    for my $key (keys %dirty) {
        $self->{writer}->($key, $dirty{$key}{value});
        $self->{stats}{db_writes}++;
    }
    $self->{stats}{flushes}++;
    return scalar keys %dirty;
}

sub dirty_count { scalar keys %{$_[0]->{dirty}} }
sub stats { %{$_[0]->{stats}} }
}

package main;

printf "=== Write-through vs Write-behind ===\n\n";

my @wt_writes;
my %backend;

my $wt = Cache::WriteThrough->new(
    writer => sub { push @wt_writes, "$_[0]=$_[1]"; $backend{$_[0]} = $_[1] },
    reader => sub { $backend{$_[0]} },
);

printf "Write-through:\n";
$wt->set("user:1", { name => "Alice" });
$wt->set("user:2", { name => "Bob" });
$wt->set("user:1", { name => "Alice Updated" });

printf "  DB writes: %d (immediate)\n", scalar @wt_writes;
printf "  Cache vs DB consistent: %s\n",
    $wt->{cache}{"user:1"}{name} eq $backend{"user:1"}{name} ? "YES" : "NO";

# Write-behind
my @wb_writes;
my $wb = Cache::WriteBehind->new(
    writer     => sub { push @wb_writes, "$_[0]"; $backend{$_[0]} = $_[1] },
    flush_size => 3,
);

printf "\nWrite-behind:\n";
$wb->set("post:1", { title => "Hello" });
$wb->set("post:2", { title => "World" });
printf "  After 2 writes: dirty=%d db_writes=%d\n", $wb->dirty_count, 0;

$wb->set("post:3", { title => "Perl" });  # Auto-flushes at 3
printf "  After 3 writes (auto-flush): dirty=%d db_writes=%d\n",
    $wb->dirty_count, (my %s = $wb->stats)[qw(db_writes)];

# Explicitly flush remainder
$wb->set("post:4", { title => "Rocks" });
my $flushed = $wb->flush;
printf "  Manual flush: %d items written\n", $flushed;

my %wt_stats = $wt->stats;
my %wb_stats = $wb->stats;
printf "\nWrite-through: cache_writes=%d db_writes=%d\n",
    $wt_stats{cache_writes}, $wt_stats{db_writes};
printf "Write-behind:  cache_writes=%d db_writes=%d flushes=%d\n",
    $wb_stats{cache_writes}, $wb_stats{db_writes}, $wb_stats{flushes};
```

---

## Step 405: Cache-Aside Pattern

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Cache::Aside;
# Application manages cache explicitly

sub new {
    my ($class, %opts) = @_;
    return bless {
        _store   => {},
        ttl      => $opts{ttl} // 300,
        compress => $opts{compress} // 0,
        stats    => { hits=>0, misses=>0, sets=>0, deletes=>0 },
    }, $class;
}

sub get    { $_[0]->_get_entry($_[1]) ? do { $_[0]->{stats}{hits}++; $_[0]->{_store}{$_[1]}{value} } : do { $_[0]->{stats}{misses}++; undef } }
sub set    { my($self,$k,$v,%o)=@_; $self->{_store}{$k}={value=>$v,exp=>time()+($o{ttl}//$self->{ttl}),tags=>$o{tags}//[]}; $self->{stats}{sets}++ }
sub delete { my($self,$k)=@_; delete $self->{_store}{$k}; $self->{stats}{deletes}++ }

sub _get_entry {
    my ($self, $k) = @_;
    return 0 unless exists $self->{_store}{$k};
    return 0 if $self->{_store}{$k}{exp} < time();
    return 1;
}

sub get_or_load {
    my ($self, $key, $loader, %opts) = @_;
    my $val = $self->get($key);
    return $val if defined $val;
    
    $val = $loader->();
    $self->set($key, $val, %opts) if defined $val;
    return $val;
}

sub invalidate_tags {
    my ($self, @tags) = @_;
    my %tag_set = map { $_ => 1 } @tags;
    my $removed = 0;
    for my $key (keys %{$self->{_store}}) {
        my @entry_tags = @{$self->{_store}{$key}{tags} // []};
        if (grep { $tag_set{$_} } @entry_tags) {
            delete $self->{_store}{$key};
            $removed++;
        }
    }
    return $removed;
}

sub remember {
    my ($self, $key, $seconds, $callback) = @_;
    return $self->get_or_load($key, $callback, ttl => $seconds);
}

sub size     { scalar keys %{$_[0]->{_store}} }
sub stats    { %{$_[0]->{stats}} }
sub hit_rate { my $s=$_[0]->{stats}; my $t=$s->{hits}+$s->{misses}; $t?sprintf("%.1f%%",100*$s->{hits}/$t):"n/a" }
}

package main;

printf "=== Cache-Aside Pattern ===\n\n";

my $cache = Cache::Aside->new(ttl => 60);

# Simulate DB
my %db = (1=>{id=>1,name=>"Alice",dept=>"eng"},2=>{id=>2,name=>"Bob",dept=>"mkt"},3=>{id=>3,name=>"Carol",dept=>"eng"});
my $db_hits = 0;

my $load_user = sub {
    my $id = shift;
    $db_hits++;
    return $db{$id};
};

# Cache-aside: check cache, on miss load from DB, store in cache
printf "User lookups (cache-aside):\n";
for my $id (1,2,1,3,2,1) {
    my $user = $cache->get_or_load("user:$id", sub { $load_user->($id) },
                                   tags => ["user", "user:$id"]);
    printf "  user:%d => %s\n", $id, $user->{name};
}

printf "\nDB hits: %d (expected: 3 - each unique user once)\n", $db_hits;
printf "Cache size: %d\n", $cache->size;

# remember() shorthand
printf "\nremember() helper:\n";
my $count = $cache->remember("user_count", 300, sub {
    printf "  (computing user count...)\n";
    return scalar keys %db;
});
printf "  user_count: %d\n", $count;
my $count2 = $cache->remember("user_count", 300, sub { 999 });  # Should use cache
printf "  user_count (cached): %d\n\n", $count2;

# Tag-based invalidation
printf "Tag-based invalidation:\n";
$cache->set("user:1",   {id=>1, name=>"Alice Updated"}, tags=>["user","user:1"]);
$cache->set("user:list",{total=>3},                     tags=>["user"]);
$cache->set("post:1",   {title=>"Hello"},               tags=>["post"]);
printf "  Before: cache_size=%d\n", $cache->size;

my $removed = $cache->invalidate_tags("user");
printf "  invalidate_tags('user'): removed=%d\n", $removed;
printf "  After:  cache_size=%d (post:1 still there)\n\n", $cache->size;

my %stats = $cache->stats;
printf "Stats: hits=%d misses=%d sets=%d deletes=%d hit_rate=%s\n",
    $stats{hits}, $stats{misses}, $stats{sets}, $stats{deletes}, $cache->hit_rate;
```

---

## Step 406: Distributed Cache Simulation

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(sha1_hex);

{
package Cache::Distributed;

# Consistent hashing for key distribution

sub new {
    my ($class, %opts) = @_;
    my $self = bless {
        nodes       => [],
        ring        => {},
        replicas    => $opts{replicas} // 150,
        data        => {},
        stats       => { hits=>0, misses=>0, redirects=>0 },
    }, $class;
    
    for my $node (@{$opts{nodes}/[]}) {
        $self->add_node($node);
    }
    return $self;
}

sub add_node {
    my ($self, $name) = @_;
    push @{$self->{nodes}}, $name;
    
    # Virtual nodes for even distribution
    for my $i (0..$self->{replicas}-1) {
        my $hash = _hash("$name:$i");
        $self->{ring}{$hash} = $name;
    }
}

sub remove_node {
    my ($self, $name) = @_;
    @{$self->{nodes}} = grep { $_ ne $name } @{$self->{nodes}};
    for my $i (0..$self->{replicas}-1) {
        delete $self->{ring}{_hash("$name:$i")};
    }
}

sub get_node {
    my ($self, $key) = @_;
    return undef unless %{$self->{ring}};
    
    my $hash   = _hash($key);
    my @sorted = sort keys %{$self->{ring}};
    
    # Find first node >= hash (wrap around)
    for my $h (@sorted) {
        return $self->{ring}{$h} if $h ge $hash;
    }
    return $self->{ring}{$sorted[0]};
}

sub set {
    my ($self, $key, $value, %opts) = @_;
    my $node = $self->get_node($key);
    $self->{data}{"$node:$key"} = { value=>$value, exp=>time()+($opts{ttl}//300) };
}

sub get {
    my ($self, $key) = @_;
    my $node  = $self->get_node($key);
    my $entry = $self->{data}{"$node:$key"};
    if ($entry && $entry->{exp} > time()) {
        $self->{stats}{hits}++;
        return $entry->{value};
    }
    $self->{stats}{misses}++;
    return undef;
}

sub distribution {
    my $self = shift;
    my %count;
    $count{$_}++ for values %{$self->{ring}};
    my %pct;
    my $total = scalar keys %{$self->{ring}};
    $pct{$_} = sprintf "%.1f%%", 100*$count{$_}/$total for keys %count;
    return %pct;
}

sub _hash { sha1_hex($_[0]) }
}

package main;

printf "=== Distributed Cache (Consistent Hashing) ===\n\n";

my $dc = Cache::Distributed->new(
    nodes    => ["node1", "node2", "node3"],
    replicas => 100,
);

printf "Node distribution:\n";
my %dist = $dc->distribution;
printf "  %s: %s\n", $_, $dist{$_} for sort keys %dist;

# Store data
printf "\nStoring 30 keys:\n";
my %key_nodes;
for my $i (1..30) {
    my $key = "key_$i";
    $dc->set($key, "value_$i");
    $key_nodes{$dc->get_node($key)}{count}++;
}

printf "  Key distribution:\n";
printf "    %s: %d keys\n", $_, $key_nodes{$_}{count} for sort keys %key_nodes;

# Verify retrieval
printf "\nRetrieving:\n";
my $ok = 0;
for my $i (1..30) {
    $ok++ if $dc->get("key_$i") eq "value_$i";
}
printf "  All %d/30 keys retrieved correctly\n\n", $ok;

# Simulate node failure
printf "Adding node4 (rebalancing):\n";
$dc->add_node("node4");
my %new_dist = $dc->distribution;
printf "  New distribution:\n";
printf "    %s: %s\n", $_, $new_dist{$_} for sort keys %new_dist;

# In real scenario, some keys would now route to node4
my $node4_keys = 0;
for my $i (1..30) {
    $node4_keys++ if ($dc->get_node("key_$i") // "") eq "node4";
}
printf "  Keys now routed to node4: %d\n", $node4_keys;
```

---

## Step 407: Memoization & Function Caching

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Memoize;

sub new {
    my ($class, %opts) = @_;
    return bless {
        ttl     => $opts{ttl}     // 0,
        maxsize => $opts{maxsize} // 0,
        _cache  => {},
        _lru    => [],
        stats   => { hits=>0, misses=>0, computes=>0 },
    }, $class;
}

sub wrap {
    my ($self, $fn, %opts) = @_;
    my $key_fn = $opts{key} // sub { join "\0", @_ };
    
    return sub {
        my $cache_key = $key_fn->(@_);
        
        my $entry = $self->{_cache}{$cache_key};
        if ($entry && (!$self->{ttl} || $entry->{exp} > time())) {
            $self->{stats}{hits}++;
            return $entry->{result};
        }
        
        $self->{stats}{misses}++;
        $self->{stats}{computes}++;
        my $result = $fn->(@_);
        $self->{_cache}{$cache_key} = {
            result => $result,
            exp    => $self->{ttl} ? time() + $self->{ttl} : undef,
        };
        return $result;
    };
}

sub stats    { %{$_[0]->{stats}} }
sub cache_size { scalar keys %{$_[0]->{_cache}} }
sub clear    { $_[0]->{_cache} = {}; $_[0]->{stats} = {hits=>0,misses=>0,computes=>0} }
}

# Decorator style
sub memoize_fn {
    my ($fn, %opts) = @_;
    my %cache;
    my $ttl = $opts{ttl} // 0;
    my $hits = 0; my $misses = 0;
    
    my $wrapped = sub {
        my $key = join "\0", map { defined $_ ? $_ : "(undef)" } @_;
        my $entry = $cache{$key};
        if ($entry && (!$ttl || $entry->{exp} > time())) {
            $hits++;
            return wantarray ? @{$entry->{result}} : $entry->{result}[0];
        }
        $misses++;
        my @result = $fn->(@_);
        $cache{$key} = { result => \@result, exp => $ttl ? time()+$ttl : undef };
        return wantarray ? @result : $result[0];
    };
    
    return ($wrapped, sub { hits=>$hits, misses=>$misses });
}

package main;

printf "=== Memoization ===\n\n";

my $memo = Memoize->new(ttl => 60);

# Expensive Fibonacci
my $fib_calls = 0;
my $fib; $fib = $memo->wrap(sub {
    my $n = shift;
    $fib_calls++;
    return $n if $n <= 1;
    return $fib->($n-1) + $fib->($n-2);
});

printf "Fibonacci (memoized):\n";
for my $n (10, 20, 10, 30, 20) {
    my $result = $fib->($n);
    printf "  fib(%2d) = %d\n", $n, $result;
}
printf "  Total recursive calls: %d\n", $fib_calls;
my %stats = $memo->stats;
printf "  Cache hits=%d misses=%d\n\n", $stats{hits}, $stats{misses};

# Custom key function
my $memo2 = Memoize->new;
my $user_loader = $memo2->wrap(
    sub { my ($id,$role)=@_; printf "  [LOAD user:$id/$role]\n"; return { id=>$id, role=>$role } },
    key => sub { "$_[0]:$_[1]" }
);

printf "Custom key memoize:\n";
$user_loader->(1, "admin");
$user_loader->(1, "admin");  # cache hit
$user_loader->(1, "user");   # different role = different key
$user_loader->(2, "user");
$user_loader->(1, "admin");  # cache hit

my %s2 = $memo2->stats;
printf "  hits=%d misses=%d size=%d\n\n", $s2{hits}, $s2{misses}, $memo2->cache_size;

# Decorator function style
printf "Function decorator:\n";
my $compute_calls = 0;
my ($cached_compute, $get_stats) = memoize_fn(sub {
    my ($x, $y) = @_;
    $compute_calls++;
    return $x * $y + sqrt($x + $y);
}, ttl => 30);

for my $pair ([3,4],[5,6],[3,4],[7,8],[5,6]) {
    my $result = $cached_compute->(@$pair);
    printf "  compute(%d,%d) = %.2f\n", $pair->[0], $pair->[1], $result;
}
printf "  Actual compute calls: %d/5\n", $compute_calls;
```

---

## Step 408: Cache Stampede Protection

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Cache::Safe;
# Prevents cache stampede with locking and probabilistic early refresh

my %locks;

sub new {
    my ($class, %opts) = @_;
    return bless {
        _cache    => {},
        ttl       => $opts{ttl}       // 300,
        beta      => $opts{beta}      // 1.0,   # XFetch probability
        lock_ttl  => $opts{lock_ttl}  // 5,
        stats     => { hits=>0, misses=>0, stampedes_prevented=>0 },
    }, $class;
}

sub fetch {
    my ($self, $key, $recompute, %opts) = @_;
    my $ttl  = $opts{ttl} // $self->{ttl};
    my $beta = $opts{beta} // $self->{beta};
    
    my $entry = $self->{_cache}{$key};
    
    # Probabilistic early expiry (XFetch algorithm)
    if ($entry) {
        my $delta = $entry->{delta};
        my $expiry= $entry->{exp};
        my $now   = time();
        
        # Early refresh probability: -delta * beta * log(rand())
        my $should_refresh = $now - $delta * $beta * log(rand()) >= $expiry;
        
        unless ($should_refresh) {
            $self->{stats}{hits}++;
            return $entry->{value};
        }
    }
    
    # Lock to prevent stampede
    if ($locks{$key} && $locks{$key} > time()) {
        $self->{stats}{stampedes_prevented}++;
        # Return stale value while lock held
        return $entry ? $entry->{value} : undef;
    }
    
    # Acquire lock
    $locks{$key} = time() + $self->{lock_ttl};
    
    my $start  = time();
    my $value  = $recompute->();
    my $delta  = time() - $start;
    
    $self->{_cache}{$key} = {
        value => $value,
        exp   => time() + $ttl,
        delta => $delta || 0.001,
    };
    
    delete $locks{$key};
    $self->{stats}{misses}++;
    return $value;
}

sub stats    { %{$_[0]->{stats}} }
sub get      { $_[0]->{_cache}{$_[1]}{value} }
sub invalidate { delete $_[0]->{_cache}{$_[1]} }
}

package main;

printf "=== Cache Stampede Protection ===\n\n";

my $cache = Cache::Safe->new(ttl => 60, beta => 1.0);
my $recomputes = 0;

my $loader = sub {
    $recomputes++;
    printf "  [RECOMPUTE called]\n";
    return { data => "expensive result", computed_at => time() };
};

printf "First 5 requests (first is cold):\n";
for my $i (1..5) {
    my $result = $cache->fetch("hot_key", $loader, ttl => 60);
    printf "  Request %d: %s\n", $i, $result ? "got data" : "nil";
}

printf "Recomputes: %d (expected: 1)\n", $recomputes;

# Simulate stampede (multiple rapid requests with cache forced expired)
printf "\nSimulating expiry:\n";
$cache->invalidate("hot_key");

# Simulate concurrent requests by holding the lock
$locks{stampede_key} = time() + 10 if eval { *locks = \%Cache::Safe::locks; 1 };
{
    no strict 'refs';
    %{"Cache::Safe::locks"} = (stampede_key => time()+10);
}

$recomputes = 0;
my $r1 = $cache->fetch("stampede_key", sub { $recomputes++; "value" });
my $r2 = $cache->fetch("stampede_key", sub { $recomputes++; "value" });
my %stats = $cache->stats;
printf "Stampede prevented: %d times\n", $stats{stampedes_prevented};
printf "Recomputes with lock: %d\n\n", $recomputes;
```

---

## Step 409: Cache Metrics & Monitoring

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Cache::Metrics;

sub new {
    my ($class, $cache) = @_;
    return bless {
        cache    => $cache,
        samples  => [],
        max_samples => 1000,
    }, $class;
}

sub record {
    my ($self, %event) = @_;
    push @{$self->{samples}}, { %event, ts => time() };
    if (@{$self->{samples}} > $self->{max_samples}) {
        shift @{$self->{samples}};
    }
}

sub summary {
    my $self = shift;
    my @s = @{$self->{samples}};
    return {} unless @s;
    
    my %type_count;
    my @latencies;
    
    for my $s (@s) {
        $type_count{$s->{type}}++;
        push @latencies, $s->{latency} if $s->{latency};
    }
    
    my $total = scalar @s;
    my $hits   = $type_count{hit}   // 0;
    my $misses = $type_count{miss}  // 0;
    
    @latencies = sort { $a <=> $b } @latencies;
    my $p50 = @latencies ? $latencies[int(@latencies*0.50)] : 0;
    my $p95 = @latencies ? $latencies[int(@latencies*0.95)] : 0;
    my $p99 = @latencies ? $latencies[int(@latencies*0.99)] : 0;
    my $avg = @latencies ? do { my $sum=0; $sum+=$_ for @latencies; $sum/@latencies } : 0;
    
    return {
        total    => $total,
        hits     => $hits,
        misses   => $misses,
        hit_rate => $total ? sprintf("%.1f%%", 100*$hits/$total) : "n/a",
        latency  => { avg=>$avg, p50=>$p50, p95=>$p95, p99=>$p99 },
        events   => \%type_count,
    };
}

sub percentiles {
    my ($self, @pcts) = @_;
    my @lat = sort { $a <=> $b } map { $_->{latency}//() } @{$self->{samples}};
    return {} unless @lat;
    return { map { $_ => $lat[int(@lat * $_/100)] } @pcts };
}
}

package main;

printf "=== Cache Metrics ===\n\n";

# Simulate a cache with metrics
my %cache_store;
my $metrics = Cache::Metrics->new(undef);

srand(42);  # Reproducible

my $total_requests = 200;
my $hot_keys   = [map { "hot_$_" } 1..5];   # 80% of traffic
my $cold_keys  = [map { "cold_$_" } 1..50]; # 20% of traffic

printf "Running %d simulated requests:\n", $total_requests;
for my $i (1..$total_requests) {
    # Zipf-like distribution: 80% hot keys
    my $key = rand() < 0.8
        ? $hot_keys->[rand(@$hot_keys)]
        : $cold_keys->[rand(@$cold_keys)];
    
    my $start = time();
    my $type;
    if (exists $cache_store{$key}) {
        $type = "hit";
    } else {
        $cache_store{$key} = 1;
        $type = "miss";
    }
    
    # Simulate latency (hits ~1ms, misses ~10ms)
    my $latency = $type eq "hit" ? 0.001 + rand(0.002) : 0.008 + rand(0.015);
    $metrics->record(type => $type, key => $key, latency => $latency);
}

my $summary = $metrics->summary;
printf "\nMetrics Summary:\n";
printf "  Total requests: %d\n",   $summary->{total};
printf "  Hits:           %d\n",   $summary->{hits};
printf "  Misses:         %d\n",   $summary->{misses};
printf "  Hit rate:       %s\n",   $summary->{hit_rate};
printf "  Latency avg:    %.3fms\n", $summary->{latency}{avg}*1000;
printf "  Latency p50:    %.3fms\n", $summary->{latency}{p50}*1000;
printf "  Latency p95:    %.3fms\n", $summary->{latency}{p95}*1000;
printf "  Latency p99:    %.3fms\n", $summary->{latency}{p99}*1000;

my %pcts = $metrics->percentiles(50, 90, 95, 99);
printf "\nPercentile breakdown:\n";
printf "  p%d: %.3fms\n", $_, $pcts{$_}*1000 for sort { $a<=>$b } keys %pcts;
```

---

## Step 410: Capstone — Multi-layer Cache Architecture

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

my $JSON = JSON::PP->new->utf8->canonical;

# L1: Process-local memory (fastest, smallest)
# L2: Shared memory simulation (fast, medium)
# L3: Persistent cache simulation (slower, largest)
# DB: Database (slowest, source of truth)

my %L1; my %L2; my %L3; my %DB;
my %STATS = map { $_ => { hits=>0, misses=>0, reads=>0, writes=>0 } } qw(L1 L2 L3 DB);

# Populate DB
$DB{"user:$_"} = { id=>$_, name=>"User$_", email=>"user${_}\@test.com", score=>int(rand(100)) }
    for 1..100;

sub cache_get {
    my ($key) = @_;
    
    # L1
    if (my $v = $L1{$key}) {
        $STATS{L1}{hits}++;
        return $v->{value};
    }
    $STATS{L1}{misses}++;
    
    # L2
    if (my $v = $L2{$key}) {
        $STATS{L2}{hits}++;
        $L1{$key} = { value=>$v->{value}, exp=>time()+30 };  # promote to L1
        return $v->{value};
    }
    $STATS{L2}{misses}++;
    
    # L3
    if (my $v = $L3{$key}) {
        $STATS{L3}{hits}++;
        $L2{$key} = { value=>$v->{value}, exp=>time()+120 }; # promote to L2
        $L1{$key} = { value=>$v->{value}, exp=>time()+30 };  # promote to L1
        return $v->{value};
    }
    $STATS{L3}{misses}++;
    
    # DB
    my $val = $DB{$key};
    $STATS{DB}{reads}++;
    return undef unless $val;
    
    # Fill all layers
    $L3{$key} = { value=>$val, exp=>time()+3600 };
    $L2{$key} = { value=>$val, exp=>time()+120  };
    $L1{$key} = { value=>$val, exp=>time()+30   };
    
    return $val;
}

sub cache_set {
    my ($key, $value) = @_;
    $DB{$key}  = $value;  # Source of truth
    $L3{$key}  = { value=>$value, exp=>time()+3600 };
    $L2{$key}  = { value=>$value, exp=>time()+120  };
    $L1{$key}  = { value=>$value, exp=>time()+30   };
    $STATS{DB}{writes}++;
}

sub cache_invalidate {
    my ($key) = @_;
    delete $L1{$key};
    delete $L2{$key};
    delete $L3{$key};
}

printf "=== Multi-layer Cache Architecture ===\n\n";

printf "Scenario 1: Cold start (100 unique keys)\n";
cache_get("user:$_") for 1..100;
printf "  DB reads: %d (all cache misses)\n\n", $STATS{DB}{reads};

# Reset stats
%STATS = map { $_ => { hits=>0, misses=>0, reads=>0, writes=>0 } } qw(L1 L2 L3 DB);

printf "Scenario 2: Hot path (popular users accessed repeatedly)\n";
my @popular = (1,5,10,25,50);
for my $round (1..20) {
    for my $id (@popular) {
        cache_get("user:$id");
    }
}
printf "  Total requests: %d\n",    20 * scalar @popular;
printf "  L1 hits: %d\n",  $STATS{L1}{hits};
printf "  L2 hits: %d\n",  $STATS{L2}{hits};
printf "  L3 hits: %d\n",  $STATS{L3}{hits};
printf "  DB hits: %d\n",  $STATS{DB}{reads};
printf "  L1 hit rate: %.1f%%\n\n",
    $STATS{L1}{hits} / (20*scalar@popular) * 100;

printf "Scenario 3: Cache invalidation cascade\n";
cache_set("user:1", { id=>1, name=>"Alice Updated", email=>"alice\@test.com" });
printf "  Written to all layers: %s\n",
    defined($L1{"user:1"}) ? "YES" : "NO";

cache_invalidate("user:5");
printf "  Invalidated user:5: L1=%s L2=%s L3=%s\n",
    (defined $L1{"user:5"} ? "exists" : "gone") x 3;

# Re-read after invalidation
my $re = cache_get("user:5");
printf "  Re-fetch from DB: %s\n\n", defined $re ? "OK ($re->{name})" : "MISS";

# Summary
printf "Cache layer summary:\n";
printf "  L1 (process): %d keys cached\n", scalar keys %L1;
printf "  L2 (shared):  %d keys cached\n", scalar keys %L2;
printf "  L3 (persist): %d keys cached\n", scalar keys %L3;
printf "  DB:           %d records\n",     scalar keys %DB;
```

---

## สรุป Part 41 — Caching Strategies

### สิ่งที่เรียนรู้:
- **LRU Cache** — Doubly-linked list + hash, O(1) operations
- **Eviction Policies** — LRU, LFU, FIFO, Random with TTL
- **Cache Warming** — Preload, batch warm, prefix invalidation
- **Write-through/Write-behind** — Sync vs async DB writes
- **Cache-aside** — App-managed, tag-based invalidation
- **Consistent Hashing** — Virtual nodes, even distribution
- **Memoization** — Function-level caching, custom keys
- **Stampede Protection** — XFetch algorithm, distributed locks
- **Metrics** — Hit rate, latency percentiles (p50/p95/p99)
- **Capstone** — L1/L2/L3/DB multi-layer cache cascade

**ถัดไป: [Part 42 — Background Jobs & Task Queues](part_42.md)**
