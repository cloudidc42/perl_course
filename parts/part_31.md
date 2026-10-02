# Part 31: Advanced Database Patterns with DBI
## Steps 301-310: ฐานข้อมูลขั้นสูง — ORM, Transactions, Query Builder

---

## Step 301: DBI Review and Connection Pool

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

# Connection wrapper with pooling simulation
{
package DB::Pool;

my @pool;
my $dsn      = "dbi:SQLite::memory:";
my $max_size = 3;

sub get {
    my $class = shift;
    # Return existing connection if available
    for my $conn (@pool) {
        return $conn if $conn->ping;
    }
    # Create new connection
    die "Pool exhausted (max $max_size)\n" if @pool >= $max_size;
    my $dbh = DBI->connect($dsn, "", "", {
        RaiseError     => 1,
        AutoCommit     => 1,
        sqlite_unicode => 1,
        PrintError     => 0,
    });
    push @pool, $dbh;
    return $dbh;
}

sub size    { scalar @pool }
sub release { }   # In real pool: mark connection as available
sub stats   { { size => scalar @pool, max => $max_size } }
}

# Test pool
my $dbh = DB::Pool->get;
printf "Pool size: %d\n", DB::Pool->size;
printf "DB handle: %s\n", ref($dbh);
printf "Ping: %s\n", $dbh->ping ? "OK" : "FAIL";

# Create a table and insert data
$dbh->do(q{
    CREATE TABLE IF NOT EXISTS products (
        id          INTEGER PRIMARY KEY AUTOINCREMENT,
        name        TEXT NOT NULL,
        category    TEXT,
        price       REAL,
        stock       INTEGER DEFAULT 0,
        created_at  INTEGER DEFAULT (strftime('%s','now'))
    )
});

# Prepared statement
my $sth = $dbh->prepare("INSERT INTO products (name, category, price, stock) VALUES (?,?,?,?)");

my @products = (
    ["Widget A", "hardware", 9.99,  100],
    ["Widget B", "hardware", 14.99, 50],
    ["Gadget X", "electronics", 49.99, 25],
    ["Gadget Y", "electronics", 89.99, 10],
    ["Book 1",   "books",      12.99, 200],
    ["Book 2",   "books",      8.99,  150],
    ["Tool Z",   "tools",      34.99, 30],
);

$sth->execute(@$_) for @products;
printf "Inserted %d products\n", scalar @products;

# Various query styles
my $count = $dbh->selectrow_array("SELECT COUNT(*) FROM products");
printf "Total: %d\n", $count;

my @rows = @{$dbh->selectall_arrayref("SELECT * FROM products ORDER BY price", {Slice=>{}})};
printf "Cheapest: %s (\$%.2f)\n", $rows[0]{name}, $rows[0]{price};
printf "Most expensive: %s (\$%.2f)\n", $rows[-1]{name}, $rows[-1]{price};

# Group by
my $cats = $dbh->selectall_arrayref(
    "SELECT category, COUNT(*) AS n, AVG(price) AS avg_price FROM products GROUP BY category ORDER BY category",
    {Slice=>{}}
);
printf "\nCategories:\n";
printf "  %-12s %2d items  avg \$%.2f\n", $_->{category}, $_->{n}, $_->{avg_price} for @$cats;
```

---

## Step 302: Query Builder

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package DB::Query;

sub new {
    my ($class, $table) = @_;
    return bless {
        table      => $table,
        conditions => [],
        params     => [],
        columns    => ["*"],
        order      => [],
        limit_n    => undef,
        offset_n   => 0,
        joins      => [],
    }, $class;
}

sub select {
    my ($self, @cols) = @_;
    $self->{columns} = \@cols;
    return $self;
}

sub where {
    my ($self, $col, $op, $val) = @_;
    if (!defined $val) { $val = $op; $op = "=" }
    push @{$self->{conditions}}, "$col $op ?";
    push @{$self->{params}},     $val;
    return $self;
}

sub where_in {
    my ($self, $col, @vals) = @_;
    my $placeholders = join(",", ("?") x @vals);
    push @{$self->{conditions}}, "$col IN ($placeholders)";
    push @{$self->{params}},     @vals;
    return $self;
}

sub where_like {
    my ($self, $col, $pattern) = @_;
    push @{$self->{conditions}}, "$col LIKE ?";
    push @{$self->{params}},     "%$pattern%";
    return $self;
}

sub order_by {
    my ($self, $col, $dir) = @_;
    push @{$self->{order}}, "$col " . (uc($dir//"ASC"));
    return $self;
}

sub limit  { $_[0]->{limit_n}  = $_[1]; return $_[0] }
sub offset { $_[0]->{offset_n} = $_[1]; return $_[0] }

sub join_table {
    my ($self, $table, $on) = @_;
    push @{$self->{joins}}, "JOIN $table ON $on";
    return $self;
}

sub to_sql {
    my $self = shift;
    my $cols  = join(", ", @{$self->{columns}});
    my $sql   = "SELECT $cols FROM $self->{table}";
    $sql .= " " . join(" ", @{$self->{joins}}) if @{$self->{joins}};
    $sql .= " WHERE " . join(" AND ", @{$self->{conditions}}) if @{$self->{conditions}};
    $sql .= " ORDER BY " . join(", ", @{$self->{order}}) if @{$self->{order}};
    $sql .= " LIMIT $self->{limit_n}" if defined $self->{limit_n};
    $sql .= " OFFSET $self->{offset_n}" if $self->{offset_n};
    return ($sql, @{$self->{params}});
}

sub execute {
    my ($self, $dbh) = @_;
    my ($sql, @params) = $self->to_sql;
    return $dbh->selectall_arrayref($sql, {Slice=>{}}, @params);
}

sub count {
    my ($self, $dbh) = @_;
    my $copy = bless {%$self}, ref($self);
    $copy->{columns}  = ["COUNT(*) AS n"];
    $copy->{order}    = [];
    $copy->{limit_n}  = undef;
    $copy->{offset_n} = 0;
    my ($sql, @params) = $copy->to_sql;
    return $dbh->selectrow_array($sql, undef, @params);
}
}

package main;

# Use Query Builder
my $dbh = DB::Pool->get;

# Simple query
my $q1 = DB::Query->new("products")
    ->select("id", "name", "price")
    ->where("category", "electronics")
    ->order_by("price", "desc");

my ($sql, @params) = $q1->to_sql;
printf "SQL: %s\n", $sql;
printf "Params: %s\n", join(", ", @params);

my $results = $q1->execute($dbh);
printf "Electronics:\n";
printf "  %s: \$%.2f\n", $_->{name}, $_->{price} for @$results;

# Complex query
my $q2 = DB::Query->new("products")
    ->where("price", ">", 10)
    ->where_like("name", "g")
    ->order_by("price")
    ->limit(5);

my $count = $q2->count($dbh);
my $items = $q2->execute($dbh);
printf "\nFiltered (price>10, name~g): count=%d, fetched=%d\n", $count, scalar @$items;
printf "  %s: \$%.2f\n", $_->{name}, $_->{price} for @$items;

# IN query
my $q3 = DB::Query->new("products")
    ->where_in("category", "books", "tools")
    ->order_by("category")
    ->order_by("price");

my ($sql3, @p3) = $q3->to_sql;
printf "\nIN query SQL: %s\n", $sql3;
my $items3 = $q3->execute($dbh);
printf "  %-10s %-15s \$%.2f\n", $_->{category}, $_->{name}, $_->{price} for @$items3;
```

---

## Step 303: Active Record Pattern

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package ActiveRecord::Base;

my $dbh;
sub _dbh       { $dbh }
sub set_db     { $dbh = $_[1] }
sub table_name { die ref(shift) . " must define table_name\n" }
sub primary_key { "id" }

sub new {
    my ($class, %attrs) = @_;
    my $self = bless {}, $class;
    $self->{$_} = $attrs{$_} for keys %attrs;
    $self->{__new__} = 1;
    return $self;
}

sub find {
    my ($class, $id) = @_;
    my $row = $dbh->selectrow_hashref(
        "SELECT * FROM " . $class->table_name . " WHERE " . $class->primary_key . "=?",
        undef, $id);
    return unless $row;
    my $obj = bless $row, $class;
    $obj->{__new__} = 0;
    return $obj;
}

sub all {
    my $class = shift;
    my @rows = @{$dbh->selectall_arrayref("SELECT * FROM " . $class->table_name, {Slice=>{}})};
    return map { my $o = bless $_, $class; $o->{__new__} = 0; $o } @rows;
}

sub where {
    my ($class, %conditions) = @_;
    my @conds  = map { "$_ = ?" } keys %conditions;
    my @params = values %conditions;
    my @rows   = @{$dbh->selectall_arrayref(
        "SELECT * FROM " . $class->table_name . " WHERE " . join(" AND ", @conds),
        {Slice=>{}}, @params
    )};
    return map { my $o = bless $_, $class; $o->{__new__} = 0; $o } @rows;
}

sub save {
    my $self  = shift;
    my $class = ref $self;
    my $pk    = $class->primary_key;
    my $table = $class->table_name;
    
    # Remove meta keys
    my %data = map { $_ => $self->{$_} } grep { $_ !~ /^__/ } keys %$self;
    delete $data{$pk};
    
    if ($self->{__new__}) {
        my @cols = keys %data;
        my $sql  = "INSERT INTO $table (" . join(",", @cols) . ") VALUES (" . join(",", ("?") x @cols) . ")";
        $dbh->do($sql, undef, @data{@cols});
        $self->{$pk}    = $dbh->last_insert_id;
        $self->{__new__} = 0;
    } else {
        my @cols = keys %data;
        my $set  = join(", ", map { "$_=?" } @cols);
        $dbh->do("UPDATE $table SET $set WHERE $pk=?", undef, @data{@cols}, $self->{$pk});
    }
    return $self;
}

sub delete {
    my $self = shift;
    $dbh->do("DELETE FROM " . ref($self)->table_name . " WHERE " . ref($self)->primary_key . "=?",
        undef, $self->{ref($self)->primary_key});
}

sub AUTOLOAD {
    my $self = shift;
    our $AUTOLOAD;
    my $name = $AUTOLOAD;
    $name =~ s/.*:://;
    return if $name eq 'DESTROY';
    
    if (@_) {
        $self->{$name} = shift;
        return $self;
    }
    return $self->{$name};
}
}

{
package Product;
use parent -norequire, 'ActiveRecord::Base';

sub table_name { "products" }

sub is_in_stock { $_[0]->{stock} > 0 }
sub apply_discount {
    my ($self, $pct) = @_;
    $self->{price} = sprintf "%.2f", $self->{price} * (1 - $pct/100);
    return $self;
}
}

package main;

ActiveRecord::Base->set_db(DB::Pool->get);

# Find
my $p = Product->find(1);
printf "Found: %s (\$%.2f)\n", $p->name, $p->price;

# All
my @all = Product->all;
printf "Total: %d\n", scalar @all;

# Where
my @elec = Product->where(category => "electronics");
printf "Electronics: %d\n", scalar @elec;
printf "  %s\n", $_->name for @elec;

# Create new
my $new = Product->new(
    name     => "New Widget",
    category => "hardware",
    price    => 19.99,
    stock    => 75,
)->save;
printf "\nCreated: %s (id=%d)\n", $new->name, $new->id;

# Update
$new->price(24.99)->save;
my $reloaded = Product->find($new->id);
printf "Updated price: \$%.2f\n", $reloaded->price;

# Delete
$new->delete;
printf "Deleted: %s\n", Product->find($new->id) ? "still_exists" : "gone";
```

---

## Step 304: Transactions

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Transfer funds demo (using products stock as example)
{
package DB::Transaction;

sub run {
    my ($class, $dbh, $code) = @_;
    $dbh->{AutoCommit} = 0;
    my $result = eval { $code->($dbh) };
    if ($@) {
        $dbh->rollback;
        $dbh->{AutoCommit} = 1;
        die "Transaction failed: $@";
    }
    $dbh->commit;
    $dbh->{AutoCommit} = 1;
    return $result;
}

sub savepoint {
    my ($class, $dbh, $name, $code) = @_;
    $dbh->do("SAVEPOINT $name");
    eval { $code->($dbh) };
    if ($@) {
        $dbh->do("ROLLBACK TO SAVEPOINT $name");
        die $@;
    }
    $dbh->do("RELEASE SAVEPOINT $name");
}
}

package main;

my $dbh = DB::Pool->get;

# Add an inventory table
$dbh->do(q{
    CREATE TABLE IF NOT EXISTS inventory (
        id         INTEGER PRIMARY KEY AUTOINCREMENT,
        product_id INTEGER,
        quantity   INTEGER DEFAULT 0,
        reserved   INTEGER DEFAULT 0
    )
});

$dbh->do(q{
    CREATE TABLE IF NOT EXISTS orders (
        id         INTEGER PRIMARY KEY AUTOINCREMENT,
        product_id INTEGER,
        quantity   INTEGER,
        status     TEXT DEFAULT 'pending',
        created_at INTEGER DEFAULT (strftime('%s','now'))
    )
});

# Initialize inventory
for my $pid (1..3) {
    $dbh->do("INSERT OR IGNORE INTO inventory (product_id, quantity) VALUES (?,?)",
        undef, $pid, 100);
}

# Simulate order placement — must be atomic
sub place_order {
    my ($product_id, $qty) = @_;
    
    DB::Transaction->run($dbh, sub {
        my $dbh = shift;
        
        # Check inventory
        my ($avail) = $dbh->selectrow_array(
            "SELECT quantity - reserved FROM inventory WHERE product_id=?",
            undef, $product_id);
        
        die "Insufficient stock (have $avail, need $qty)\n" if ($avail//0) < $qty;
        
        # Reserve stock
        $dbh->do("UPDATE inventory SET reserved = reserved + ? WHERE product_id=?",
            undef, $qty, $product_id);
        
        # Create order
        $dbh->do("INSERT INTO orders (product_id, quantity, status) VALUES (?,?,?)",
            undef, $product_id, $qty, "pending");
        
        return $dbh->last_insert_id;
    });
}

# Success
my $order_id = eval { place_order(1, 10) };
printf "Order placed: id=%s err=%s\n", $order_id//"none", $@//"";

# Fail (too many)
my $fail_id = eval { place_order(1, 500) };
printf "Overflow order: id=%s err=%s", $fail_id//"none", $@//"none\n";

# Verify inventory
my $inv = $dbh->selectall_arrayref("SELECT * FROM inventory", {Slice=>{}});
printf "\nInventory:\n";
printf "  product=%d qty=%d reserved=%d avail=%d\n",
    $_->{product_id}, $_->{quantity}, $_->{reserved}, $_->{quantity}-$_->{reserved}
    for @$inv;

# Orders
my $orders = $dbh->selectall_arrayref("SELECT * FROM orders", {Slice=>{}});
printf "\nOrders: %d\n", scalar @$orders;
```

---

## Step 305: Data Migrations

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package DB::Migrator;

my @migrations;

sub add_migration {
    my ($class, $version, $name, $up_sql, $down_sql) = @_;
    push @migrations, {
        version  => $version,
        name     => $name,
        up       => $up_sql,
        down     => $down_sql // "",
    };
}

sub ensure_migrations_table {
    my ($class, $dbh) = @_;
    $dbh->do(q{
        CREATE TABLE IF NOT EXISTS schema_migrations (
            version    INTEGER PRIMARY KEY,
            name       TEXT,
            applied_at INTEGER DEFAULT (strftime('%s','now'))
        )
    });
}

sub current_version {
    my ($class, $dbh) = @_;
    return $dbh->selectrow_array(
        "SELECT COALESCE(MAX(version), 0) FROM schema_migrations") // 0;
}

sub migrate_up {
    my ($class, $dbh, $target) = @_;
    $class->ensure_migrations_table($dbh);
    my $current = $class->current_version($dbh);
    
    my @pending = sort { $a->{version} <=> $b->{version} }
                  grep { $_->{version} > $current &&
                         (!defined $target || $_->{version} <= $target) }
                  @migrations;
    
    printf "Current version: %d, pending: %d\n", $current, scalar @pending;
    
    for my $m (@pending) {
        printf "  Applying migration %d: %s\n", $m->{version}, $m->{name};
        
        $dbh->{AutoCommit} = 0;
        eval {
            for my $stmt (split /;\s*/, $m->{up}) {
                $stmt =~ s/^\s+|\s+$//g;
                $dbh->do($stmt) if length $stmt;
            }
            $dbh->do("INSERT INTO schema_migrations (version, name) VALUES (?,?)",
                undef, $m->{version}, $m->{name});
        };
        if ($@) {
            $dbh->rollback;
            $dbh->{AutoCommit} = 1;
            die "Migration $m->{version} failed: $@";
        }
        $dbh->commit;
        $dbh->{AutoCommit} = 1;
    }
    
    return $class->current_version($dbh);
}

sub status {
    my ($class, $dbh) = @_;
    $class->ensure_migrations_table($dbh);
    my $current  = $class->current_version($dbh);
    printf "Schema version: %d\n", $current;
    printf "Migrations registered: %d\n", scalar @migrations;
}
}

# Register migrations
DB::Migrator->add_migration(1, "create_users_table", q{
    CREATE TABLE IF NOT EXISTS app_users (
        id         INTEGER PRIMARY KEY AUTOINCREMENT,
        username   TEXT UNIQUE NOT NULL,
        created_at INTEGER DEFAULT (strftime('%s','now'))
    )
});

DB::Migrator->add_migration(2, "add_email_to_users", q{
    ALTER TABLE app_users ADD COLUMN email TEXT
});

DB::Migrator->add_migration(3, "create_profiles_table", q{
    CREATE TABLE IF NOT EXISTS profiles (
        id      INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER REFERENCES app_users(id),
        bio     TEXT,
        avatar  TEXT
    )
});

package main;

my $dbh = DB::Pool->get;

printf "Before migration:\n";
DB::Migrator->status($dbh);

printf "\nRunning migrations...\n";
my $v = DB::Migrator->migrate_up($dbh);

printf "\nAfter migration:\n";
DB::Migrator->status($dbh);
printf "Final version: %d\n", $v;

# Verify tables created
for my $table (qw(app_users profiles schema_migrations)) {
    my $exists = eval { $dbh->do("SELECT 1 FROM $table LIMIT 1"); 1 } // 0;
    printf "Table '%s': %s\n", $table, $exists ? "exists" : "missing";
}
```

---

## Step 306: Repository Pattern

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Abstract repository
{
package Repository::Base;

sub new {
    my ($class, $dbh) = @_;
    return bless { dbh => $dbh }, $class;
}

sub dbh        { $_[0]->{dbh} }
sub table_name { die ref(shift) . " must implement table_name" }

sub find_by_id {
    my ($self, $id) = @_;
    return $self->dbh->selectrow_hashref(
        "SELECT * FROM " . $self->table_name . " WHERE id=?", undef, $id);
}

sub find_all {
    my ($self, %opts) = @_;
    my $order = $opts{order_by} ? "ORDER BY $opts{order_by}" : "";
    my $limit  = defined $opts{limit} ? "LIMIT $opts{limit}" : "";
    return @{$self->dbh->selectall_arrayref(
        "SELECT * FROM " . $self->table_name . " $order $limit", {Slice=>{}})};
}

sub count {
    return $_[0]->dbh->selectrow_array("SELECT COUNT(*) FROM " . $_[0]->table_name);
}

sub create {
    my ($self, %data) = @_;
    my @cols  = keys %data;
    my $sql   = "INSERT INTO " . $self->table_name .
                " (" . join(",", @cols) . ") VALUES (" . join(",", ("?") x @cols) . ")";
    $self->dbh->do($sql, undef, @data{@cols});
    return $self->find_by_id($self->dbh->last_insert_id);
}

sub update {
    my ($self, $id, %data) = @_;
    my @cols = keys %data;
    my $set  = join(", ", map { "$_=?" } @cols);
    $self->dbh->do("UPDATE " . $self->table_name . " SET $set WHERE id=?", undef, @data{@cols}, $id);
    return $self->find_by_id($id);
}

sub delete_by_id {
    my ($self, $id) = @_;
    $self->dbh->do("DELETE FROM " . $self->table_name . " WHERE id=?", undef, $id);
}
}

# Product repository with domain-specific finders
{
package ProductRepository;
use parent -norequire, 'Repository::Base';

sub table_name { "products" }

sub find_by_category {
    my ($self, $cat) = @_;
    return @{$self->dbh->selectall_arrayref(
        "SELECT * FROM products WHERE category=? ORDER BY name",
        {Slice=>{}}, $cat
    )};
}

sub find_in_stock {
    return @{$_[0]->dbh->selectall_arrayref(
        "SELECT * FROM products WHERE stock > 0 ORDER BY stock DESC", {Slice=>{}})};
}

sub price_range {
    my ($self, $min, $max) = @_;
    return @{$self->dbh->selectall_arrayref(
        "SELECT * FROM products WHERE price BETWEEN ? AND ? ORDER BY price",
        {Slice=>{}}, $min, $max)};
}

sub adjust_stock {
    my ($self, $id, $delta) = @_;
    $self->dbh->do("UPDATE products SET stock = stock + ? WHERE id=?", undef, $delta, $id);
    return $self->find_by_id($id);
}

sub total_inventory_value {
    return $_[0]->dbh->selectrow_array("SELECT SUM(price * stock) FROM products") // 0;
}
}

package main;

my $repo = ProductRepository->new(DB::Pool->get);

printf "Product repo:\n";
printf "  Total products: %d\n", $repo->count;
printf "  In stock: %d\n", scalar $repo->find_in_stock;
printf "  Inventory value: \$%.2f\n", $repo->total_inventory_value;

printf "\nElectronics:\n";
printf "  %s: \$%.2f (stock: %d)\n", $_->{name}, $_->{price}, $_->{stock}
    for $repo->find_by_category("electronics");

printf "\nPrice range \$10-\$50:\n";
printf "  %s: \$%.2f\n", $_->{name}, $_->{price}
    for $repo->price_range(10, 50);

# CRUD via repo
my $new_product = $repo->create(
    name     => "Repo Widget",
    category => "hardware",
    price    => 7.99,
    stock    => 50,
);
printf "\nCreated: %s (id=%d)\n", $new_product->{name}, $new_product->{id};

my $updated = $repo->update($new_product->{id}, price => 8.99);
printf "Updated price: \$%.2f\n", $updated->{price};

my $stocked = $repo->adjust_stock($new_product->{id}, 10);
printf "After restock: %d units\n", $stocked->{stock};

$repo->delete_by_id($new_product->{id});
printf "Deleted: %s\n", $repo->find_by_id($new_product->{id}) ? "still_exists" : "gone";
```

---

## Step 307: N+1 Prevention with Eager Loading

```perl
#!/usr/bin/perl
use strict;
use warnings;

# Setup: orders referencing products and users
{
package DB::EagerLoader;

sub _dbh { DB::Pool->get }

# The N+1 problem — bad approach
sub load_orders_naive {
    my $class  = shift;
    my $orders = DB::Pool->get->selectall_arrayref("SELECT * FROM orders", {Slice=>{}});
    
    my $queries = 1;
    for my $order (@$orders) {
        # This causes N additional queries!
        $order->{product} = DB::Pool->get->selectrow_hashref(
            "SELECT * FROM products WHERE id=?", undef, $order->{product_id});
        $queries++;
    }
    
    return ($orders, $queries);
}

# Eager loading — good approach
sub load_orders_eager {
    my $class  = shift;
    my $orders = DB::Pool->get->selectall_arrayref("SELECT * FROM orders", {Slice=>{}});
    
    return ([], 1) unless @$orders;
    
    # Collect unique product IDs
    my %product_ids = map { $_->{product_id} => 1 } @$orders;
    my @ids         = keys %product_ids;
    
    # One query for all products
    my $placeholders = join(",", ("?") x @ids);
    my $products_list = DB::Pool->get->selectall_arrayref(
        "SELECT * FROM products WHERE id IN ($placeholders)",
        {Slice=>{}}, @ids
    );
    
    # Index by ID
    my %products = map { $_->{id} => $_ } @$products_list;
    
    # Attach to orders
    for my $order (@$orders) {
        $order->{product} = $products{ $order->{product_id} };
    }
    
    return ($orders, 2);  # Always 2 queries regardless of order count
}
}

package main;

# Add more orders for demo
my $dbh = DB::Pool->get;
for my $pid (1..3) {
    $dbh->do("INSERT INTO orders (product_id, quantity, status) VALUES (?,?,?)",
        undef, $pid, int(rand(5)+1), "pending");
}

my ($orders1, $q1) = DB::EagerLoader->load_orders_naive;
my ($orders2, $q2) = DB::EagerLoader->load_orders_eager;

printf "Naive loading:  %d orders, %d queries\n", scalar @$orders1, $q1;
printf "Eager loading:  %d orders, %d queries\n", scalar @$orders2, $q2;
printf "\nN+1 ratio: %.1fx fewer queries\n", $q1/$q2 if $q2;

printf "\nOrders (eager loaded):\n";
for my $o (@$orders2) {
    my $pname = $o->{product} ? $o->{product}{name} : "(deleted)";
    printf "  Order #%d: %s x%d (%s)\n", $o->{id}, $pname, $o->{quantity}, $o->{status};
}
```

---

## Step 308: Pagination with Cursor

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

{
package DB::Paginator;

# Offset-based (simple but has issues at large offsets)
sub paginate_offset {
    my ($class, $dbh, $table, %opts) = @_;
    my $page     = $opts{page}     // 1;
    my $per_page = $opts{per_page} // 10;
    my $order    = $opts{order}    // "id ASC";
    
    my $total  = $dbh->selectrow_array("SELECT COUNT(*) FROM $table") // 0;
    my $pages  = int(($total + $per_page - 1) / $per_page) || 1;
    my $offset = ($page - 1) * $per_page;
    
    my $items = $dbh->selectall_arrayref(
        "SELECT * FROM $table ORDER BY $order LIMIT ? OFFSET ?",
        {Slice=>{}}, $per_page, $offset
    );
    
    return {
        items   => $items,
        total   => $total,
        page    => $page,
        pages   => $pages,
        per_page => $per_page,
        has_prev => $page > 1,
        has_next => $page < $pages,
    };
}

# Cursor-based (efficient for large datasets)
sub paginate_cursor {
    my ($class, $dbh, $table, %opts) = @_;
    my $per_page  = $opts{per_page} // 10;
    my $after_id  = $opts{after_id} // 0;
    
    my $items = $dbh->selectall_arrayref(
        "SELECT * FROM $table WHERE id > ? ORDER BY id ASC LIMIT ?",
        {Slice=>{}}, $after_id, $per_page + 1  # Fetch one extra to detect next page
    );
    
    my $has_next = @$items > $per_page;
    my @page_items = @{$items}[0 .. ($per_page-1 < $#$items ? $per_page-1 : $#$items)];
    
    my $next_cursor = $has_next ? $page_items[-1]{id} : undef;
    
    return {
        items      => \@page_items,
        has_next   => $has_next,
        next_cursor => $next_cursor,
        count      => scalar @page_items,
    };
}
}

package main;

my $dbh = DB::Pool->get;

printf "=== Offset Pagination ===\n";
for my $page (1..3) {
    my $result = DB::Paginator->paginate_offset($dbh, "products", page => $page, per_page => 3);
    printf "Page %d/%d: %d items [%s]\n",
        $page, $result->{pages}, scalar @{$result->{items}},
        join(", ", map { $_->{name} } @{$result->{items}});
    last unless $result->{has_next};
}

printf "\n=== Cursor Pagination ===\n";
my $cursor = 0;
my $page_num = 0;
do {
    $page_num++;
    my $result = DB::Paginator->paginate_cursor($dbh, "products",
        per_page => 3, after_id => $cursor);
    printf "Page %d: %d items (next_cursor=%s)\n",
        $page_num, $result->{count}, $result->{next_cursor}//"end";
    printf "  %s\n", $_->{name} for @{$result->{items}};
    $cursor = $result->{next_cursor};
    last unless $result->{has_next};
} while (1);
```

---

## Step 309: Full-Text Search

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package DB::FTS;

sub setup_fts {
    my ($class, $dbh) = @_;
    eval {
        $dbh->do(q{
            CREATE VIRTUAL TABLE IF NOT EXISTS products_fts
            USING fts5(name, category, content='products', content_rowid='id')
        });
        $dbh->do("INSERT INTO products_fts(products_fts) VALUES('rebuild')");
    };
    if ($@) {
        # FTS5 not available, use manual LIKE search
        printf "(Note: FTS5 not available, using LIKE fallback)\n";
    }
}

sub search {
    my ($class, $dbh, $query, %opts) = @_;
    
    # Try FTS5 first
    my $use_fts = eval {
        $dbh->selectrow_array("SELECT * FROM products_fts LIMIT 1"); 1
    } // 0;
    
    if ($use_fts) {
        return @{$dbh->selectall_arrayref(
            "SELECT p.* FROM products p
             JOIN products_fts f ON p.id = f.rowid
             WHERE products_fts MATCH ? ORDER BY rank",
            {Slice=>{}}, $query
        )};
    }
    
    # Fallback: LIKE search
    my @terms   = split /\s+/, $query;
    my @conds   = map { "(name LIKE ? OR category LIKE ?)" } @terms;
    my @params  = map { ("%$_%", "%$_%") } @terms;
    
    return @{$dbh->selectall_arrayref(
        "SELECT * FROM products WHERE " . join(" AND ", @conds) . " ORDER BY name",
        {Slice=>{}}, @params
    )};
}

sub index_product {
    my ($class, $dbh, $id) = @_;
    eval {
        $dbh->do("INSERT OR REPLACE INTO products_fts(rowid, name, category) 
                  SELECT id, name, category FROM products WHERE id=?", undef, $id);
    };
}
}

package main;

my $dbh = DB::Pool->get;
DB::FTS->setup_fts($dbh);

printf "=== Full-Text / LIKE Search ===\n";

my @queries = ("widget", "gadget", "electronics", "book");
for my $q (@queries) {
    my @results = DB::FTS->search($dbh, $q);
    printf "Search '%s': %d results\n", $q, scalar @results;
    printf "  - %s (%s)\n", $_->{name}, $_->{category} for @results;
}
```

---

## Step 310: Capstone — Blog Database Layer

```perl
#!/usr/bin/perl
# blog_db.pl — Complete blog database layer
use strict;
use warnings;
use DBI;
use JSON::PP;

my $dbh = DBI->connect("dbi:SQLite::memory:", "", "", {
    RaiseError => 1, AutoCommit => 1, sqlite_unicode => 1 });
$dbh->do("PRAGMA foreign_keys = ON");

# Schema
for my $sql (split /;/, q{
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT UNIQUE,
    email TEXT, password TEXT, bio TEXT
);
CREATE TABLE posts (
    id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER,
    title TEXT NOT NULL, slug TEXT UNIQUE, body TEXT,
    published INTEGER DEFAULT 0, views INTEGER DEFAULT 0,
    created_at INTEGER DEFAULT (strftime('%s','now'))
);
CREATE TABLE comments (
    id INTEGER PRIMARY KEY AUTOINCREMENT, post_id INTEGER, user_id INTEGER,
    body TEXT, created_at INTEGER DEFAULT (strftime('%s','now'))
);
CREATE TABLE tags (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT UNIQUE);
CREATE TABLE post_tags (post_id INTEGER, tag_id INTEGER, PRIMARY KEY (post_id, tag_id))
}) {
    $sql =~ s/^\s+|\s+$//g;
    $dbh->do($sql) if $sql;
}

# Seed data
$dbh->do("INSERT INTO users (username, email, password) VALUES (?,?,?)", undef,
    "alice", "alice\@blog.com", "hashed");
$dbh->do("INSERT INTO users (username, email, password) VALUES (?,?,?)", undef,
    "bob", "bob\@blog.com", "hashed2");

my @post_data = (
    [1, "Hello World", "hello-world", "First post body", 1],
    [1, "Perl Tips",   "perl-tips",   "Useful Perl tips", 1],
    [2, "My Draft",    "my-draft",    "WIP...", 0],
    [1, "Advanced DBI","advanced-dbi","Deep dive into DBI", 1],
);

for my $p (@post_data) {
    $dbh->do("INSERT INTO posts (user_id, title, slug, body, published) VALUES (?,?,?,?,?)",
        undef, @$p);
}

# Tags
for my $tag (qw(perl dbi tutorial beginner advanced)) {
    $dbh->do("INSERT INTO tags (name) VALUES (?)", undef, $tag);
}

# Post-tag associations
my %tag_ids = map {
    my $t = $dbh->selectrow_hashref("SELECT * FROM tags WHERE name=?", undef, $_);
    $_ => $t->{id}
} qw(perl dbi tutorial beginner advanced);

my %post_ids = map {
    my $p = $dbh->selectrow_hashref("SELECT * FROM posts WHERE slug=?", undef, $_);
    $_ => $p->{id}
} qw(hello-world perl-tips advanced-dbi);

$dbh->do("INSERT INTO post_tags VALUES (?,?)", undef, $post_ids{"hello-world"}, $tag_ids{perl});
$dbh->do("INSERT INTO post_tags VALUES (?,?)", undef, $post_ids{"hello-world"}, $tag_ids{tutorial});
$dbh->do("INSERT INTO post_tags VALUES (?,?)", undef, $post_ids{"perl-tips"},   $tag_ids{perl});
$dbh->do("INSERT INTO post_tags VALUES (?,?)", undef, $post_ids{"perl-tips"},   $tag_ids{beginner});
$dbh->do("INSERT INTO post_tags VALUES (?,?)", undef, $post_ids{"advanced-dbi"},$tag_ids{dbi});
$dbh->do("INSERT INTO post_tags VALUES (?,?)", undef, $post_ids{"advanced-dbi"},$tag_ids{advanced});

# Comments
$dbh->do("INSERT INTO comments (post_id, user_id, body) VALUES (?,?,?)", undef,
    $post_ids{"hello-world"}, 2, "Great first post!");
$dbh->do("INSERT INTO comments (post_id, user_id, body) VALUES (?,?,?)", undef,
    $post_ids{"perl-tips"}, 2, "Very helpful!");
$dbh->do("INSERT INTO comments (post_id, user_id, body) VALUES (?,?,?)", undef,
    $post_ids{"perl-tips"}, 1, "Thanks!");

# Query: Published posts with author and tag count
printf "=== Published Posts ===\n";
my $posts = $dbh->selectall_arrayref(q{
    SELECT p.id, p.title, p.slug, p.views, u.username AS author,
           COUNT(DISTINCT pt.tag_id) AS tag_count,
           COUNT(DISTINCT c.id) AS comment_count
    FROM posts p
    JOIN users u ON p.user_id = u.id
    LEFT JOIN post_tags pt ON p.id = pt.post_id
    LEFT JOIN comments c ON p.id = c.id
    WHERE p.published = 1
    GROUP BY p.id
    ORDER BY p.created_at DESC
}, {Slice=>{}});

for my $p (@$posts) {
    printf "  [%s] \"%s\" by %s — %d tags, %d comments\n",
        $p->{slug}, $p->{title}, $p->{author}, $p->{tag_count}, $p->{comment_count};
}

# Query: Tags with post counts
printf "\n=== Tags ===\n";
my $tags = $dbh->selectall_arrayref(q{
    SELECT t.name, COUNT(pt.post_id) AS n
    FROM tags t LEFT JOIN post_tags pt ON t.id=pt.tag_id
    GROUP BY t.id ORDER BY n DESC, t.name
}, {Slice=>{}});
printf "  %-12s %d posts\n", $_->{name}, $_->{n} for @$tags;

# Query: User stats
printf "\n=== User Stats ===\n";
my $users = $dbh->selectall_arrayref(q{
    SELECT u.username,
           COUNT(DISTINCT p.id) AS posts,
           COUNT(DISTINCT c.id) AS comments
    FROM users u
    LEFT JOIN posts p ON u.id=p.user_id AND p.published=1
    LEFT JOIN comments c ON u.id=c.user_id
    GROUP BY u.id ORDER BY posts DESC
}, {Slice=>{}});
printf "  %-10s  %d posts  %d comments\n", $_->{username}, $_->{posts}, $_->{comments}
    for @$users;

# Increment view count
$dbh->do("UPDATE posts SET views=views+1 WHERE slug=?", undef, "perl-tips");
my $views = $dbh->selectrow_array("SELECT views FROM posts WHERE slug=?", undef, "perl-tips");
printf "\nperl-tips views: %d\n", $views;

printf "\nBlog database layer complete!\n";
```

---

## สรุป Part 31 — Advanced Database Patterns

ใน Part นี้คุณได้เรียนรู้:

### Concepts
- **Connection Pooling** — จัดการ DBI connections อย่างมีประสิทธิภาพ
- **Query Builder** — สร้าง SQL แบบ fluent interface
- **Active Record** — ORM pattern ที่ใช้ได้จริง
- **Transactions** — Atomic operations ด้วย commit/rollback
- **Migrations** — Version-controlled schema changes
- **Repository Pattern** — Domain-specific data access layer
- **N+1 Prevention** — Eager loading ลด queries
- **Pagination** — Offset-based และ cursor-based
- **Full-Text Search** — FTS5 หรือ LIKE fallback
- **Blog capstone** — Real-world multi-table queries

**ถัดไป: [Part 32 — Web Application Security](part_32.md)**
