# Part 19: DBI Database Programming
## Steps 181-190: การเขียนโปรแกรมฐานข้อมูลด้วย DBI

---

## Step 181: DBI พื้นฐาน

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

# =====================
# DBI — Database Interface
# =====================

# Connect to SQLite
my $dbh = DBI->connect(
    "dbi:SQLite:dbname=/tmp/mydb.db",
    "",      # username (ไม่ใช้กับ SQLite)
    "",      # password
    {
        RaiseError     => 1,        # die on error
        AutoCommit     => 1,        # auto-commit
        sqlite_unicode => 1,        # UTF-8 support
        PrintError     => 0,        # no auto-print errors
    }
);

print "Connected: $dbh\n";

# =====================
# DDL — Create tables
# =====================

$dbh->do(<<'SQL');
CREATE TABLE IF NOT EXISTS employees (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    first_name TEXT NOT NULL,
    last_name  TEXT NOT NULL,
    email      TEXT UNIQUE,
    salary     REAL DEFAULT 0,
    dept       TEXT,
    hired_date DATE DEFAULT (date('now'))
)
SQL

$dbh->do(<<'SQL');
CREATE TABLE IF NOT EXISTS departments (
    id   INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT UNIQUE NOT NULL,
    head TEXT
)
SQL

# =====================
# INSERT — Adding data
# =====================

# Simple insert
$dbh->do("INSERT OR IGNORE INTO departments (name, head) VALUES ('Engineering','Alice')");
$dbh->do("INSERT OR IGNORE INTO departments (name, head) VALUES ('Marketing','Bob')");
$dbh->do("INSERT OR IGNORE INTO departments (name, head) VALUES ('Finance','Carol')");

# Insert with placeholders (parameterized query — ALWAYS use this!)
my $insert = $dbh->prepare(
    "INSERT INTO employees (first_name, last_name, email, salary, dept) VALUES (?,?,?,?,?)"
);

my @staff = (
    ["Alice",   "Smith",   "alice\@co.com",  85000, "Engineering"],
    ["Bob",     "Jones",   "bob\@co.com",    72000, "Marketing"],
    ["Carol",   "Davis",   "carol\@co.com",  90000, "Engineering"],
    ["Dave",    "Wilson",  "dave\@co.com",   67000, "Finance"],
    ["Eve",     "Brown",   "eve\@co.com",    75000, "Engineering"],
    ["Frank",   "Miller",  "frank\@co.com",  68000, "Marketing"],
);

# Clear old data
$dbh->do("DELETE FROM employees");

for my $person (@staff) {
    $insert->execute(@$person);
}

printf "Inserted %d employees\n", scalar @staff;

# =====================
# SELECT — Reading data
# =====================

# fetchall_arrayref
my $all = $dbh->selectall_arrayref(
    "SELECT id, first_name, last_name, salary, dept FROM employees ORDER BY salary DESC"
);

print "\nAll employees by salary:\n";
printf "  %-3s %-12s %-12s %8s %-12s\n", "ID", "First", "Last", "Salary", "Dept";
print "  " . "-" x 50 . "\n";
for my $row (@$all) {
    printf "  %-3d %-12s %-12s %8.0f %-12s\n", @$row;
}

# fetchall_hashref style
my $hash_rows = $dbh->selectall_arrayref(
    "SELECT * FROM employees WHERE dept=?",
    { Slice => {} },
    "Engineering"
);

print "\nEngineering team:\n";
for my $e (@$hash_rows) {
    printf "  %s %s — \$%.0f\n", $e->{first_name}, $e->{last_name}, $e->{salary};
}

# Single row
my $best = $dbh->selectrow_hashref(
    "SELECT * FROM employees ORDER BY salary DESC LIMIT 1"
);
printf "\nHighest paid: %s %s (\$%.0f)\n",
    $best->{first_name}, $best->{last_name}, $best->{salary};

# Aggregate
my ($avg) = $dbh->selectrow_array("SELECT AVG(salary) FROM employees");
printf "Average salary: \$%.0f\n", $avg;

# =====================
# UPDATE and DELETE
# =====================

my $rows = $dbh->do(
    "UPDATE employees SET salary = salary * 1.10 WHERE dept = ?",
    undef, "Engineering"
);
printf "\nRaised salary for %d Engineering employees\n", $rows;

$dbh->do("DELETE FROM employees WHERE salary < ?", undef, 60000);

# =====================
# Aggregate queries
# =====================

my $dept_stats = $dbh->selectall_arrayref(<<'SQL', { Slice => {} });
SELECT dept,
       COUNT(*) AS count,
       AVG(salary) AS avg_salary,
       MAX(salary) AS max_salary,
       MIN(salary) AS min_salary
FROM employees
GROUP BY dept
ORDER BY avg_salary DESC
SQL

print "\nDepartment stats:\n";
for my $d (@$dept_stats) {
    printf "  %-12s count=%d avg=\$%.0f max=\$%.0f\n",
        $d->{dept}, $d->{count}, $d->{avg_salary}, $d->{max_salary};
}

$dbh->disconnect;
print "\nDone.\n";
```

---

## Step 182: Prepared Statements และ Transactions

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

my $dbh = DBI->connect("dbi:SQLite:dbname=/tmp/bank.db","","",
    { RaiseError => 1, AutoCommit => 1, sqlite_unicode => 1 });

$dbh->do(<<'SQL');
CREATE TABLE IF NOT EXISTS accounts (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    name    TEXT NOT NULL,
    balance REAL DEFAULT 0 CHECK(balance >= 0)
)
SQL

$dbh->do(<<'SQL');
CREATE TABLE IF NOT EXISTS transactions (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    from_acct   INTEGER,
    to_acct     INTEGER,
    amount      REAL,
    note        TEXT,
    created_at  DATETIME DEFAULT (datetime('now')),
    FOREIGN KEY (from_acct) REFERENCES accounts(id),
    FOREIGN KEY (to_acct)   REFERENCES accounts(id)
)
SQL

# Seed accounts
$dbh->do("DELETE FROM accounts");
$dbh->do("DELETE FROM transactions");
$dbh->do("INSERT INTO accounts (id,name,balance) VALUES (1,'Alice',10000)");
$dbh->do("INSERT INTO accounts (id,name,balance) VALUES (2,'Bob',5000)");
$dbh->do("INSERT INTO accounts (id,name,balance) VALUES (3,'Carol',8000)");

# =====================
# Prepared statement
# =====================

my $sth_balance = $dbh->prepare("SELECT balance FROM accounts WHERE id=?");

sub get_balance {
    my $id = shift;
    $sth_balance->execute($id);
    my ($bal) = $sth_balance->fetchrow_array;
    return $bal;
}

printf "Initial balances:\n";
printf "  Alice: \$%.0f\n", get_balance(1);
printf "  Bob:   \$%.0f\n", get_balance(2);
printf "  Carol: \$%.0f\n", get_balance(3);

# =====================
# Transaction — ACID
# =====================

sub transfer {
    my ($from_id, $to_id, $amount, $note) = @_;
    
    # Begin transaction
    $dbh->{AutoCommit} = 0;
    
    eval {
        my $from_bal = get_balance($from_id);
        die "Insufficient funds (have $from_bal, need $amount)\n"
            if $from_bal < $amount;
        
        $dbh->do("UPDATE accounts SET balance = balance - ? WHERE id = ?",
            undef, $amount, $from_id);
        
        $dbh->do("UPDATE accounts SET balance = balance + ? WHERE id = ?",
            undef, $amount, $to_id);
        
        $dbh->do("INSERT INTO transactions (from_acct,to_acct,amount,note) VALUES (?,?,?,?)",
            undef, $from_id, $to_id, $amount, $note);
        
        $dbh->commit;
        printf "  Transferred \$%.0f from acct %d to acct %d\n", $amount, $from_id, $to_id;
    };
    
    if ($@) {
        $dbh->rollback;
        print "  Transfer failed: $@";
    }
    
    $dbh->{AutoCommit} = 1;
}

print "\nTransactions:\n";
transfer(1, 2, 2000, "Payment for services");
transfer(2, 3, 1500, "Rent payment");
transfer(3, 1, 500,  "Refund");
transfer(3, 1, 9999, "This should fail");   # Should fail — insufficient

printf "\nFinal balances:\n";
my $accts = $dbh->selectall_arrayref("SELECT id,name,balance FROM accounts", { Slice => {} });
printf "  %s: \$%.0f\n", $_->{name}, $_->{balance} for @$accts;

# Transaction history
print "\nTransaction log:\n";
my $log = $dbh->selectall_arrayref(<<'SQL', { Slice => {} });
SELECT t.id, a1.name AS from_name, a2.name AS to_name, t.amount, t.note, t.created_at
FROM transactions t
JOIN accounts a1 ON t.from_acct = a1.id
JOIN accounts a2 ON t.to_acct   = a2.id
SQL

for my $tx (@$log) {
    printf "  #%d: %s → %s \$%.0f (%s)\n",
        $tx->{id}, $tx->{from_name}, $tx->{to_name}, $tx->{amount}, $tx->{note};
}

$dbh->disconnect;
```

---

## Step 183: ORM Pattern

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

# =====================
# Simple ORM (Object-Relational Mapping)
# =====================

package ORM::Base;

my $DBH;

sub set_dbh { $DBH = $_[1] }
sub dbh     { $DBH }

sub new {
    my ($class, %data) = @_;
    return bless { _data => \%data, _dirty => {} }, $class;
}

sub table  { die ref(shift) . "->table() not implemented" }
sub fields { die ref(shift) . "->fields() not implemented" }

sub AUTOLOAD {
    my $self = shift;
    our $AUTOLOAD;
    my $name = $AUTOLOAD;
    $name =~ s/.*:://;
    
    return if $name eq 'DESTROY';
    
    if (@_) {
        # Setter
        $self->{_data}{$name}  = $_[0];
        $self->{_dirty}{$name} = 1;
        return $self;
    } else {
        # Getter
        return $self->{_data}{$name};
    }
}

sub find {
    my ($class, $id) = @_;
    my $row = $DBH->selectrow_hashref(
        "SELECT * FROM " . $class->table . " WHERE id=?", undef, $id
    );
    return undef unless $row;
    return $class->new(%$row);
}

sub where {
    my ($class, %conds) = @_;
    my @keys = keys %conds;
    my $sql = "SELECT * FROM " . $class->table;
    $sql .= " WHERE " . join(" AND ", map { "$_ = ?" } @keys) if @keys;
    
    my $rows = $DBH->selectall_arrayref($sql, { Slice => {} }, @conds{@keys});
    return map { $class->new(%$_) } @$rows;
}

sub all {
    my $class = shift;
    my $rows = $DBH->selectall_arrayref(
        "SELECT * FROM " . $class->table, { Slice => {} }
    );
    return map { $class->new(%$_) } @$rows;
}

sub save {
    my $self = shift;
    my $class = ref $self;
    
    if ($self->{_data}{id}) {
        # UPDATE
        return unless %{$self->{_dirty}};
        my @fields = keys %{$self->{_dirty}};
        my $sql = "UPDATE " . $self->table .
                  " SET " . join(", ", map { "$_ = ?" } @fields) .
                  " WHERE id = ?";
        $DBH->do($sql, undef, @{$self->{_data}}{@fields}, $self->{_data}{id});
        $self->{_dirty} = {};
    } else {
        # INSERT
        my @fields = grep { $_ ne 'id' } keys %{$self->{_data}};
        my $sql = "INSERT INTO " . $self->table .
                  " (" . join(",", @fields) . ")" .
                  " VALUES (" . join(",", ("?") x @fields) . ")";
        $DBH->do($sql, undef, @{$self->{_data}}{@fields});
        $self->{_data}{id} = $DBH->last_insert_id;
    }
    
    return $self;
}

sub delete {
    my $self = shift;
    $DBH->do("DELETE FROM " . $self->table . " WHERE id=?", undef, $self->{_data}{id});
    delete $self->{_data}{id};
}

sub to_hash {
    my $self = shift;
    return %{$self->{_data}};
}

sub count {
    my ($class, %where) = @_;
    my @keys = keys %where;
    my $sql = "SELECT COUNT(*) FROM " . $class->table;
    $sql .= " WHERE " . join(" AND ", map { "$_ = ?" } @keys) if @keys;
    my ($c) = $DBH->selectrow_array($sql, undef, @where{@keys});
    return $c;
}

# =====================
# Model definitions
# =====================

package Model::User;
use parent -norequire, 'ORM::Base';

sub table  { 'users' }
sub fields { [qw(id name email age)] }

sub posts {
    my $self = shift;
    return Model::Post->where(user_id => $self->id);
}

package Model::Post;
use parent -norequire, 'ORM::Base';

sub table  { 'posts' }
sub fields { [qw(id user_id title body created_at)] }

sub author {
    my $self = shift;
    return Model::User->find($self->user_id);
}

# =====================
# Main
# =====================

package main;

my $dbh = DBI->connect("dbi:SQLite:dbname=/tmp/orm_demo.db","","",
    { RaiseError => 1, AutoCommit => 1 });

$dbh->do("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, email TEXT, age INTEGER)");
$dbh->do("CREATE TABLE IF NOT EXISTS posts (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, title TEXT, body TEXT, created_at DATETIME DEFAULT (datetime('now')))");
$dbh->do("DELETE FROM users");
$dbh->do("DELETE FROM posts");

ORM::Base->set_dbh($dbh);

# Create users
my $alice = Model::User->new(name => "Alice", email => "alice\@example.com", age => 28)->save;
my $bob   = Model::User->new(name => "Bob",   email => "bob\@example.com",   age => 35)->save;

printf "Created user: %s (id=%d)\n", $alice->name, $alice->id;
printf "Created user: %s (id=%d)\n", $bob->name,   $bob->id;

# Create posts
Model::Post->new(user_id => $alice->id, title => "Hello Perl!", body => "My first post")->save;
Model::Post->new(user_id => $alice->id, title => "DBI is great", body => "Database blog")->save;
Model::Post->new(user_id => $bob->id,   title => "Bob's blog",   body => "Hello world")->save;

# Find and update
$alice->age(29)->save;
printf "Alice's updated age: %d\n", Model::User->find($alice->id)->age;

# Query
print "\nAll users:\n";
for my $u (Model::User->all) {
    my @posts = $u->posts;
    printf "  %s (%d) — %d posts\n", $u->name, $u->age, scalar @posts;
    for my $p (@posts) {
        printf "    - %s\n", $p->title;
    }
}

printf "\nTotal users: %d\n", Model::User->count;
printf "Alice's posts: %d\n", Model::Post->count(user_id => $alice->id);

$dbh->disconnect;
```

---

## Step 184: Query Builder

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# SQL Query Builder
# =====================

package QueryBuilder;

sub new {
    my ($class, $dbh, $table) = @_;
    return bless {
        dbh     => $dbh,
        table   => $table,
        selects  => ['*'],
        wheres  => [],
        params  => [],
        orders  => [],
        limit   => undef,
        offset  => undef,
        joins   => [],
        groups  => [],
        havings => [],
    }, $class;
}

sub select {
    my ($self, @cols) = @_;
    $self->{selects} = \@cols;
    return $self;
}

sub where {
    my ($self, $cond, @params) = @_;
    push @{$self->{wheres}}, $cond;
    push @{$self->{params}}, @params;
    return $self;
}

sub order_by {
    my ($self, $col, $dir) = @_;
    $dir = uc($dir // 'ASC');
    push @{$self->{orders}}, "$col $dir";
    return $self;
}

sub limit {
    my ($self, $n) = @_;
    $self->{limit} = $n;
    return $self;
}

sub offset {
    my ($self, $n) = @_;
    $self->{offset} = $n;
    return $self;
}

sub join_table {
    my ($self, $type, $table, $on) = @_;
    push @{$self->{joins}}, "$type JOIN $table ON $on";
    return $self;
}

sub group_by {
    my ($self, @cols) = @_;
    push @{$self->{groups}}, @cols;
    return $self;
}

sub having {
    my ($self, $cond, @params) = @_;
    push @{$self->{havings}}, $cond;
    push @{$self->{params}}, @params;
    return $self;
}

sub to_sql {
    my $self = shift;
    
    my $sql = "SELECT " . join(", ", @{$self->{selects}}) . " FROM $self->{table}";
    
    $sql .= " " . join(" ", @{$self->{joins}}) if @{$self->{joins}};
    
    if (@{$self->{wheres}}) {
        $sql .= " WHERE " . join(" AND ", @{$self->{wheres}});
    }
    
    if (@{$self->{groups}}) {
        $sql .= " GROUP BY " . join(", ", @{$self->{groups}});
    }
    
    if (@{$self->{havings}}) {
        $sql .= " HAVING " . join(" AND ", @{$self->{havings}});
    }
    
    if (@{$self->{orders}}) {
        $sql .= " ORDER BY " . join(", ", @{$self->{orders}});
    }
    
    $sql .= " LIMIT $self->{limit}"   if defined $self->{limit};
    $sql .= " OFFSET $self->{offset}" if defined $self->{offset};
    
    return ($sql, @{$self->{params}});
}

sub get {
    my $self = shift;
    my ($sql, @params) = $self->to_sql;
    return $self->{dbh}->selectall_arrayref($sql, { Slice => {} }, @params);
}

sub first {
    my $self = shift;
    $self->limit(1);
    my ($sql, @params) = $self->to_sql;
    return $self->{dbh}->selectrow_hashref($sql, undef, @params);
}

sub count_query {
    my $self = shift;
    my $clone = { %$self };
    $clone->{selects} = ['COUNT(*) AS total'];
    $clone->{orders}  = [];
    $clone->{limit}   = undef;
    $clone->{offset}  = undef;
    my $qb = bless $clone, ref($self);
    my ($sql, @params) = $qb->to_sql;
    my ($total) = $self->{dbh}->selectrow_array($sql, undef, @params);
    return $total;
}

# =====================
# Demo
# =====================

package main;

use DBI;

my $dbh = DBI->connect("dbi:SQLite:dbname=/tmp/qb_demo.db","","",
    { RaiseError => 1, AutoCommit => 1 });

$dbh->do("CREATE TABLE IF NOT EXISTS products (
    id INTEGER PRIMARY KEY,
    name TEXT,
    category TEXT,
    price REAL,
    stock INTEGER,
    active INTEGER DEFAULT 1
)");

$dbh->do("DELETE FROM products");
$dbh->do("INSERT INTO products (name,category,price,stock) VALUES
    ('Apple','Fruit',1.99,100),
    ('Banana','Fruit',0.99,200),
    ('Carrot','Vegetable',1.49,80),
    ('Broccoli','Vegetable',2.99,60),
    ('Milk','Dairy',3.99,40),
    ('Cheese','Dairy',5.99,30),
    ('Bread','Bakery',2.49,50),
    ('Croissant','Bakery',1.99,25)");

my $qb = QueryBuilder->new($dbh, 'products');

# Simple query
my $fruits = $qb->select('name','price')
               ->where('category = ?', 'Fruit')
               ->order_by('price', 'asc')
               ->get;

print "Fruits:\n";
printf "  %-15s \$%.2f\n", $_->{name}, $_->{price} for @$fruits;

# Complex query
my $qb2 = QueryBuilder->new($dbh, 'products');
my $expensive = $qb2->select('name','category','price')
                   ->where('price > ?', 2.00)
                   ->where('active = ?', 1)
                   ->order_by('category')
                   ->order_by('price', 'desc')
                   ->limit(5)
                   ->get;

print "\nExpensive items (price > \$2):\n";
printf "  %-15s %-12s \$%.2f\n", $_->{name}, $_->{category}, $_->{price}
    for @$expensive;

# Group by
my $qb3 = QueryBuilder->new($dbh, 'products');
my ($sql, @params) = $qb3->select('category', 'COUNT(*) AS count', 'AVG(price) AS avg_price')
                         ->group_by('category')
                         ->having('COUNT(*) > ?', 1)
                         ->order_by('avg_price', 'desc')
                         ->to_sql;

print "\nGenerated SQL:\n  $sql\n";
my $stats = $dbh->selectall_arrayref($sql, { Slice => {} }, @params);
printf "  %-12s %d items avg \$%.2f\n", $_->{category}, $_->{count}, $_->{avg_price}
    for @$stats;

$dbh->disconnect;
```

---

## Step 185: Database Migration System

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;
use POSIX qw(strftime);

# =====================
# Database Migration
# =====================

package Migration;

sub new {
    my ($class, $dbh) = @_;
    my $self = bless { dbh => $dbh }, $class;
    $self->_init;
    return $self;
}

sub _init {
    my $self = shift;
    $self->{dbh}->do(<<'SQL');
CREATE TABLE IF NOT EXISTS schema_migrations (
    version    TEXT PRIMARY KEY,
    applied_at DATETIME DEFAULT (datetime('now'))
)
SQL
}

sub applied {
    my $self = shift;
    my $rows = $self->{dbh}->selectall_arrayref(
        "SELECT version FROM schema_migrations ORDER BY version"
    );
    return map { $_->[0] } @$rows;
}

sub pending {
    my ($self, @migrations) = @_;
    my %done = map { $_ => 1 } $self->applied;
    return grep { !$done{$_->{version}} } @migrations;
}

sub run_up {
    my ($self, $migration) = @_;
    
    $self->{dbh}{AutoCommit} = 0;
    eval {
        $migration->{up}->($self->{dbh});
        $self->{dbh}->do("INSERT INTO schema_migrations (version) VALUES (?)",
            undef, $migration->{version});
        $self->{dbh}->commit;
        printf "  [+] Applied: %s\n", $migration->{version};
    };
    if ($@) {
        $self->{dbh}->rollback;
        printf "  [!] Failed: %s — %s\n", $migration->{version}, $@;
    }
    $self->{dbh}{AutoCommit} = 1;
}

sub run_down {
    my ($self, $migration) = @_;
    
    $self->{dbh}{AutoCommit} = 0;
    eval {
        $migration->{down}->($self->{dbh}) if $migration->{down};
        $self->{dbh}->do("DELETE FROM schema_migrations WHERE version=?",
            undef, $migration->{version});
        $self->{dbh}->commit;
        printf "  [-] Rolled back: %s\n", $migration->{version};
    };
    if ($@) {
        $self->{dbh}->rollback;
        printf "  [!] Rollback failed: %s — %s\n", $migration->{version}, $@;
    }
    $self->{dbh}{AutoCommit} = 1;
}

sub migrate {
    my ($self, @migrations) = @_;
    my @todo = $self->pending(@migrations);
    
    if (@todo) {
        printf "Running %d migration(s)...\n", scalar @todo;
        $self->run_up($_) for @todo;
    } else {
        print "Already up to date.\n";
    }
}

sub rollback {
    my ($self, $steps, @migrations) = @_;
    $steps //= 1;
    my @applied = reverse $self->applied;
    my %by_ver  = map { $_->{version} => $_ } @migrations;
    
    for my $ver (@applied[0..$steps-1]) {
        $self->run_down($by_ver{$ver}) if $by_ver{$ver};
    }
}

# =====================
# Define migrations
# =====================

package main;

my $dbh = DBI->connect("dbi:SQLite:dbname=/tmp/migrate_demo.db","","",
    { RaiseError => 1, AutoCommit => 1 });

my @migrations = (
    {
        version => '001_create_users',
        up => sub {
            my $dbh = shift;
            $dbh->do("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT, email TEXT, created_at DATETIME DEFAULT (datetime('now')))");
        },
        down => sub { shift->do("DROP TABLE users") },
    },
    {
        version => '002_create_posts',
        up => sub {
            my $dbh = shift;
            $dbh->do("CREATE TABLE posts (id INTEGER PRIMARY KEY, user_id INTEGER, title TEXT, body TEXT)");
            $dbh->do("CREATE INDEX idx_posts_user ON posts(user_id)");
        },
        down => sub {
            shift->do("DROP INDEX IF EXISTS idx_posts_user");
            shift->do("DROP TABLE posts");
        },
    },
    {
        version => '003_add_user_role',
        up => sub {
            shift->do("ALTER TABLE users ADD COLUMN role TEXT DEFAULT 'user'");
        },
        down => sub {
            # SQLite doesn't support DROP COLUMN easily
            shift->do("CREATE TABLE users_tmp AS SELECT id,name,email,created_at FROM users");
            shift->do("DROP TABLE users");
            shift->do("ALTER TABLE users_tmp RENAME TO users");
        },
    },
    {
        version => '004_create_comments',
        up => sub {
            shift->do("CREATE TABLE comments (
                id INTEGER PRIMARY KEY,
                post_id INTEGER,
                user_id INTEGER,
                body TEXT,
                created_at DATETIME DEFAULT (datetime('now'))
            )");
        },
        down => sub { shift->do("DROP TABLE comments") },
    },
);

my $m = Migration->new($dbh);

print "Current schema version: ";
my @applied = $m->applied;
print @applied ? $applied[-1] : "(none)";
print "\n\n";

print "Migrating:\n";
$m->migrate(@migrations);

print "\nApplied migrations:\n";
printf "  %s\n", $_ for $m->applied;

print "\nRolling back last migration:\n";
$m->rollback(1, @migrations);

print "\nMigrating again:\n";
$m->migrate(@migrations);

$dbh->disconnect;
```

---

## Step 186: Full-text Search

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

my $dbh = DBI->connect("dbi:SQLite:dbname=/tmp/fts_demo.db","","",
    { RaiseError => 1, AutoCommit => 1 });

# SQLite FTS5 virtual table
$dbh->do("DROP TABLE IF EXISTS docs_fts");
$dbh->do("DROP TABLE IF EXISTS docs");

$dbh->do(<<'SQL');
CREATE TABLE docs (
    id      INTEGER PRIMARY KEY,
    title   TEXT,
    content TEXT,
    author  TEXT,
    tags    TEXT
)
SQL

# FTS5 table
$dbh->do(<<'SQL');
CREATE VIRTUAL TABLE docs_fts USING fts5(
    title, content, author, tags,
    content='docs',
    content_rowid='id'
)
SQL

# Triggers to keep FTS in sync
$dbh->do(<<'SQL');
CREATE TRIGGER docs_ai AFTER INSERT ON docs BEGIN
    INSERT INTO docs_fts(rowid, title, content, author, tags)
    VALUES (new.id, new.title, new.content, new.author, new.tags);
END
SQL

$dbh->do(<<'SQL');
CREATE TRIGGER docs_ad AFTER DELETE ON docs BEGIN
    INSERT INTO docs_fts(docs_fts, rowid, title, content, author, tags)
    VALUES('delete', old.id, old.title, old.content, old.author, old.tags);
END
SQL

# Insert test data
my @docs = (
    { title => "Perl CGI Programming",
      content => "Learn how to build web applications with Perl CGI. CGI stands for Common Gateway Interface.",
      author => "Alice", tags => "perl cgi web" },
    { title => "Database Design",
      content => "Introduction to relational database design with SQL and normalization techniques.",
      author => "Bob", tags => "database sql" },
    { title => "Perl DBI Tutorial",
      content => "Using DBI to connect to databases from Perl. Covers SQLite, MySQL, PostgreSQL.",
      author => "Alice", tags => "perl database dbi" },
    { title => "Web Security Best Practices",
      content => "How to secure your web applications against XSS, SQL injection, and CSRF attacks.",
      author => "Carol", tags => "security web" },
    { title => "Regular Expressions in Perl",
      content => "Master regular expressions with Perl regex engine. Lookahead, lookbehind, captures.",
      author => "Dave", tags => "perl regex" },
    { title => "MySQL Performance Tuning",
      content => "Optimize your MySQL database queries, indexes, and configuration for better performance.",
      author => "Bob", tags => "database mysql performance" },
);

my $insert = $dbh->prepare("INSERT INTO docs (title,content,author,tags) VALUES (?,?,?,?)");
$insert->execute($_->{title}, $_->{content}, $_->{author}, $_->{tags}) for @docs;

print "Indexed ", scalar @docs, " documents\n\n";

# =====================
# FTS queries
# =====================

sub search {
    my ($query, $limit) = @_;
    $limit //= 10;
    
    return $dbh->selectall_arrayref(<<'SQL', { Slice => {} }, $query, $limit);
SELECT docs.id, docs.title, docs.author, docs.tags,
       snippet(docs_fts, 1, '<b>', '</b>', '...', 20) AS excerpt,
       bm25(docs_fts) AS rank
FROM docs_fts
JOIN docs ON docs.id = docs_fts.rowid
WHERE docs_fts MATCH ?
ORDER BY rank
LIMIT ?
SQL
}

sub search_by_author {
    my ($author, $query) = @_;
    return $dbh->selectall_arrayref(<<'SQL', { Slice => {} }, $query, $author);
SELECT docs.id, docs.title, docs.author, docs.tags
FROM docs_fts
JOIN docs ON docs.id = docs_fts.rowid
WHERE docs_fts MATCH ? AND docs.author = ?
ORDER BY bm25(docs_fts)
SQL
}

# Test searches
my @tests = ("perl", "database", "web", "perl AND database");

for my $q (@tests) {
    my $results = search($q);
    printf "Search: '%s' → %d results\n", $q, scalar @$results;
    for my $r (@$results) {
        printf "  [%d] %s by %s\n", $r->{id}, $r->{title}, $r->{author};
    }
    print "\n";
}

# Author-specific search
my $alice_perl = search_by_author("Alice", "perl");
print "Alice's perl documents:\n";
printf "  - %s\n", $_->{title} for @$alice_perl;

$dbh->disconnect;
```

---

## Step 187: Connection Pool

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

# =====================
# Database Connection Pool
# =====================

package DB::Pool;

my %pools;

sub get_pool {
    my ($class, $name) = @_;
    $name //= 'default';
    return $pools{$name};
}

sub new {
    my ($class, %opts) = @_;
    
    my $self = bless {
        dsn       => $opts{dsn},
        user      => $opts{user}     // '',
        pass      => $opts{pass}     // '',
        attrs     => $opts{attrs}    // { RaiseError => 1, AutoCommit => 1 },
        min_size  => $opts{min_size} // 2,
        max_size  => $opts{max_size} // 10,
        idle      => [],
        active    => 0,
        name      => $opts{name}     // 'default',
    }, $class;
    
    $pools{$self->{name}} = $self;
    
    # Pre-create minimum connections
    for (1..$self->{min_size}) {
        push @{$self->{idle}}, $self->_new_connection;
    }
    
    return $self;
}

sub _new_connection {
    my $self = shift;
    return DBI->connect($self->{dsn}, $self->{user}, $self->{pass}, $self->{attrs});
}

sub acquire {
    my ($self, $timeout) = @_;
    $timeout //= 30;
    
    # Reuse idle connection
    while (my $dbh = shift @{$self->{idle}}) {
        if ($dbh->ping) {
            $self->{active}++;
            return $dbh;
        }
        # Dead connection — discard
    }
    
    # Create new if under limit
    if ($self->{active} < $self->{max_size}) {
        my $dbh = $self->_new_connection;
        $self->{active}++;
        return $dbh;
    }
    
    die "Connection pool exhausted (max=$self->{max_size})\n";
}

sub release {
    my ($self, $dbh) = @_;
    
    if ($dbh->ping) {
        # Reset state
        $dbh->rollback unless $dbh->{AutoCommit};
        $dbh->{AutoCommit} = 1;
        push @{$self->{idle}}, $dbh;
    }
    
    $self->{active}--;
}

sub with_connection {
    my ($self, $code) = @_;
    my $dbh = $self->acquire;
    my $result;
    eval { $result = $code->($dbh) };
    my $err = $@;
    $self->release($dbh);
    die $err if $err;
    return $result;
}

sub stats {
    my $self = shift;
    return {
        pool     => $self->{name},
        idle     => scalar @{$self->{idle}},
        active   => $self->{active},
        total    => scalar @{$self->{idle}} + $self->{active},
        max_size => $self->{max_size},
    };
}

sub destroy_all {
    my $self = shift;
    for my $dbh (@{$self->{idle}}) {
        $dbh->disconnect;
    }
    $self->{idle} = [];
}

# =====================
# Demo
# =====================

package main;

my $pool = DB::Pool->new(
    name     => 'default',
    dsn      => 'dbi:SQLite:dbname=/tmp/pool_demo.db',
    min_size => 2,
    max_size => 5,
);

printf "Pool created: %s\n", $pool->{dsn};
my $stats = $pool->stats;
printf "Initial: idle=%d, active=%d\n", $stats->{idle}, $stats->{active};

# Use connection with auto-release
$pool->with_connection(sub {
    my $dbh = shift;
    $dbh->do("CREATE TABLE IF NOT EXISTS test (id INTEGER PRIMARY KEY, val TEXT)");
    $dbh->do("DELETE FROM test");
    $dbh->do("INSERT INTO test (val) VALUES (?)", undef, "hello");
    $dbh->do("INSERT INTO test (val) VALUES (?)", undef, "world");
    printf "Inserted rows inside with_connection\n";
});

# Manual acquire/release
my $dbh = $pool->acquire;
my $rows = $dbh->selectall_arrayref("SELECT * FROM test", { Slice => {} });
printf "Fetched: %s\n", join(", ", map { $_->{val} } @$rows);
$pool->release($dbh);

$stats = $pool->stats;
printf "\nFinal stats: idle=%d, active=%d, total=%d\n",
    $stats->{idle}, $stats->{active}, $stats->{total};

$pool->destroy_all;
print "Pool cleaned up.\n";
```

---

## Step 188: Reports และ Export

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;
use POSIX qw(strftime);

my $dbh = DBI->connect("dbi:SQLite:dbname=/tmp/report_demo.db","","",
    { RaiseError => 1, AutoCommit => 1 });

# Setup test data
$dbh->do("CREATE TABLE IF NOT EXISTS sales (
    id INTEGER PRIMARY KEY,
    product TEXT,
    amount REAL,
    qty INTEGER,
    region TEXT,
    sale_date DATE
)");

$dbh->do("DELETE FROM sales");

my @products = ("Widget A","Widget B","Gadget X","Gadget Y","Tool Z");
my @regions  = ("North","South","East","West");
my @insert   = map {
    my $prod   = $products[rand @products];
    my $region = $regions[rand @regions];
    my $qty    = 1 + int rand 20;
    my $price  = 10 + rand 90;
    my $days   = int rand 365;
    [$prod, $price * $qty, $qty, $region,
     strftime("%Y-%m-%d", localtime(time - $days * 86400))]
} 1..200;

my $ins = $dbh->prepare("INSERT INTO sales (product,amount,qty,region,sale_date) VALUES (?,?,?,?,?)");
$ins->execute(@$_) for @insert;

# =====================
# Report generators
# =====================

sub report_summary {
    print "\n=== SALES SUMMARY REPORT ===\n";
    my ($total, $count, $avg) = $dbh->selectrow_array(
        "SELECT SUM(amount), COUNT(*), AVG(amount) FROM sales"
    );
    printf "Total Sales: \$%,.0f\n", $total;
    printf "Transactions: %d\n",      $count;
    printf "Average:      \$%,.0f\n", $avg;
}

sub report_by_region {
    print "\n=== SALES BY REGION ===\n";
    my $data = $dbh->selectall_arrayref(<<'SQL', { Slice => {} });
SELECT region,
       COUNT(*) AS txns,
       SUM(amount) AS total,
       AVG(amount) AS avg,
       MAX(amount) AS max_sale
FROM sales
GROUP BY region
ORDER BY total DESC
SQL
    printf "%-8s %5s %12s %10s %10s\n", "Region","Txns","Total","Average","Max";
    print "-" x 50 . "\n";
    printf "%-8s %5d %12.0f %10.0f %10.0f\n",
        $_->{region}, $_->{txns}, $_->{total}, $_->{avg}, $_->{max_sale}
    for @$data;
}

sub report_top_products {
    print "\n=== TOP PRODUCTS ===\n";
    my $data = $dbh->selectall_arrayref(<<'SQL', { Slice => {} });
SELECT product,
       COUNT(*) AS sales_count,
       SUM(qty) AS total_qty,
       SUM(amount) AS revenue
FROM sales
GROUP BY product
ORDER BY revenue DESC
SQL
    printf "%-12s %6s %8s %12s\n", "Product","Sales","Units","Revenue";
    print "-" x 42 . "\n";
    printf "%-12s %6d %8d %12.0f\n",
        $_->{product}, $_->{sales_count}, $_->{total_qty}, $_->{revenue}
    for @$data;
}

# Export to CSV
sub export_csv {
    my ($outfile) = @_;
    
    open(my $fh, '>:utf8', $outfile) or die "Cannot write: $!";
    
    print $fh join(",", qw(ID Product Amount Qty Region Date)), "\n";
    
    my $rows = $dbh->selectall_arrayref(
        "SELECT id,product,amount,qty,region,sale_date FROM sales ORDER BY id",
        { Slice => {} }
    );
    
    for my $r (@$rows) {
        printf $fh "%d,\"%s\",%.2f,%d,\"%s\",\"%s\"\n",
            $r->{id}, $r->{product}, $r->{amount},
            $r->{qty}, $r->{region}, $r->{sale_date};
    }
    
    close $fh;
    printf "Exported %d rows to %s\n", scalar @$rows, $outfile;
}

# Export to HTML
sub export_html {
    my ($outfile) = @_;
    
    my $rows = $dbh->selectall_arrayref(
        "SELECT id,product,amount,qty,region,sale_date FROM sales ORDER BY sale_date DESC LIMIT 20",
        { Slice => {} }
    );
    
    open(my $fh, '>:utf8', $outfile) or die "Cannot write: $!";
    
    print $fh <<HTML;
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>Sales Report</title>
<style>
body { font-family: sans-serif; padding: 20px; }
table { border-collapse: collapse; width: 100%; }
th,td { border: 1px solid #ddd; padding: 8px; text-align: left; }
th { background: #2c5aa0; color: white; }
tr:nth-child(even) { background: #f2f2f2; }
</style>
</head>
<body>
<h1>Sales Report</h1>
<table>
<tr><th>ID</th><th>Product</th><th>Amount</th><th>Qty</th><th>Region</th><th>Date</th></tr>
HTML
    
    for my $r (@$rows) {
        printf $fh "<tr><td>%d</td><td>%s</td><td>\$%.0f</td><td>%d</td><td>%s</td><td>%s</td></tr>\n",
            $r->{id}, $r->{product}, $r->{amount},
            $r->{qty}, $r->{region}, $r->{sale_date};
    }
    
    print $fh "</table></body></html>\n";
    close $fh;
    
    printf "Exported HTML report: %s\n", $outfile;
}

report_summary();
report_by_region();
report_top_products();
export_csv("/tmp/sales_report.csv");
export_html("/tmp/sales_report.html");

$dbh->disconnect;
```

---

## Step 189: Database Fixtures และ Testing

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

# =====================
# Test::More pattern
# =====================

my ($pass, $fail) = (0, 0);

sub ok {
    my ($cond, $name) = @_;
    if ($cond) {
        printf "ok - %s\n", $name // "test";
        $pass++;
    } else {
        printf "not ok - %s\n", $name // "test";
        $fail++;
    }
}

sub is {
    my ($got, $expected, $name) = @_;
    ok($got eq $expected, "$name (got: '$got', expected: '$expected')");
}

sub is_num {
    my ($got, $expected, $name) = @_;
    ok($got == $expected, "$name (got: $got, expected: $expected)");
}

sub done_testing {
    printf "\n%d passed, %d failed.\n", $pass, $fail;
}

# =====================
# Test Database helper
# =====================

package TestDB;

sub new {
    my $class = shift;
    my $dbh = DBI->connect("dbi:SQLite:dbname=:memory:","","",
        { RaiseError => 1, AutoCommit => 1 });
    return bless { dbh => $dbh }, $class;
}

sub dbh { $_[0]->{dbh} }

sub setup_schema {
    my ($self, @sql) = @_;
    $self->{dbh}->do($_) for @sql;
}

sub load_fixture {
    my ($self, $table, @rows) = @_;
    return unless @rows;
    my @fields = keys %{$rows[0]};
    my $sql = "INSERT INTO $table (" . join(",", @fields) . ") VALUES (" .
              join(",", ("?") x @fields) . ")";
    my $sth = $self->{dbh}->prepare($sql);
    $sth->execute(@$_{@fields}) for @rows;
}

sub teardown { $_[0]->{dbh}->disconnect }

# =====================
# Model to test
# =====================

package Model::Product;

sub new {
    my ($class, $dbh) = @_;
    return bless { dbh => $dbh }, $class;
}

sub find    { $_[0]->{dbh}->selectrow_hashref("SELECT * FROM products WHERE id=?", undef, $_[1]) }
sub all     { @{$_[0]->{dbh}->selectall_arrayref("SELECT * FROM products ORDER BY id", { Slice => {} })} }
sub count   { ($_[0]->{dbh}->selectrow_array("SELECT COUNT(*) FROM products"))[0] }
sub by_category {
    @{$_[0]->{dbh}->selectall_arrayref("SELECT * FROM products WHERE category=?", { Slice => {} }, $_[1])}
}
sub create  {
    my ($self, %d) = @_;
    $self->{dbh}->do("INSERT INTO products (name,price,category) VALUES (?,?,?)",
        undef, $d{name}, $d{price}, $d{category});
    return $self->find($self->{dbh}->last_insert_id);
}
sub update  {
    my ($self, $id, %d) = @_;
    $self->{dbh}->do("UPDATE products SET name=?, price=? WHERE id=?", undef, $d{name}, $d{price}, $id);
    return $self->find($id);
}
sub delete  { $_[0]->{dbh}->do("DELETE FROM products WHERE id=?", undef, $_[1]) }

# =====================
# Tests
# =====================

package main;

my $tdb = TestDB->new;

$tdb->setup_schema(
    "CREATE TABLE products (id INTEGER PRIMARY KEY, name TEXT, price REAL, category TEXT)"
);

$tdb->load_fixture('products',
    { name => "Widget A", price => 9.99,  category => "widget" },
    { name => "Widget B", price => 14.99, category => "widget" },
    { name => "Gadget X", price => 24.99, category => "gadget" },
);

my $m = Model::Product->new($tdb->dbh);

print "=== Product Model Tests ===\n\n";

is_num($m->count, 3, "count returns 3");

my $p = $m->find(1);
ok(defined $p, "find(1) returns product");
is($p->{name}, "Widget A", "product name correct");
is_num($p->{price}, 9.99, "product price correct");

my @widgets = $m->by_category("widget");
is_num(scalar @widgets, 2, "two widgets in category");

my $new = $m->create(name => "New Gadget", price => 39.99, category => "gadget");
ok(defined $new->{id}, "create returns new product with id");
is_num($m->count, 4, "count is 4 after create");

my $updated = $m->update($new->{id}, name => "Better Gadget", price => 44.99);
is($updated->{name}, "Better Gadget", "update changes name");
is_num($updated->{price}, 44.99, "update changes price");

$m->delete($new->{id});
is_num($m->count, 3, "count back to 3 after delete");

ok(!$m->find($new->{id}), "deleted product not found");

$tdb->teardown;
done_testing();
```

---

## Step 190: โปรแกรมสรุป — Complete Database App

```perl
#!/usr/bin/perl
# db_app.pl — Complete Inventory Management System
use strict;
use warnings;
use DBI;
use POSIX qw(strftime);

my $dbh = DBI->connect("dbi:SQLite:dbname=/tmp/inventory.db","","",
    { RaiseError => 1, AutoCommit => 1, sqlite_unicode => 1 });

# Schema
$dbh->do("CREATE TABLE IF NOT EXISTS products (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    sku TEXT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    category TEXT,
    price REAL NOT NULL,
    cost REAL DEFAULT 0,
    stock INTEGER DEFAULT 0,
    min_stock INTEGER DEFAULT 5,
    created_at DATETIME DEFAULT (datetime('now'))
)");

$dbh->do("CREATE TABLE IF NOT EXISTS stock_movements (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    product_id INTEGER,
    type TEXT,   -- 'in','out','adjust'
    quantity INTEGER,
    note TEXT,
    created_at DATETIME DEFAULT (datetime('now'))
)");

# Seed
unless (($dbh->selectrow_array("SELECT COUNT(*) FROM products"))[0]) {
    my $ins = $dbh->prepare("INSERT INTO products (sku,name,category,price,cost,stock,min_stock) VALUES (?,?,?,?,?,?,?)");
    my @items = (
        ["SKU001","Laptop","Electronics", 999.99, 650.00, 15, 5],
        ["SKU002","Mouse", "Electronics", 29.99,  15.00, 50, 10],
        ["SKU003","Desk Chair","Furniture",199.99,100.00, 8, 3],
        ["SKU004","Monitor","Electronics",349.99, 200.00, 12, 5],
        ["SKU005","Keyboard","Electronics", 59.99, 30.00, 30, 10],
        ["SKU006","Webcam","Electronics",  89.99, 45.00,  3, 5],  # below min!
    );
    $ins->execute(@$_) for @items;
}

# =====================
# Inventory operations
# =====================

sub receive_stock {
    my ($sku, $qty, $note) = @_;
    my $p = $dbh->selectrow_hashref("SELECT * FROM products WHERE sku=?", undef, $sku);
    die "Product not found: $sku\n" unless $p;
    
    $dbh->{AutoCommit} = 0;
    eval {
        $dbh->do("UPDATE products SET stock = stock + ? WHERE id=?", undef, $qty, $p->{id});
        $dbh->do("INSERT INTO stock_movements (product_id,type,quantity,note) VALUES (?,?,?,?)",
            undef, $p->{id}, 'in', $qty, $note//"Received");
        $dbh->commit;
    };
    $dbh->rollback if $@;
    $dbh->{AutoCommit} = 1;
    die $@ if $@;
    
    printf "Received %d of %s\n", $qty, $sku;
}

sub sell_item {
    my ($sku, $qty, $note) = @_;
    my $p = $dbh->selectrow_hashref("SELECT * FROM products WHERE sku=?", undef, $sku);
    die "Product not found: $sku\n" unless $p;
    die "Insufficient stock (have $p->{stock}, need $qty)\n" if $p->{stock} < $qty;
    
    $dbh->{AutoCommit} = 0;
    eval {
        $dbh->do("UPDATE products SET stock = stock - ? WHERE id=?", undef, $qty, $p->{id});
        $dbh->do("INSERT INTO stock_movements (product_id,type,quantity,note) VALUES (?,?,?,?)",
            undef, $p->{id}, 'out', $qty, $note//"Sold");
        $dbh->commit;
    };
    $dbh->rollback if $@;
    $dbh->{AutoCommit} = 1;
    die $@ if $@;
    
    printf "Sold %d of %s\n", $qty, $sku;
}

# =====================
# Reports
# =====================

sub print_inventory {
    print "\n=== INVENTORY ===\n";
    printf "%-8s %-18s %-12s %8s %8s %6s %6s %s\n",
        "SKU","Name","Category","Price","Cost","Stock","Min","Status";
    print "-" x 80 . "\n";
    
    my $items = $dbh->selectall_arrayref("SELECT * FROM products ORDER BY category,name", { Slice => {} });
    for my $p (@$items) {
        my $status = $p->{stock} <= $p->{min_stock} ? "⚠ LOW" : "OK";
        my $margin = $p->{price} - $p->{cost};
        printf "%-8s %-18s %-12s %8.2f %8.2f %6d %6d %s\n",
            $p->{sku}, $p->{name}, $p->{category},
            $p->{price}, $p->{cost}, $p->{stock}, $p->{min_stock}, $status;
    }
    
    my ($total_val) = $dbh->selectrow_array("SELECT SUM(price * stock) FROM products");
    printf "\nTotal inventory value: \$%,.0f\n", $total_val // 0;
}

sub print_low_stock {
    print "\n=== LOW STOCK ALERT ===\n";
    my $low = $dbh->selectall_arrayref(
        "SELECT * FROM products WHERE stock <= min_stock ORDER BY stock",
        { Slice => {} }
    );
    if (@$low) {
        for my $p (@$low) {
            printf "  ⚠ %s %s: stock=%d (min=%d)\n",
                $p->{sku}, $p->{name}, $p->{stock}, $p->{min_stock};
        }
    } else {
        print "  All items adequately stocked.\n";
    }
}

sub print_movement_log {
    my $limit = shift // 10;
    print "\n=== STOCK MOVEMENT LOG ===\n";
    my $log = $dbh->selectall_arrayref(<<'SQL', { Slice => {} }, $limit);
SELECT m.*, p.sku, p.name
FROM stock_movements m
JOIN products p ON m.product_id = p.id
ORDER BY m.id DESC
LIMIT ?
SQL
    for my $e (@$log) {
        my $arrow = $e->{type} eq 'in' ? '↑' : '↓';
        printf "  %s [%s] %s %-8s %-15s qty=%d %s\n",
            $arrow, $e->{created_at}, $e->{type},
            $e->{sku}, $e->{name}, $e->{quantity}, $e->{note}//"";
    }
}

# Demo run
print_inventory();
print_low_stock();

print "\nReceiving stock:\n";
receive_stock("SKU006", 20, "Restock order #1234");
receive_stock("SKU001", 5,  "Restock order #1235");

print "\nSelling items:\n";
eval { sell_item("SKU002", 10, "Order #5001") };
print "Error: $@" if $@;

eval { sell_item("SKU001", 100, "This should fail") };
print "Expected error: $@" if $@;

print_movement_log(5);
print_inventory();
print_low_stock();

$dbh->disconnect;
print "\nDone.\n";
```

---

## สรุป Part 19

ใน Part นี้คุณได้เรียนรู้:
- ✅ DBI พื้นฐาน — connect, prepare, execute
- ✅ Transactions — begin/commit/rollback
- ✅ ORM Pattern
- ✅ Query Builder
- ✅ Database Migrations
- ✅ Full-text Search (SQLite FTS5)
- ✅ Connection Pool
- ✅ Reports และ Export (CSV, HTML)
- ✅ Test Fixtures
- ✅ Inventory Management App

**ถัดไป: [Part 20 — Beginner Capstone Project](part_20.md)**
