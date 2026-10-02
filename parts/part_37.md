# Part 37: Advanced DBI & ORM Patterns
## Steps 361-370: Mastering Database Access in Perl

---

## Step 361: DBI Connection Pool & Reconnect

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

{
package DBI::Pool;

sub new {
    my ($class, %opts) = @_;
    return bless {
        dsn        => $opts{dsn},
        user       => $opts{user}     // "",
        pass       => $opts{password} // "",
        options    => $opts{options}  // { RaiseError => 1, AutoCommit => 1 },
        max_size   => $opts{max_size} // 10,
        min_size   => $opts{min_size} // 2,
        pool       => [],
        in_use     => [],
        stats      => { acquired => 0, released => 0, created => 0, destroyed => 0 },
        timeout    => $opts{timeout}  // 30,
    }, $class;
}

sub _create_connection {
    my $self = shift;
    my $dbh = DBI->connect($self->{dsn}, $self->{user}, $self->{pass}, $self->{options});
    $self->{stats}{created}++;
    return { dbh => $dbh, created_at => time(), last_used => time(), id => $self->{stats}{created} };
}

sub acquire {
    my $self = shift;
    
    # Try to get from pool
    while (my $conn = shift @{$self->{pool}}) {
        # Check connection is alive
        if (eval { $conn->{dbh}->ping }) {
            push @{$self->{in_use}}, $conn;
            $conn->{last_used} = time();
            $self->{stats}{acquired}++;
            return $conn->{dbh};
        } else {
            # Dead connection
            eval { $conn->{dbh}->disconnect };
            $self->{stats}{destroyed}++;
        }
    }
    
    # Check pool size
    if (@{$self->{in_use}} >= $self->{max_size}) {
        die "Connection pool exhausted (max=$self->{max_size})";
    }
    
    # Create new
    my $conn = $self->_create_connection;
    push @{$self->{in_use}}, $conn;
    $self->{stats}{acquired}++;
    return $conn->{dbh};
}

sub release {
    my ($self, $dbh) = @_;
    
    # Find and remove from in_use
    my $conn;
    @{$self->{in_use}} = grep {
        if ($_->{dbh} == $dbh) { $conn = $_; 0 } else { 1 }
    } @{$self->{in_use}};
    
    return unless $conn;
    $self->{stats}{released}++;
    
    # Return to pool if healthy
    if (@{$self->{pool}} < $self->{min_size}) {
        $conn->{last_used} = time();
        eval { $dbh->rollback if !$dbh->{AutoCommit} };
        push @{$self->{pool}}, $conn;
    } else {
        eval { $dbh->disconnect };
        $self->{stats}{destroyed}++;
    }
}

sub with_connection {
    my ($self, $code) = @_;
    my $dbh = $self->acquire;
    my $result = eval { $code->($dbh) };
    my $err = $@;
    $self->release($dbh);
    die $err if $err;
    return $result;
}

sub stats {
    my $self = shift;
    return {
        %{$self->{stats}},
        pool_size  => scalar @{$self->{pool}},
        in_use     => scalar @{$self->{in_use}},
        total      => scalar @{$self->{pool}} + scalar @{$self->{in_use}},
    };
}

sub close_all {
    my $self = shift;
    for my $conn (@{$self->{pool}}, @{$self->{in_use}}) {
        eval { $conn->{dbh}->disconnect };
        $self->{stats}{destroyed}++;
    }
    $self->{pool} = $self->{in_use} = [];
}
}

package main;

printf "=== DBI Connection Pool ===\n\n";

my $pool = DBI::Pool->new(
    dsn      => "dbi:SQLite::memory:",
    min_size => 2,
    max_size => 5,
);

# Initialize with table
$pool->with_connection(sub {
    my $dbh = shift;
    $dbh->do("CREATE TABLE test (id INTEGER PRIMARY KEY, val TEXT)");
    $dbh->do("INSERT INTO test VALUES (1, 'hello')");
    $dbh->do("INSERT INTO test VALUES (2, 'world')");
});

# Multiple operations
for my $i (1..3) {
    $pool->with_connection(sub {
        my $dbh = shift;
        my $row = $dbh->selectrow_hashref("SELECT * FROM test WHERE id=?", undef, $i % 2 + 1);
        printf "Query %d: %s\n", $i, $row ? $row->{val} : "not found";
    });
}

my $stats = $pool->stats;
printf "\nPool stats:\n";
printf "  created=%d  acquired=%d  released=%d  destroyed=%d\n",
    $stats->{created}, $stats->{acquired}, $stats->{released}, $stats->{destroyed};
printf "  pool_size=%d  in_use=%d\n", $stats->{pool_size}, $stats->{in_use};

$pool->close_all;
printf "  After close: %s\n", $pool->stats->{total} == 0 ? "OK" : "FAIL";
```

---

## Step 362: Fluent Query Builder

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;
use List::Util qw(any);

{
package DB::Query;

sub new {
    my ($class, $dbh) = @_;
    return bless {
        dbh     => $dbh,
        _table  => undef,
        _select => ["*"],
        _where  => [],
        _binds  => [],
        _joins  => [],
        _order  => [],
        _group  => [],
        _having => [],
        _limit  => undef,
        _offset => 0,
        _distinct => 0,
        _lock   => undef,
    }, $class;
}

sub table   { my $s=shift->_clone; $s->{_table}=$_[0]; $s }
sub from    { $_[0]->table($_[1]) }
sub select  { my $s=shift->_clone; $s->{_select}=[ref$_[0]?@{$_[0]}:@_]; $s }
sub distinct{ my $s=shift->_clone; $s->{_distinct}=1; $s }

sub where {
    my ($self, $cond, @binds) = @_;
    my $s = $self->_clone;
    if (ref $cond eq 'HASH') {
        for my $k (sort keys %$cond) {
            if (ref $cond->{$k} eq 'ARRAY') {
                my @placeholders = ("?") x @{$cond->{$k}};
                push @{$s->{_where}}, "$k IN (" . join(",", @placeholders) . ")";
                push @{$s->{_binds}}, @{$cond->{$k}};
            } elsif (!defined $cond->{$k}) {
                push @{$s->{_where}}, "$k IS NULL";
            } else {
                push @{$s->{_where}}, "$k = ?";
                push @{$s->{_binds}}, $cond->{$k};
            }
        }
    } else {
        push @{$s->{_where}}, $cond;
        push @{$s->{_binds}}, @binds;
    }
    return $s;
}

sub where_not {
    my ($self, $cond, @binds) = @_;
    my $s = $self->_clone;
    if (ref $cond eq 'HASH') {
        for my $k (sort keys %$cond) {
            push @{$s->{_where}}, "$k != ?";
            push @{$s->{_binds}}, $cond->{$k};
        }
    } else {
        push @{$s->{_where}}, "NOT ($cond)";
        push @{$s->{_binds}}, @binds;
    }
    return $s;
}

sub where_like {
    my ($self, $col, $pattern) = @_;
    my $s = $self->_clone;
    push @{$s->{_where}}, "$col LIKE ?";
    push @{$s->{_binds}}, $pattern;
    return $s;
}

sub where_between {
    my ($self, $col, $min, $max) = @_;
    my $s = $self->_clone;
    push @{$s->{_where}}, "$col BETWEEN ? AND ?";
    push @{$s->{_binds}}, $min, $max;
    return $s;
}

sub or_where {
    my ($self, $cond, @binds) = @_;
    my $s = $self->_clone;
    my $last = pop @{$s->{_where}};
    push @{$s->{_where}}, "($last OR $cond)";
    push @{$s->{_binds}}, @binds;
    return $s;
}

sub join_table {
    my ($self, $table, $on, $type) = @_;
    my $s = $self->_clone;
    $type //= "INNER";
    push @{$s->{_joins}}, "$type JOIN $table ON $on";
    return $s;
}

sub left_join  { $_[0]->join_table($_[1], $_[2], "LEFT") }
sub right_join { $_[0]->join_table($_[1], $_[2], "RIGHT") }
sub inner_join { $_[0]->join_table($_[1], $_[2], "INNER") }

sub order_by {
    my ($self, @cols) = @_;
    my $s = $self->_clone;
    push @{$s->{_order}}, @cols;
    return $s;
}

sub order_asc  { $_[0]->order_by("$_[1] ASC") }
sub order_desc { $_[0]->order_by("$_[1] DESC") }

sub group_by {
    my ($self, @cols) = @_;
    my $s = $self->_clone;
    push @{$s->{_group}}, @cols;
    return $s;
}

sub having {
    my ($self, $cond, @binds) = @_;
    my $s = $self->_clone;
    push @{$s->{_having}}, $cond;
    push @{$s->{_binds}}, @binds;
    return $s;
}

sub limit  { my $s=shift->_clone; $s->{_limit}=$_[0]; $s }
sub offset { my $s=shift->_clone; $s->{_offset}=$_[0]; $s }
sub page   { my ($self,$n,$size)=@_; $self->limit($size//10)->offset(($n-1)*($size//10)) }

sub to_sql {
    my $self = shift;
    my $distinct = $self->{_distinct} ? "DISTINCT " : "";
    my $sql = sprintf "SELECT %s%s FROM %s",
        $distinct,
        join(", ", @{$self->{_select}}),
        $self->{_table} // die "No table set";
    
    $sql .= " " . join(" ", @{$self->{_joins}}) if @{$self->{_joins}};
    $sql .= " WHERE " . join(" AND ", @{$self->{_where}}) if @{$self->{_where}};
    $sql .= " GROUP BY " . join(", ", @{$self->{_group}}) if @{$self->{_group}};
    $sql .= " HAVING " . join(" AND ", @{$self->{_having}}) if @{$self->{_having}};
    $sql .= " ORDER BY " . join(", ", @{$self->{_order}}) if @{$self->{_order}};
    $sql .= " LIMIT $self->{_limit}"   if defined $self->{_limit};
    $sql .= " OFFSET $self->{_offset}" if $self->{_offset};
    
    return ($sql, @{$self->{_binds}});
}

sub get      { my ($self) = @_; my ($sql,@b) = $self->to_sql; $self->{dbh}->selectall_arrayref($sql,{Slice=>{}},@b) }
sub first    { my $rows = shift->limit(1)->get; $rows->[0] }
sub count    { my $s = shift->_clone; $s->{_select}=["COUNT(*)"]; my ($sql,@b)=$s->to_sql; $s->{dbh}->selectrow_array($sql,undef,@b)+0 }
sub exists   { shift->count > 0 }
sub pluck    { my ($self,$col) = @_; $self->{_select}=[$col]; [map{$_->{$col}} @{$self->get}] }

sub _clone {
    my $self = shift;
    return bless {
        %$self,
        _select => [@{$self->{_select}}],
        _where  => [@{$self->{_where}}],
        _binds  => [@{$self->{_binds}}],
        _joins  => [@{$self->{_joins}}],
        _order  => [@{$self->{_order}}],
        _group  => [@{$self->{_group}}],
        _having => [@{$self->{_having}}],
    }, ref $self;
}
}

package main;

printf "=== Fluent Query Builder ===\n\n";

my $dbh = DBI->connect("dbi:SQLite::memory:", "", "", {RaiseError=>1, AutoCommit=>1});
$dbh->do("CREATE TABLE products (id INTEGER PRIMARY KEY, name TEXT, price REAL, category TEXT, stock INTEGER)");
$dbh->do("CREATE TABLE categories (id INTEGER PRIMARY KEY, name TEXT, description TEXT)");

for my $p (
    [1,"Widget A",9.99,"electronics",100],
    [2,"Widget B",19.99,"electronics",50],
    [3,"Gadget C",5.99,"accessories",200],
    [4,"Gadget D",24.99,"electronics",10],
    [5,"Case E",14.99,"accessories",75],
    [6,"Cable F",3.99,"accessories",150],
) { $dbh->do("INSERT INTO products VALUES (?,?,?,?,?)", undef, @$p) }

my $q = DB::Query->new($dbh);

# Basic query
my ($sql, @binds) = $q->table("products")->where({category => "electronics"})->order_desc("price")->to_sql;
printf "SQL: %s (binds: %s)\n", $sql, join(",", @binds);

# Execute
my $electronics = $q->table("products")
    ->where({category => "electronics"})
    ->order_desc("price")
    ->get;
printf "\nElectronics (%d):\n", scalar @$electronics;
printf "  [%d] %-15s \$%.2f\n", $_->{id}, $_->{name}, $_->{price} for @$electronics;

# Complex query
my $cheap = $q->table("products")
    ->where_between("price", 5, 15)
    ->where_not({stock => 0})
    ->order_asc("price")
    ->get;
printf "\nPrice \$5-\$15 (%d):\n", scalar @$cheap;
printf "  %-15s \$%.2f\n", $_->{name}, $_->{price} for @$cheap;

# Count & exists
printf "\nAll products: %d\n",         $q->table("products")->count;
printf "Electronics: %d\n",            $q->table("products")->where({category=>"electronics"})->count;
printf "Has accessories: %s\n",        $q->table("products")->where({category=>"accessories"})->exists ? "yes":"no";

# Pluck
my $names = $q->table("products")->order_asc("name")->pluck("name");
printf "\nProduct names: %s\n", join(", ", @$names);

# Aggregate
my ($agg_sql, @ab) = $q->table("products")
    ->select(["category", "COUNT(*) as count", "AVG(price) as avg_price", "SUM(stock) as total_stock"])
    ->group_by("category")
    ->order_asc("category")
    ->to_sql;
printf "\nAggregate SQL: %s\n", $agg_sql;
my $aggs = $dbh->selectall_arrayref($agg_sql, {Slice=>{}}, @ab);
printf "  %-15s count=%d avg=\$%.2f stock=%d\n",
    $_->{category}, $_->{count}, $_->{avg_price}, $_->{total_stock} for @$aggs;

# Pagination
printf "\nPage 1 (2 per page):\n";
my $page1 = $q->table("products")->order_asc("id")->page(1, 2)->get;
printf "  %s\n", $_->{name} for @$page1;
printf "Page 2:\n";
my $page2 = $q->table("products")->order_asc("id")->page(2, 2)->get;
printf "  %s\n", $_->{name} for @$page2;
```

---

## Step 363: Advanced ActiveRecord

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

my $DBH;

{
package ActiveRecord;

sub dbh { $DBH }

sub setup {
    my ($class, $dbh) = @_;
    $DBH = $dbh;
}

sub new {
    my ($class, %attrs) = @_;
    my $self = bless { _attrs => {}, _dirty => {}, _new_record => 1 }, $class;
    $self->_set_attr($_, $attrs{$_}) for keys %attrs;
    return $self;
}

sub _set_attr {
    my ($self, $key, $val) = @_;
    if (exists $self->{_attrs}{$key} && defined $val) {
        $self->{_dirty}{$key} = 1;
    }
    $self->{_attrs}{$key} = $val;
}

sub AUTOLOAD {
    my $self = shift;
    our $AUTOLOAD;
    my $name = $AUTOLOAD;
    $name =~ s/.*:://;
    return if $name eq 'DESTROY';
    
    if (@_) {
        $self->{_dirty}{$name} = 1 unless $self->{_new_record};
        $self->{_attrs}{$name} = shift;
        return $self;
    }
    return $self->{_attrs}{$name};
}

sub save {
    my $self = shift;
    my $class = ref $self;
    my $table = $class->table;
    my $pk    = $class->primary_key // "id";
    
    if ($self->{_new_record}) {
        $self->before_create if $self->can("before_create");
        my @cols  = grep { $_ ne $pk } keys %{$self->{_attrs}};
        my @vals  = map  { $self->{_attrs}{$_} } @cols;
        my $sql   = sprintf "INSERT INTO $table (%s) VALUES (%s)",
            join(",", @cols), join(",", ("?") x @cols);
        $DBH->do($sql, undef, @vals);
        $self->{_attrs}{$pk} = $DBH->last_insert_id;
        $self->{_new_record} = 0;
        $self->after_create if $self->can("after_create");
    } else {
        $self->before_update if $self->can("before_update");
        my @dirty = keys %{$self->{_dirty}};
        return $self unless @dirty;
        my @vals  = map { $self->{_attrs}{$_} } @dirty;
        my $set   = join(", ", map { "$_=?" } @dirty);
        $DBH->do("UPDATE $table SET $set WHERE $pk=?", undef, @vals, $self->{_attrs}{$pk});
        $self->{_dirty} = {};
        $self->after_update if $self->can("after_update");
    }
    return $self;
}

sub delete {
    my $self = shift;
    my $class = ref $self;
    my $table = $class->table;
    my $pk    = $class->primary_key // "id";
    $self->before_destroy if $self->can("before_destroy");
    $DBH->do("DELETE FROM $table WHERE $pk=?", undef, $self->{_attrs}{$pk});
    return $self;
}

sub reload {
    my $self  = shift;
    my $class = ref $self;
    my $fresh = $class->find($self->id);
    $self->{_attrs} = $fresh->{_attrs};
    $self->{_dirty} = {};
    return $self;
}

sub _load {
    my ($class, $row) = @_;
    my $self = bless { _attrs => $row, _dirty => {}, _new_record => 0 }, $class;
    return $self;
}

sub find {
    my ($class, $id) = @_;
    my $pk  = $class->primary_key // "id";
    my $row = $DBH->selectrow_hashref("SELECT * FROM " . $class->table . " WHERE $pk=?", undef, $id);
    return $row ? $class->_load($row) : undef;
}

sub all {
    my ($class, %opts) = @_;
    my $order = $opts{order} ? " ORDER BY $opts{order}" : "";
    my $limit = $opts{limit} ? " LIMIT $opts{limit}" : "";
    my $rows  = $DBH->selectall_arrayref("SELECT * FROM " . $class->table . $order . $limit, {Slice=>{}});
    return [map { $class->_load($_) } @$rows];
}

sub where {
    my ($class, %conds) = @_;
    my @where; my @binds;
    for my $k (sort keys %conds) {
        if (ref $conds{$k} eq 'ARRAY') {
            push @where, "$k IN (" . join(",",("?")x@{$conds{$k}}) . ")";
            push @binds, @{$conds{$k}};
        } else {
            push @where, "$k=?"; push @binds, $conds{$k};
        }
    }
    my $sql  = "SELECT * FROM " . $class->table . " WHERE " . join(" AND ", @where);
    my $rows = $DBH->selectall_arrayref($sql, {Slice=>{}}, @binds);
    return [map { $class->_load($_) } @$rows];
}

sub count { $DBH->selectrow_array("SELECT COUNT(*) FROM " . $_[0]->table)+0 }

sub create {
    my ($class, %attrs) = @_;
    return $class->new(%attrs)->save;
}

sub update_attrs {
    my ($self, %attrs) = @_;
    $self->{_dirty}{$_} = 1 for keys %attrs;
    $self->{_attrs}{$_} = $attrs{$_} for keys %attrs;
    return $self->save;
}

# Validations
sub validates {
    my ($class, $field, %rules) = @_;
    no strict 'refs';
    my $existing = \&{"${class}::_validations"};
    # Would store validation rules
}

sub is_valid {
    my $self  = shift;
    my $class = ref $self;
    $self->{_errors} = {};
    $self->validate if $self->can("validate");
    return !%{$self->{_errors}};
}

sub add_error { $_[0]->{_errors}{$_[1]} = $_[2] }
sub errors    { %{$_[0]->{_errors}//{}  } }

# Associations
sub has_many {
    my ($class, $assoc, %opts) = @_;
    my $foreign_key = $opts{foreign_key} // lc($class) . "_id";
    my $assoc_class = $opts{class_name}  // ucfirst($assoc);
    $assoc_class =~ s/s$// unless $opts{class_name};
    no strict 'refs';
    *{"${class}::${assoc}"} = sub {
        my $self = shift;
        my $ac = $assoc_class;
        $ac->where($foreign_key => $self->id);
    };
}

sub belongs_to {
    my ($class, $assoc, %opts) = @_;
    my $foreign_key = $opts{foreign_key} // "${assoc}_id";
    my $assoc_class = $opts{class_name}  // ucfirst($assoc);
    no strict 'refs';
    *{"${class}::${assoc}"} = sub {
        my $self = shift;
        $assoc_class->find($self->$foreign_key);
    };
}

sub table       { die ref(shift) . " must define table()" }
sub primary_key { "id" }
}

# Models
{
package Author;
use parent -norequire, 'ActiveRecord';
sub table { "authors" }

ActiveRecord::has_many("Author", "books", class_name => "Book");

sub validate {
    my $self = shift;
    $self->add_error("name", "required") unless $self->name;
    $self->add_error("email", "invalid") unless ($self->email//"") =~ /@/;
}

sub before_create { $_[0]->{_attrs}{created_at} = time() }
}

{
package Book;
use parent -norequire, 'ActiveRecord';
sub table { "books" }

ActiveRecord::belongs_to("Book", "author", class_name => "Author");

sub validate {
    my $self = shift;
    $self->add_error("title", "required") unless $self->title;
    $self->add_error("price", "must be positive") if ($self->price//0) <= 0;
}
}

package main;

$DBH = DBI->connect("dbi:SQLite::memory:", "", "", {RaiseError=>1, AutoCommit=>1});
$DBH->do("CREATE TABLE authors (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, email TEXT, bio TEXT, created_at INTEGER)");
$DBH->do("CREATE TABLE books (id INTEGER PRIMARY KEY AUTOINCREMENT, title TEXT, price REAL, author_id INTEGER, isbn TEXT)");

ActiveRecord->setup($DBH);

printf "=== Advanced ActiveRecord ===\n\n";

# Create
my $alice = Author->create(name => "Alice Smith", email => 'alice@example.com', bio => "Perl expert");
my $bob   = Author->create(name => "Bob Jones",   email => 'bob@example.com',   bio => "Ruby dev");
printf "Created: %s (id=%d)\n", $_->name, $_->id for ($alice, $bob);

# Books
for my $b (
    { title => "Learning Perl", price => 39.99, author_id => $alice->id, isbn => "978-1" },
    { title => "Perl Cookbook", price => 49.99, author_id => $alice->id, isbn => "978-2" },
    { title => "Ruby Way",      price => 44.99, author_id => $bob->id,   isbn => "978-3" },
) { Book->create(%$b) }

# Query
my $alice_books = $alice->books;
printf "\nAlice's books (%d):\n", scalar @$alice_books;
printf "  %s (\$%.2f)\n", $_->title, $_->price for @$alice_books;

# Update
$alice->update_attrs(bio => "Perl expert and author");
printf "\nUpdated Alice bio: %s\n", Author->find($alice->id)->bio;

# Validations
printf "\nValidation tests:\n";
my $bad = Author->new(name => "", email => "not-email");
printf "  Valid: %s\n", $bad->is_valid ? "yes" : "no";
my %errs = $bad->errors;
printf "  Errors: %s\n", join(", ", map { "$_: $errs{$_}" } sort keys %errs);

my $good = Author->new(name => "Carol", email => 'carol@example.com');
printf "  Valid: %s\n", $good->is_valid ? "yes" : "no";

# Count
printf "\nTotal authors: %d  books: %d\n", Author->count, Book->count;
```

---

## Step 364: Transactions and Savepoints

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

{
package DB::Transaction;

sub new {
    my ($class, $dbh) = @_;
    return bless { dbh => $dbh, depth => 0, savepoints => [] }, $class;
}

sub begin {
    my $self = shift;
    if ($self->{depth} == 0) {
        $self->{dbh}->begin_work;
    } else {
        my $sp = "sp_" . $self->{depth};
        $self->{dbh}->do("SAVEPOINT $sp");
        push @{$self->{savepoints}}, $sp;
    }
    $self->{depth}++;
    return $self;
}

sub commit {
    my $self = shift;
    die "No active transaction" if $self->{depth} <= 0;
    $self->{depth}--;
    
    if ($self->{depth} == 0) {
        $self->{dbh}->commit;
    } else {
        my $sp = pop @{$self->{savepoints}};
        $self->{dbh}->do("RELEASE SAVEPOINT $sp");
    }
    return $self;
}

sub rollback {
    my $self = shift;
    die "No active transaction" if $self->{depth} <= 0;
    $self->{depth}--;
    
    if ($self->{depth} == 0) {
        $self->{dbh}->rollback;
        @{$self->{savepoints}} = ();
    } else {
        my $sp = pop @{$self->{savepoints}};
        $self->{dbh}->do("ROLLBACK TO SAVEPOINT $sp");
    }
    return $self;
}

sub transaction {
    my ($self, $code) = @_;
    $self->begin;
    my $result = eval { $code->() };
    if ($@) {
        $self->rollback;
        die $@;
    }
    $self->commit;
    return $result;
}

sub nested_transaction {
    my ($self, $code) = @_;
    # Same as transaction — uses savepoints when nested
    return $self->transaction($code);
}

sub depth { $_[0]->{depth} }
}

package main;

printf "=== Transactions & Savepoints ===\n\n";

my $dbh = DBI->connect("dbi:SQLite::memory:", "", "", {
    RaiseError => 1, AutoCommit => 0 });

$dbh->do("CREATE TABLE accounts (id INTEGER PRIMARY KEY, name TEXT, balance REAL)");
$dbh->do("INSERT INTO accounts VALUES (1, 'Alice', 1000)");
$dbh->do("INSERT INTO accounts VALUES (2, 'Bob', 500)");
$dbh->commit;

my $tx = DB::Transaction->new($dbh);

sub balance { $dbh->selectrow_array("SELECT balance FROM accounts WHERE id=?", undef, $_[0])+0 }
sub transfer {
    my ($from, $to, $amount) = @_;
    my $bal = balance($from);
    die "Insufficient funds (have $bal, need $amount)\n" if $bal < $amount;
    $dbh->do("UPDATE accounts SET balance=balance-? WHERE id=?", undef, $amount, $from);
    $dbh->do("UPDATE accounts SET balance=balance+? WHERE id=?", undef, $amount, $to);
}

# Successful transfer
printf "Before: Alice=%.2f  Bob=%.2f\n", balance(1), balance(2);

$tx->transaction(sub {
    transfer(1, 2, 200);
    printf "During txn: Alice=%.2f  Bob=%.2f\n", balance(1), balance(2);
});

printf "After commit: Alice=%.2f  Bob=%.2f\n\n", balance(1), balance(2);

# Failed transfer (rollback)
eval {
    $tx->transaction(sub {
        transfer(1, 2, 100);  # OK
        transfer(2, 1, 1000); # Will fail — insufficient funds
    });
};
printf "After failed txn: %s\n", $@ ? "rolled back" : "committed";
printf "Balances: Alice=%.2f  Bob=%.2f\n\n", balance(1), balance(2);

# Nested (savepoints)
$tx->transaction(sub {
    transfer(1, 2, 50);
    printf "Outer txn done\n";
    
    eval {
        $tx->nested_transaction(sub {
            transfer(1, 2, 50);
            transfer(1, 2, 50);
            die "Oops, cancel inner\n";
        });
    };
    printf "Inner failed: %s", $@ if $@;
    # Inner rolled back to savepoint, outer continues
});

printf "After nested: Alice=%.2f  Bob=%.2f\n", balance(1), balance(2);
```

---

## Step 365: Schema Migrations

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

{
package DB::Migrations;

sub new {
    my ($class, $dbh) = @_;
    my $self = bless { dbh => $dbh, migrations => [] }, $class;
    $self->_ensure_schema_table;
    return $self;
}

sub _ensure_schema_table {
    my $self = shift;
    $self->{dbh}->do(<<SQL);
CREATE TABLE IF NOT EXISTS schema_migrations (
    version    TEXT PRIMARY KEY,
    name       TEXT NOT NULL,
    applied_at INTEGER DEFAULT (strftime('%s','now')),
    status     TEXT DEFAULT 'applied'
)
SQL
}

sub add {
    my ($self, $version, $name, %steps) = @_;
    push @{$self->{migrations}}, {
        version => $version,
        name    => $name,
        up      => $steps{up}   // sub {},
        down    => $steps{down} // sub {},
    };
    return $self;
}

sub applied {
    my $self = shift;
    my $rows = $self->{dbh}->selectall_arrayref(
        "SELECT version FROM schema_migrations WHERE status='applied' ORDER BY version",
        {Slice=>{}});
    return { map { $_->{version} => 1 } @$rows };
}

sub pending {
    my $self    = shift;
    my $applied = $self->applied;
    return grep { !$applied->{$_->{version}} } @{$self->{migrations}};
}

sub migrate {
    my ($self, %opts) = @_;
    my $target = $opts{to};
    my $dry    = $opts{dry_run} // 0;
    
    my @pending = $self->pending;
    unless (@pending) {
        printf "Nothing to migrate.\n";
        return 0;
    }
    
    my $count = 0;
    for my $m (@pending) {
        last if defined $target && $m->{version} gt $target;
        
        printf "%s Migrating %s: %s\n", $dry?"[DRY]":"", $m->{version}, $m->{name};
        unless ($dry) {
            eval {
                $self->{dbh}->begin_work;
                $m->{up}->($self->{dbh});
                $self->{dbh}->do("INSERT INTO schema_migrations (version, name) VALUES (?,?)",
                    undef, $m->{version}, $m->{name});
                $self->{dbh}->commit;
            };
            if ($@) {
                $self->{dbh}->rollback;
                printf "  FAILED: %s\n", $@;
                return $count;
            }
        }
        $count++;
    }
    printf "Migrated %d version(s).\n", $count;
    return $count;
}

sub rollback {
    my ($self, $steps) = @_;
    $steps //= 1;
    
    my $applied = $self->applied;
    my @done = sort { $b cmp $a } grep { $applied->{$_} } keys %$applied;
    
    my $count = 0;
    for my $version (splice @done, 0, $steps) {
        my ($m) = grep { $_->{version} eq $version } @{$self->{migrations}};
        next unless $m;
        
        printf "Rolling back %s: %s\n", $m->{version}, $m->{name};
        eval {
            $self->{dbh}->begin_work;
            $m->{down}->($self->{dbh});
            $self->{dbh}->do("DELETE FROM schema_migrations WHERE version=?", undef, $version);
            $self->{dbh}->commit;
        };
        if ($@) {
            $self->{dbh}->rollback;
            printf "  FAILED: %s\n", $@;
            return $count;
        }
        $count++;
    }
    printf "Rolled back %d version(s).\n", $count;
    return $count;
}

sub status {
    my $self    = shift;
    my $applied = $self->applied;
    printf "%-15s %-30s %s\n", "Version", "Name", "Status";
    printf "%s\n", "-" x 55;
    for my $m (@{$self->{migrations}}) {
        printf "%-15s %-30s %s\n", $m->{version}, $m->{name},
            $applied->{$m->{version}} ? "applied" : "pending";
    }
}
}

package main;

printf "=== Schema Migrations ===\n\n";

my $dbh = DBI->connect("dbi:SQLite::memory:", "", "", {RaiseError=>1, AutoCommit=>0});
$dbh->commit;

my $mg = DB::Migrations->new($dbh);

$mg->add("20240101_001", "create_users_table",
    up   => sub { $_[0]->do("CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT UNIQUE, email TEXT)") },
    down => sub { $_[0]->do("DROP TABLE users") },
);

$mg->add("20240101_002", "add_profile_fields",
    up   => sub {
        $_[0]->do("ALTER TABLE users ADD COLUMN full_name TEXT");
        $_[0]->do("ALTER TABLE users ADD COLUMN bio TEXT");
    },
    down => sub { $_[0]->do("DROP TABLE users"); $_[0]->do("CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT UNIQUE, email TEXT)") },
);

$mg->add("20240101_003", "create_posts_table",
    up   => sub { $_[0]->do("CREATE TABLE posts (id INTEGER PRIMARY KEY, user_id INTEGER, title TEXT, body TEXT)") },
    down => sub { $_[0]->do("DROP TABLE posts") },
);

$mg->add("20240115_001", "add_post_timestamps",
    up   => sub { $_[0]->do("ALTER TABLE posts ADD COLUMN created_at INTEGER DEFAULT (strftime('%s','now'))") },
    down => sub { },
);

printf "Initial status:\n";
$mg->status;

printf "\nRunning migrations:\n";
$mg->migrate;

printf "\nFinal status:\n";
$mg->status;

printf "\nRolling back 1:\n";
$mg->rollback(1);

printf "\nStatus after rollback:\n";
$mg->status;

# Verify tables
my @tables = map { $_->[0] } @{$dbh->selectall_arrayref("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")};
printf "\nTables: %s\n", join(", ", @tables);
```

---

## Step 366: Repository Pattern

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

{
package Repository::Base;

sub new {
    my ($class, $dbh) = @_;
    return bless { dbh => $dbh, cache => {} }, $class;
}

sub find        { die ref(shift) . " must implement find" }
sub find_all    { die ref(shift) . " must implement find_all" }
sub save        { die ref(shift) . " must implement save" }
sub delete      { die ref(shift) . " must implement delete" }

sub find_by_id {
    my ($self, $id) = @_;
    return $self->{cache}{$id} if exists $self->{cache}{$id};
    my $obj = $self->find($id);
    $self->{cache}{$id} = $obj if $obj;
    return $obj;
}

sub invalidate_cache { delete $_[0]->{cache}{$_[1]} }
sub clear_cache      { $_[0]->{cache} = {} }
}

{
package User::Repository;
use parent -norequire, 'Repository::Base';

sub _row_to_user {
    my $row = shift;
    return bless { %$row }, "User::Entity";
}

sub find {
    my ($self, $id) = @_;
    my $row = $self->{dbh}->selectrow_hashref("SELECT * FROM users WHERE id=?", undef, $id);
    return $row ? _row_to_user($row) : undef;
}

sub find_by_email {
    my ($self, $email) = @_;
    my $row = $self->{dbh}->selectrow_hashref("SELECT * FROM users WHERE email=?", undef, $email);
    return $row ? _row_to_user($row) : undef;
}

sub find_by_username {
    my ($self, $username) = @_;
    my $row = $self->{dbh}->selectrow_hashref("SELECT * FROM users WHERE username=?", undef, $username);
    return $row ? _row_to_user($row) : undef;
}

sub find_all {
    my ($self, %opts) = @_;
    my $limit  = $opts{limit}  // 50;
    my $offset = $opts{offset} // 0;
    my $order  = $opts{order}  // "id";
    my $rows   = $self->{dbh}->selectall_arrayref(
        "SELECT * FROM users ORDER BY $order LIMIT ? OFFSET ?",
        {Slice=>{}}, $limit, $offset);
    return [map { _row_to_user($_) } @$rows];
}

sub find_with_posts {
    my ($self, $id) = @_;
    my $user = $self->find($id) or return undef;
    my $posts = $self->{dbh}->selectall_arrayref(
        "SELECT * FROM posts WHERE user_id=? ORDER BY id", {Slice=>{}}, $id);
    $user->{posts} = $posts;
    return $user;
}

sub save {
    my ($self, $user) = @_;
    if ($user->{id}) {
        $self->{dbh}->do("UPDATE users SET username=?,email=?,full_name=? WHERE id=?",
            undef, $user->{username}, $user->{email}, $user->{full_name}, $user->{id});
        $self->invalidate_cache($user->{id});
    } else {
        $self->{dbh}->do("INSERT INTO users (username,email,full_name) VALUES (?,?,?)",
            undef, $user->{username}, $user->{email}, $user->{full_name});
        $user->{id} = $self->{dbh}->last_insert_id;
    }
    return $user;
}

sub delete {
    my ($self, $id) = @_;
    $self->{dbh}->do("DELETE FROM users WHERE id=?", undef, $id);
    $self->invalidate_cache($id);
}

sub count { $_[0]->{dbh}->selectrow_array("SELECT COUNT(*) FROM users")+0 }

sub search {
    my ($self, $q) = @_;
    my $rows = $self->{dbh}->selectall_arrayref(
        "SELECT * FROM users WHERE username LIKE ? OR email LIKE ? OR full_name LIKE ?",
        {Slice=>{}}, "%$q%","$q%","%$q%");
    return [map { _row_to_user($_) } @$rows];
}
}

{
package User::Entity;
sub id        { $_[0]->{id} }
sub username  { $_[0]->{username} }
sub email     { $_[0]->{email} }
sub full_name { $_[0]->{full_name} }
sub posts     { $_[0]->{posts} // [] }
sub to_hash   { +{ id=>$_[0]->{id}, username=>$_[0]->{username}, email=>$_[0]->{email} } }
}

package main;

my $dbh = DBI->connect("dbi:SQLite::memory:", "", "", {RaiseError=>1, AutoCommit=>1});
$dbh->do("CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT, email TEXT, full_name TEXT)");
$dbh->do("CREATE TABLE posts (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, title TEXT, body TEXT)");

for my $u (["alice","alice\@test.com","Alice Smith"],["bob","bob\@test.com","Bob Jones"],["carol","carol\@test.com","Carol Davis"]) {
    $dbh->do("INSERT INTO users (username,email,full_name) VALUES (?,?,?)", undef, @$u);
}
for my $p ([1,"Perl Post 1","Content 1"],[1,"Perl Post 2","Content 2"],[2,"Ruby Post","Ruby content"]) {
    $dbh->do("INSERT INTO posts (user_id,title,body) VALUES (?,?,?)", undef, @$p);
}

printf "=== Repository Pattern ===\n\n";

my $repo = User::Repository->new($dbh);

printf "Find by ID:\n";
my $u = $repo->find_by_id(1);
printf "  %s (%s)\n", $u->username, $u->email;

printf "\nFind by email:\n";
my $u2 = $repo->find_by_email("bob\@test.com");
printf "  %s\n", $u2->full_name;

printf "\nAll users (%d):\n", $repo->count;
printf "  %s\n", $_->username for @{$repo->find_all};

printf "\nWith posts:\n";
my $alice = $repo->find_with_posts(1);
printf "  %s has %d posts\n", $alice->username, scalar @{$alice->posts};

printf "\nSearch 'al':\n";
my $found = $repo->search("al");
printf "  %s\n", $_->username for @$found;

# Update via repository
$alice->{full_name} = "Alice J. Smith";
$repo->save($alice);
printf "\nUpdated: %s\n", $repo->find(1)->full_name;
```

---

## Step 367: N+1 Query Prevention

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

{
package Loader;

# DataLoader / batch loading
sub new {
    my ($class, $dbh) = @_;
    return bless { dbh => $dbh, batches => {}, results => {} }, $class;
}

# Eager loading
sub load_with {
    my ($class, $dbh, $primary_rows, $assoc, %opts) = @_;
    return [] unless @$primary_rows;
    
    my $fk    = $opts{foreign_key} // "id";
    my $pk    = $opts{primary_key} // "id";
    my $table = $opts{table};
    my $many  = $opts{many}  // 1;
    my $group = $opts{group} // $fk;  # Key to group by
    
    my @ids = map { $_->{$pk} } @$primary_rows;
    my $placeholders = join(",", ("?") x @ids);
    
    my $rows = $dbh->selectall_arrayref(
        "SELECT * FROM $table WHERE $fk IN ($placeholders) ORDER BY $fk",
        {Slice=>{}}, @ids);
    
    # Group
    my %grouped;
    for my $row (@$rows) {
        if ($many) { push @{$grouped{$row->{$fk}}}, $row }
        else { $grouped{$row->{$fk}} = $row }
    }
    
    # Attach
    for my $row (@$primary_rows) {
        $row->{$assoc} = $grouped{$row->{$pk}} // ($many ? [] : undef);
    }
    
    return $primary_rows;
}

# Query counter for detecting N+1
my $query_count = 0;
my @query_log;

sub reset_counter { $query_count = 0; @query_log = () }
sub query_count   { $query_count }
sub query_log     { @query_log }

sub counted_query {
    my ($dbh, $sql, $bind) = @_;
    $query_count++;
    push @query_log, $sql;
    return $dbh->selectall_arrayref($sql, {Slice=>{}}, ref $bind ? @$bind : ());
}
}

package main;

my $dbh = DBI->connect("dbi:SQLite::memory:", "", "", {RaiseError=>1, AutoCommit=>1});

# Schema
$dbh->do("CREATE TABLE authors  (id INTEGER PRIMARY KEY, name TEXT, country TEXT)");
$dbh->do("CREATE TABLE books    (id INTEGER PRIMARY KEY, author_id INTEGER, title TEXT, year INTEGER)");
$dbh->do("CREATE TABLE reviews  (id INTEGER PRIMARY KEY, book_id INTEGER, rating INTEGER, text TEXT)");

# Seed
for my $a ([1,"Alice","US"],[2,"Bob","UK"],[3,"Carol","CA"]) {
    $dbh->do("INSERT INTO authors VALUES (?,?,?)", undef, @$a) }
for my $b ([1,1,"Perl Basics",2020],[2,1,"Advanced Perl",2021],[3,2,"Ruby Way",2019],[4,2,"Rails Guide",2022],[5,3,"Go Tour",2023]) {
    $dbh->do("INSERT INTO books VALUES (?,?,?,?)", undef, @$b) }
for my $r ([1,1,5,"Excellent"],[2,1,4,"Good"],[3,2,5,"Must read"],[4,3,3,"OK"],[5,4,4,"Helpful"],[6,5,5,"Great"]) {
    $dbh->do("INSERT INTO reviews VALUES (?,?,?,?)", undef, @$r) }

printf "=== N+1 Prevention ===\n\n";

# === BAD: N+1 query ===
printf "--- N+1 (BAD) ---\n";
Loader::reset_counter();

my $authors = Loader::counted_query($dbh, "SELECT * FROM authors");
for my $a (@$authors) {
    my $books = Loader::counted_query($dbh, "SELECT * FROM books WHERE author_id=?", [$a->{id}]);
    $a->{books} = $books;
    for my $b (@$books) {
        my $reviews = Loader::counted_query($dbh, "SELECT * FROM reviews WHERE book_id=?", [$b->{id}]);
        $b->{reviews} = $reviews;
    }
}

printf "Queries: %d (N+1 problem!)\n", Loader::query_count;

# === GOOD: Eager loading ===
printf "\n--- Eager loading (GOOD) ---\n";
Loader::reset_counter();

my $authors2 = Loader::counted_query($dbh, "SELECT * FROM authors");
Loader::load_with($dbh, $authors2, "books",
    table => "books", foreign_key => "author_id", primary_key => "id");

my @all_books = map { @{$_->{books}} } @$authors2;
Loader::load_with($dbh, \@all_books, "reviews",
    table => "reviews", foreign_key => "book_id", primary_key => "id");

printf "Queries: %d (only 3 regardless of data size!)\n", Loader::query_count;

# Display results
printf "\nAuthors with books:\n";
for my $a (@$authors2) {
    printf "  %s (%s): %d books\n", $a->{name}, $a->{country}, scalar @{$a->{books}};
    for my $b (@{$a->{books}}) {
        printf "    - %s (%d) — %d reviews\n", $b->{title}, $b->{year}, scalar @{$b->{reviews}};
    }
}
```

---

## Step 368: Database Seeding

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;
use List::Util qw(shuffle);

{
package DB::Seeder;

my @first_names = qw(Alice Bob Carol Dave Eve Frank Grace Henry Ivy Jack);
my @last_names  = qw(Smith Jones Davis Brown Wilson Taylor Moore Anderson Thomas);
my @domains     = qw(gmail.com yahoo.com outlook.com example.com test.org);
my @categories  = qw(tech science art history fiction biography self-help);
my @lorem       = qw(lorem ipsum dolor sit amet consectetur adipiscing elit sed do eiusmod tempor incididunt ut labore et dolore magna aliqua);

sub new { bless { dbh => $_[1], counts => {} }, $_[0] }

sub _rand_name {
    $first_names[rand @first_names] . " " . $last_names[rand @last_names]
}

sub _rand_email {
    my $name = lc $_[0] // $first_names[rand @first_names];
    $name =~ s/ /./g;
    return "$name\@" . $domains[rand @domains];
}

sub _rand_text {
    my $words = $_[0] // 10;
    join " ", map { $lorem[rand @lorem] } 1..$words;
}

sub _rand_title {
    my @words = qw(Guide Book Introduction Advanced Mastering Learning Professional Complete);
    my @topics = qw(Perl Ruby Python JavaScript Go Rust Web Database Cloud API);
    return $words[rand @words] . " to " . $topics[rand @topics];
}

sub seed_users {
    my ($self, $count) = @_;
    $count //= 10;
    my $sth = $self->{dbh}->prepare(
        "INSERT INTO users (username, email, full_name, bio) VALUES (?,?,?,?)");
    
    for my $i (1..$count) {
        my $name  = _rand_name();
        my $uname = lc($name =~ s/ /_/r) . $i;
        $sth->execute($uname, _rand_email($uname), $name, _rand_text(8));
    }
    $self->{counts}{users} = $count;
    return $self;
}

sub seed_posts {
    my ($self, $per_user) = @_;
    $per_user //= 3;
    my $users = $self->{dbh}->selectall_arrayref("SELECT id FROM users", {Slice=>{}});
    my $sth   = $self->{dbh}->prepare(
        "INSERT INTO posts (user_id, title, body, category, published) VALUES (?,?,?,?,?)");
    
    my $count = 0;
    for my $u (@$users) {
        for (1..$per_user) {
            $sth->execute($u->{id}, _rand_title(), _rand_text(50),
                $categories[rand @categories], rand() > 0.2 ? 1 : 0);
            $count++;
        }
    }
    $self->{counts}{posts} = $count;
    return $self;
}

sub seed_comments {
    my ($self, $per_post) = @_;
    $per_post //= 2;
    my $posts = $self->{dbh}->selectall_arrayref("SELECT id FROM posts WHERE published=1", {Slice=>{}});
    my $users = $self->{dbh}->selectall_arrayref("SELECT id FROM users", {Slice=>{}});
    my $sth   = $self->{dbh}->prepare("INSERT INTO comments (post_id, user_id, body) VALUES (?,?,?)");
    
    my $count = 0;
    for my $p (@$posts) {
        for (1..$per_post) {
            my $u = $users->[rand @$users];
            $sth->execute($p->{id}, $u->{id}, _rand_text(15));
            $count++;
        }
    }
    $self->{counts}{comments} = $count;
    return $self;
}

sub report {
    my $self = shift;
    printf "Seeded: %s\n", join(", ", map { "$_=$self->{counts}{$_}" } sort keys %{$self->{counts}});
}
}

package main;

printf "=== Database Seeding ===\n\n";

my $dbh = DBI->connect("dbi:SQLite::memory:", "", "", {RaiseError=>1, AutoCommit=>1});
$dbh->do("CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT, email TEXT, full_name TEXT, bio TEXT)");
$dbh->do("CREATE TABLE posts (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, title TEXT, body TEXT, category TEXT, published INTEGER DEFAULT 1)");
$dbh->do("CREATE TABLE comments (id INTEGER PRIMARY KEY AUTOINCREMENT, post_id INTEGER, user_id INTEGER, body TEXT)");

my $seeder = DB::Seeder->new($dbh);
$seeder->seed_users(5)->seed_posts(3)->seed_comments(2);
$seeder->report;

printf "\nSample data:\n";
my $sample = $dbh->selectall_arrayref(
    "SELECT u.username, COUNT(DISTINCT p.id) as posts, COUNT(c.id) as comments FROM users u LEFT JOIN posts p ON p.user_id=u.id LEFT JOIN comments c ON c.user_id=u.id GROUP BY u.id ORDER BY u.id",
    {Slice=>{}});
printf "  %-20s posts=%d  comments=%d\n", $_->{username}, $_->{posts}, $_->{comments} for @$sample;

printf "\nCategories:\n";
my $cats = $dbh->selectall_arrayref("SELECT category, COUNT(*) as n FROM posts GROUP BY category ORDER BY n DESC", {Slice=>{}});
printf "  %-15s %d\n", $_->{category}, $_->{n} for @$cats;
```

---

## Step 369: Full-text Search

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

{
package DB::FullTextSearch;

sub new {
    my ($class, $dbh, %opts) = @_;
    return bless {
        dbh    => $dbh,
        table  => $opts{table},
        fields => $opts{fields} // [],
        fts_ok => undef,
    }, $class;
}

sub setup {
    my $self = shift;
    my $tbl  = $self->{table};
    my @flds = @{$self->{fields}};
    
    # Try FTS5
    eval {
        $self->{dbh}->do(qq{
            CREATE VIRTUAL TABLE IF NOT EXISTS ${tbl}_fts
            USING fts5(content="${tbl}", ${\ join(",",@flds) })
        });
        $self->{fts_ok} = "fts5";
    };
    
    # Fallback: FTS4
    unless ($self->{fts_ok}) {
        eval {
            $self->{dbh}->do(qq{
                CREATE VIRTUAL TABLE IF NOT EXISTS ${tbl}_fts
                USING fts4(content="${tbl}", ${\ join(",",@flds) })
            });
            $self->{fts_ok} = "fts4";
        };
    }
    
    if ($self->{fts_ok}) {
        # Populate FTS table
        my $select = join(",", @flds);
        $self->{dbh}->do("INSERT INTO ${tbl}_fts SELECT rowid,$select FROM $tbl");
        printf "  FTS setup: %s\n", $self->{fts_ok};
    } else {
        printf "  FTS not available — using LIKE fallback\n";
    }
    return $self;
}

sub search {
    my ($self, $query, %opts) = @_;
    my $limit  = $opts{limit}  // 10;
    my $offset = $opts{offset} // 0;
    my $tbl    = $self->{table};
    
    if ($self->{fts_ok}) {
        # Use FTS
        my $rows = eval {
            $self->{dbh}->selectall_arrayref(
                "SELECT t.* FROM ${tbl}_fts f JOIN $tbl t ON t.rowid=f.rowid WHERE ${tbl}_fts MATCH ? ORDER BY rank LIMIT ? OFFSET ?",
                {Slice=>{}}, $query, $limit, $offset);
        };
        return $rows if $rows && !$@;
    }
    
    # LIKE fallback
    my @flds  = @{$self->{fields}};
    my @where = map { "$_ LIKE ?" } @flds;
    my @binds = map { "%$query%" } @flds;
    
    return $self->{dbh}->selectall_arrayref(
        "SELECT * FROM $tbl WHERE " . join(" OR ", @where) . " LIMIT ? OFFSET ?",
        {Slice=>{}}, @binds, $limit, $offset);
}

sub search_count {
    my ($self, $query) = @_;
    my @flds  = @{$self->{fields}};
    my @where = map { "$_ LIKE ?" } @flds;
    my @binds = map { "%$query%" } @flds;
    return $self->{dbh}->selectrow_array(
        "SELECT COUNT(*) FROM $self->{table} WHERE " . join(" OR ", @where),
        undef, @binds)+0;
}

sub highlight {
    my ($class, $text, $query, %opts) = @_;
    my $pre  = $opts{pre}  // "<mark>";
    my $post = $opts{post} // "</mark>";
    (my $r = $text) =~ s/(\Q$query\E)/${pre}${1}${post}/gi;
    return $r;
}
}

package main;

printf "=== Full-text Search ===\n\n";

my $dbh = DBI->connect("dbi:SQLite::memory:", "", "", {RaiseError=>1, AutoCommit=>1});
$dbh->do("CREATE TABLE articles (id INTEGER PRIMARY KEY AUTOINCREMENT, title TEXT, body TEXT, author TEXT, tags TEXT)");

my @articles = (
    ["Perl Programming Basics",     "Learn Perl from scratch. Variables, loops, and subroutines.",           "Alice", "perl,beginner"],
    ["Advanced Perl Techniques",    "Master references, closures, and metaprogramming in Perl.",             "Bob",   "perl,advanced"],
    ["Template Toolkit Guide",      "Complete guide to TT2 templates for Perl web development.",             "Alice", "perl,web,templates"],
    ["Python vs Perl",              "Comparing Python and Perl for data processing and web development.",    "Carol", "python,perl,comparison"],
    ["DBI Database Access",         "Using DBI and SQLite for database-driven Perl applications.",           "Bob",   "perl,database,dbi"],
    ["JavaScript Basics",           "Getting started with JavaScript for web development.",                  "Dave",  "javascript,web,beginner"],
    ["Moose Object System",         "Object-oriented programming in Perl using Moose framework.",            "Alice", "perl,oop,moose"],
    ["Web Scraping with Perl",      "Extract data from websites using LWP and HTML parsing in Perl.",        "Bob",   "perl,web,scraping"],
);

for my $a (@articles) {
    $dbh->do("INSERT INTO articles (title,body,author,tags) VALUES (?,?,?,?)", undef, @$a);
}

my $fts = DB::FullTextSearch->new($dbh,
    table  => "articles",
    fields => ["title", "body", "tags"],
);
$fts->setup;

printf "\n";
for my $query ("Perl", "web development", "database", "JavaScript", "OOP", "Python Perl") {
    my $results = $fts->search($query, limit => 5);
    printf "Search '%s': %d results\n", $query, scalar @$results;
    for my $r (@{$results}[0..1]) {
        next unless $r;
        my $highlighted = DB::FullTextSearch->highlight($r->{title}, $query, pre=>"[", post=>"]");
        printf "  - %s\n", $highlighted;
    }
}

printf "\nHighlight: %s\n", DB::FullTextSearch->highlight(
    "Perl is great for web development with Perl tools",
    "Perl", pre => ">>", post => "<<");
```

---

## Step 370: Capstone — ORM Framework

```perl
#!/usr/bin/perl
# orm_framework.pl — Complete ORM system
use strict;
use warnings;
use DBI;

my $DBH = DBI->connect("dbi:SQLite::memory:", "", "", {RaiseError=>1, AutoCommit=>1});

# Schema
for my $sql (
    "CREATE TABLE categories (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT UNIQUE, description TEXT)",
    "CREATE TABLE products (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, price REAL, stock INTEGER DEFAULT 0, category_id INTEGER, active INTEGER DEFAULT 1, created_at INTEGER DEFAULT (strftime('%s','now')))",
    "CREATE TABLE orders (id INTEGER PRIMARY KEY AUTOINCREMENT, customer TEXT, total REAL, status TEXT DEFAULT 'pending', created_at INTEGER DEFAULT (strftime('%s','now')))",
    "CREATE TABLE order_items (id INTEGER PRIMARY KEY AUTOINCREMENT, order_id INTEGER, product_id INTEGER, qty INTEGER, unit_price REAL)",
) { $DBH->do($sql) }

# Seed
$DBH->do("INSERT INTO categories (name,description) VALUES (?,?)", undef, "Electronics", "Electronic devices");
$DBH->do("INSERT INTO categories (name,description) VALUES (?,?)", undef, "Books", "Programming books");
for my $p ([1,"Laptop",999.99,10],[1,"Keyboard",49.99,50],[1,"Mouse",29.99,100],[2,"Perl Book",39.99,25],[2,"Python Book",34.99,15]) {
    $DBH->do("INSERT INTO products (category_id,name,price,stock) VALUES (?,?,?,?)", undef, @$p) }

# Application
{
package Store;

sub low_stock_alert {
    my ($threshold) = @_;
    $threshold //= 20;
    return $DBH->selectall_arrayref(
        "SELECT p.name, p.stock, c.name as category FROM products p JOIN categories c ON c.id=p.category_id WHERE p.stock < ? AND p.active=1 ORDER BY p.stock",
        {Slice=>{}}, $threshold);
}

sub category_summary {
    return $DBH->selectall_arrayref(
        "SELECT c.name, COUNT(p.id) as product_count, AVG(p.price) as avg_price, SUM(p.stock) as total_stock FROM categories c LEFT JOIN products p ON p.category_id=c.id GROUP BY c.id ORDER BY c.name",
        {Slice=>{}});
}

sub place_order {
    my ($customer, @items) = @_;
    
    $DBH->begin_work;
    eval {
        my $total = 0;
        for my $item (@items) {
            my $p = $DBH->selectrow_hashref("SELECT * FROM products WHERE id=? AND active=1 FOR UPDATE", undef, $item->{product_id});
            die "Product $item->{product_id} not found\n" unless $p;
            die "Insufficient stock for $p->{name}\n" if $p->{stock} < $item->{qty};
            $total += $p->{price} * $item->{qty};
            $item->{unit_price} = $p->{price};
        }
        
        $DBH->do("INSERT INTO orders (customer, total) VALUES (?,?)", undef, $customer, $total);
        my $order_id = $DBH->last_insert_id;
        
        for my $item (@items) {
            $DBH->do("INSERT INTO order_items (order_id,product_id,qty,unit_price) VALUES (?,?,?,?)",
                undef, $order_id, $item->{product_id}, $item->{qty}, $item->{unit_price});
            $DBH->do("UPDATE products SET stock=stock-? WHERE id=?",
                undef, $item->{qty}, $item->{product_id});
        }
        
        $DBH->commit;
        return $order_id;
    };
    if ($@) {
        $DBH->rollback;
        die $@;
    }
}

sub order_details {
    my ($order_id) = @_;
    my $order = $DBH->selectrow_hashref("SELECT * FROM orders WHERE id=?", undef, $order_id);
    return unless $order;
    my $items = $DBH->selectall_arrayref(
        "SELECT oi.*, p.name as product_name FROM order_items oi JOIN products p ON p.id=oi.product_id WHERE oi.order_id=?",
        {Slice=>{}}, $order_id);
    $order->{items} = $items;
    return $order;
}
}

printf "=== ORM Framework Capstone (Store) ===\n\n";

printf "Category summary:\n";
for my $cat (@{Store::category_summary()}) {
    printf "  %-15s products=%d  avg_price=\$%.2f  stock=%d\n",
        $cat->{name}, $cat->{product_count}, $cat->{avg_price}//0, $cat->{total_stock}//0;
}

printf "\nLow stock (< 20):\n";
for my $p (@{Store::low_stock_alert(20)}) {
    printf "  %-20s stock=%d  (%s)\n", $p->{name}, $p->{stock}, $p->{category};
}

printf "\nPlacing orders:\n";
for my $order (
    ["Alice", [{product_id=>1,qty=>1},{product_id=>2,qty=>2}]],
    ["Bob",   [{product_id=>4,qty=>1},{product_id=>5,qty=>1}]],
) {
    my ($customer, $items) = @$order;
    my $order_id = eval { Store::place_order($customer, @$items) };
    if ($@) { printf "  %s: FAILED — %s", $customer, $@ }
    else {
        my $od = Store::order_details($order_id);
        printf "  %s: order #%d total=\$%.2f  items=%d\n",
            $customer, $order_id, $od->{total}, scalar @{$od->{items}};
    }
}

printf "\nInventory after orders:\n";
my $inv = $DBH->selectall_arrayref("SELECT name, stock FROM products ORDER BY category_id, name", {Slice=>{}});
printf "  %-20s stock=%d\n", $_->{name}, $_->{stock} for @$inv;
```

---

## สรุป Part 37 — Advanced DBI & ORM Patterns

### สิ่งที่เรียนรู้:
- **Connection Pool** — Acquire/release/reconnect/stats
- **Fluent Query Builder** — Chainable where/join/order/group/limit/page
- **Advanced ActiveRecord** — Callbacks, validations, associations (has_many/belongs_to)
- **Transactions & Savepoints** — Nested transactions with SAVEPOINT
- **Schema Migrations** — Versioned up/down/rollback/status
- **Repository Pattern** — Clean separation of data access
- **N+1 Prevention** — Eager loading with batch queries
- **Database Seeding** — Fake data generation
- **Full-text Search** — FTS5/FTS4 with LIKE fallback
- **ORM Capstone** — Store application with orders/inventory

**ถัดไป: [Part 38 — REST API Design Patterns](part_38.md)**
