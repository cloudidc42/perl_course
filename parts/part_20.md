# Part 20: Beginner Capstone Project
## Steps 191-200: โปรเจกต์จบระดับพื้นฐาน — Bookstore System

---

## Overview

ในตอนนี้เราจะสร้าง **Bookstore Management System** ที่รวมทุกสิ่งที่เรียนมาตั้งแต่ Part 1-19

**ฟีเจอร์:**
- จัดการหนังสือ (CRUD)
- ระบบสมาชิก
- ตะกร้าสินค้า + การสั่งซื้อ
- รายงาน
- Web interface (CGI)
- SQLite database
- Authentication

---

## Step 191: Database Schema

```perl
#!/usr/bin/perl
# schema.pl — Setup database
use strict;
use warnings;
use DBI;

my $dbh = DBI->connect("dbi:SQLite:dbname=/tmp/bookstore.db","","",
    { RaiseError => 1, AutoCommit => 1, sqlite_unicode => 1 });

my @sql = (
    # Books
    <<'SQL',
CREATE TABLE IF NOT EXISTS books (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    isbn        TEXT UNIQUE,
    title       TEXT NOT NULL,
    author      TEXT NOT NULL,
    publisher   TEXT,
    category    TEXT,
    price       REAL NOT NULL DEFAULT 0,
    stock       INTEGER DEFAULT 0,
    description TEXT,
    cover_url   TEXT,
    created_at  DATETIME DEFAULT (datetime('now'))
)
SQL

    # Users
    <<'SQL',
CREATE TABLE IF NOT EXISTS users (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    username      TEXT UNIQUE NOT NULL,
    email         TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    full_name     TEXT,
    phone         TEXT,
    address       TEXT,
    role          TEXT DEFAULT 'customer',
    created_at    DATETIME DEFAULT (datetime('now'))
)
SQL

    # Orders
    <<'SQL',
CREATE TABLE IF NOT EXISTS orders (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id     INTEGER NOT NULL,
    total       REAL NOT NULL DEFAULT 0,
    status      TEXT DEFAULT 'pending',
    shipping_addr TEXT,
    created_at  DATETIME DEFAULT (datetime('now')),
    FOREIGN KEY (user_id) REFERENCES users(id)
)
SQL

    # Order items
    <<'SQL',
CREATE TABLE IF NOT EXISTS order_items (
    id       INTEGER PRIMARY KEY AUTOINCREMENT,
    order_id INTEGER NOT NULL,
    book_id  INTEGER NOT NULL,
    qty      INTEGER NOT NULL DEFAULT 1,
    price    REAL NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (book_id)  REFERENCES books(id)
)
SQL

    # Cart
    <<'SQL',
CREATE TABLE IF NOT EXISTS cart (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    book_id INTEGER NOT NULL,
    qty     INTEGER NOT NULL DEFAULT 1,
    added_at DATETIME DEFAULT (datetime('now')),
    UNIQUE(user_id, book_id),
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (book_id) REFERENCES books(id)
)
SQL

    # Reviews
    <<'SQL',
CREATE TABLE IF NOT EXISTS reviews (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    book_id    INTEGER NOT NULL,
    user_id    INTEGER NOT NULL,
    rating     INTEGER CHECK(rating BETWEEN 1 AND 5),
    comment    TEXT,
    created_at DATETIME DEFAULT (datetime('now')),
    UNIQUE(book_id, user_id)
)
SQL

    # Sessions
    <<'SQL',
CREATE TABLE IF NOT EXISTS sessions (
    id          TEXT PRIMARY KEY,
    user_id     INTEGER,
    cart_data   TEXT DEFAULT '{}',
    expires_at  INTEGER NOT NULL
)
SQL
);

$dbh->do($_) for @sql;
print "Schema created.\n";

# =====================
# Seed data
# =====================

unless (($dbh->selectrow_array("SELECT COUNT(*) FROM books"))[0]) {
    my $ins = $dbh->prepare("INSERT INTO books (isbn,title,author,publisher,category,price,stock,description) VALUES (?,?,?,?,?,?,?,?)");
    
    my @books = (
        ["978-0-596-00492-7","Learning Perl","Randal L. Schwartz","O'Reilly","Perl",39.99,50,"The best intro to Perl programming"],
        ["978-0-596-00193-3","Programming Perl","Larry Wall","O'Reilly","Perl",54.99,30,"The Camel book — definitive Perl reference"],
        ["978-0-596-52010-6","Perl Cookbook","Tom Christiansen","O'Reilly","Perl",49.99,25,"Practical Perl programming recipes"],
        ["978-0-596-51494-2","Modern Perl","chromatic","O'Reilly","Perl",34.99,40,"Modern Perl idioms and practices"],
        ["978-0-596-00430-9","Mastering Regular Expressions","Jeffrey Friedl","O'Reilly","Programming",44.99,20,"The regex bible"],
        ["978-0-596-00523-8","CGI Programming with Perl","Scott Guelich","O'Reilly","Web",39.99,15,"Building web apps with CGI and Perl"],
        ["978-0-596-00227-5","Learning MySQL","Seyed Tahaghoghi","O'Reilly","Database",39.99,35,"MySQL for developers"],
        ["978-0-596-00145-2","Web Design in a Nutshell","Jennifer Niederst","O'Reilly","Web",34.99,28,"Complete web design reference"],
    );
    
    $ins->execute(@$_) for @books;
    print "Seeded ", scalar @books, " books.\n";
}

$dbh->disconnect;
print "Setup complete.\n";
```

---

## Step 192: Model Layer

```perl
#!/usr/bin/perl
# models.pm — Data models
package BookstoreDB;

use strict;
use warnings;
use DBI;
use Digest::SHA qw(sha256_hex);
use JSON::PP;

my $DBH;

sub init {
    $DBH = DBI->connect("dbi:SQLite:dbname=/tmp/bookstore.db","","",
        { RaiseError => 1, AutoCommit => 1, sqlite_unicode => 1 });
}

sub dbh { $DBH }

# =====================
# Book Model
# =====================

package Model::Book;

sub all {
    my (%opts) = @_;
    my $limit  = $opts{limit}  // 20;
    my $offset = $opts{offset} // 0;
    my $cat    = $opts{category};
    my $search = $opts{search};
    
    my $sql    = "SELECT b.*, COALESCE(AVG(r.rating),0) AS avg_rating, COUNT(r.id) AS review_count
                  FROM books b LEFT JOIN reviews r ON b.id = r.book_id";
    my @params;
    my @where;
    
    if ($cat) {
        push @where, "b.category = ?";
        push @params, $cat;
    }
    if ($search) {
        push @where, "(b.title LIKE ? OR b.author LIKE ? OR b.description LIKE ?)";
        push @params, "%$search%", "%$search%", "%$search%";
    }
    
    $sql .= " WHERE " . join(" AND ", @where) if @where;
    $sql .= " GROUP BY b.id ORDER BY b.title LIMIT ? OFFSET ?";
    push @params, $limit, $offset;
    
    return @{BookstoreDB::dbh->selectall_arrayref($sql, { Slice => {} }, @params)};
}

sub find {
    my $id = shift;
    return BookstoreDB::dbh->selectrow_hashref(
        "SELECT b.*, COALESCE(AVG(r.rating),0) AS avg_rating, COUNT(r.id) AS review_count
         FROM books b LEFT JOIN reviews r ON b.id = r.book_id
         WHERE b.id = ? GROUP BY b.id",
        undef, $id
    );
}

sub find_by_isbn {
    return BookstoreDB::dbh->selectrow_hashref(
        "SELECT * FROM books WHERE isbn = ?", undef, $_[0]
    );
}

sub categories {
    my $rows = BookstoreDB::dbh->selectall_arrayref(
        "SELECT category, COUNT(*) AS count FROM books GROUP BY category ORDER BY category"
    );
    return map { { name => $_->[0], count => $_->[1] } } @$rows;
}

sub create {
    my (%d) = @_;
    BookstoreDB::dbh->do("INSERT INTO books (isbn,title,author,publisher,category,price,stock,description) VALUES (?,?,?,?,?,?,?,?)",
        undef, $d{isbn}//'', $d{title}, $d{author}, $d{publisher}//'',
        $d{category}//'', $d{price}//0, $d{stock}//0, $d{description}//'');
    return BookstoreDB::dbh->last_insert_id;
}

sub update {
    my ($id, %d) = @_;
    BookstoreDB::dbh->do("UPDATE books SET title=?,author=?,publisher=?,category=?,price=?,stock=?,description=? WHERE id=?",
        undef, $d{title}, $d{author}, $d{publisher}//'', $d{category}//'',
        $d{price}//0, $d{stock}//0, $d{description}//'', $id);
}

sub update_stock {
    my ($id, $delta) = @_;
    BookstoreDB::dbh->do("UPDATE books SET stock = stock + ? WHERE id = ?", undef, $delta, $id);
}

# =====================
# User Model
# =====================

package Model::User;

sub hash_password {
    my ($pwd, $salt) = @_;
    $salt //= join "", map { ('a'..'z','0'..'9')[rand 36] } 1..12;
    return (sha256_hex($salt . $pwd . "bookstore_secret"), $salt);
}

sub verify_password {
    my ($pwd, $hash, $salt) = @_;
    return sha256_hex($salt . $pwd . "bookstore_secret") eq $hash;
}

sub create {
    my (%d) = @_;
    my ($hash, $salt) = hash_password($d{password});
    eval {
        BookstoreDB::dbh->do("INSERT INTO users (username,email,password_hash,full_name,phone,address) VALUES (?,?,?,?,?,?)",
            undef, $d{username}, $d{email}, "$hash:$salt",
            $d{full_name}//'', $d{phone}//'', $d{address}//'');
    };
    return $@ ? undef : BookstoreDB::dbh->last_insert_id;
}

sub find { BookstoreDB::dbh->selectrow_hashref("SELECT * FROM users WHERE id=?", undef, $_[0]) }

sub find_by_username {
    BookstoreDB::dbh->selectrow_hashref("SELECT * FROM users WHERE username=?", undef, $_[0])
}

sub authenticate {
    my ($username, $password) = @_;
    my $user = find_by_username($username);
    return undef unless $user;
    return undef unless $user->{password_hash} =~ /^([^:]+):(.+)$/;
    return verify_password($password, $1, $2) ? $user : undef;
}

# =====================
# Cart Model
# =====================

package Model::Cart;

sub items {
    my $user_id = shift;
    return @{BookstoreDB::dbh->selectall_arrayref(<<'SQL', { Slice => {} }, $user_id)};
SELECT c.*, b.title, b.author, b.price AS unit_price, b.stock
FROM cart c
JOIN books b ON c.book_id = b.id
WHERE c.user_id = ?
ORDER BY c.added_at
SQL
}

sub add {
    my ($user_id, $book_id, $qty) = @_;
    $qty //= 1;
    BookstoreDB::dbh->do(<<'SQL', undef, $user_id, $book_id, $qty, $qty);
INSERT INTO cart (user_id, book_id, qty) VALUES (?,?,?)
ON CONFLICT(user_id,book_id) DO UPDATE SET qty = qty + ?
SQL
}

sub update_qty {
    my ($user_id, $book_id, $qty) = @_;
    if ($qty <= 0) {
        remove($user_id, $book_id);
    } else {
        BookstoreDB::dbh->do("UPDATE cart SET qty=? WHERE user_id=? AND book_id=?",
            undef, $qty, $user_id, $book_id);
    }
}

sub remove  { BookstoreDB::dbh->do("DELETE FROM cart WHERE user_id=? AND book_id=?", undef, $_[0], $_[1]) }
sub clear   { BookstoreDB::dbh->do("DELETE FROM cart WHERE user_id=?", undef, $_[0]) }

sub total {
    my $user_id = shift;
    my ($total) = BookstoreDB::dbh->selectrow_array(
        "SELECT COALESCE(SUM(c.qty * b.price), 0) FROM cart c JOIN books b ON c.book_id=b.id WHERE c.user_id=?",
        undef, $user_id
    );
    return $total;
}

sub count {
    my $user_id = shift;
    my ($count) = BookstoreDB::dbh->selectrow_array(
        "SELECT COALESCE(SUM(qty),0) FROM cart WHERE user_id=?", undef, $user_id
    );
    return $count;
}

# =====================
# Order Model
# =====================

package Model::Order;

sub create_from_cart {
    my ($user_id, $shipping_addr) = @_;
    
    my @items = Model::Cart::items($user_id);
    return undef unless @items;
    
    my $total = Model::Cart::total($user_id);
    my $dbh   = BookstoreDB::dbh;
    
    $dbh->{AutoCommit} = 0;
    my $order_id;
    
    eval {
        $dbh->do("INSERT INTO orders (user_id,total,shipping_addr) VALUES (?,?,?)",
            undef, $user_id, $total, $shipping_addr);
        $order_id = $dbh->last_insert_id;
        
        for my $item (@items) {
            # Check stock
            die "Insufficient stock for $item->{title}\n"
                if $item->{stock} < $item->{qty};
            
            $dbh->do("INSERT INTO order_items (order_id,book_id,qty,price) VALUES (?,?,?,?)",
                undef, $order_id, $item->{book_id}, $item->{qty}, $item->{unit_price});
            
            # Reduce stock
            $dbh->do("UPDATE books SET stock = stock - ? WHERE id = ?",
                undef, $item->{qty}, $item->{book_id});
        }
        
        Model::Cart::clear($user_id);
        $dbh->commit;
    };
    
    if ($@) {
        $dbh->rollback;
        $dbh->{AutoCommit} = 1;
        die $@;
    }
    
    $dbh->{AutoCommit} = 1;
    return $order_id;
}

sub find {
    my $id = shift;
    my $order = BookstoreDB::dbh->selectrow_hashref("SELECT * FROM orders WHERE id=?", undef, $id);
    return undef unless $order;
    
    $order->{items} = BookstoreDB::dbh->selectall_arrayref(<<'SQL', { Slice => {} }, $id);
SELECT oi.*, b.title, b.author, b.isbn
FROM order_items oi JOIN books b ON oi.book_id = b.id
WHERE oi.order_id = ?
SQL
    return $order;
}

sub by_user {
    my $user_id = shift;
    return @{BookstoreDB::dbh->selectall_arrayref(
        "SELECT * FROM orders WHERE user_id=? ORDER BY created_at DESC",
        { Slice => {} }, $user_id
    )};
}

sub update_status {
    my ($id, $status) = @_;
    BookstoreDB::dbh->do("UPDATE orders SET status=? WHERE id=?", undef, $status, $id);
}

# =====================
# Test it
# =====================

package main;

BookstoreDB::init();

# Create test user
my $uid = Model::User::create(
    username  => "testuser",
    email     => "test\@bookstore.com",
    password  => "secret123",
    full_name => "Test User",
);

if ($uid) {
    print "Created user: $uid\n";
} else {
    my $u = Model::User::find_by_username("testuser");
    $uid = $u->{id};
    print "Using existing user: $uid\n";
}

# Get books
my @books = Model::Book::all(limit => 3);
printf "Found %d books (showing 3)\n", scalar @books;
printf "  - %s by %s (\$%.2f)\n", $_->{title}, $_->{author}, $_->{price} for @books;

# Add to cart
Model::Cart::clear($uid);
Model::Cart::add($uid, $books[0]{id}, 1);
Model::Cart::add($uid, $books[1]{id}, 2);

my @cart = Model::Cart::items($uid);
printf "\nCart: %d items, total \$%.2f\n", scalar @cart, Model::Cart::total($uid);

# Create order
eval {
    my $oid = Model::Order::create_from_cart($uid, "123 Main St, Bangkok");
    my $order = Model::Order::find($oid);
    printf "\nOrder #%d created — Total: \$%.2f\n", $order->{id}, $order->{total};
    printf "  - %s x%d @ \$%.2f\n", $_->{title}, $_->{qty}, $_->{price} for @{$order->{items}};
};
print "Error: $@" if $@;

BookstoreDB::dbh->disconnect;
```

---

## Step 193: CGI Web Interface

```perl
#!/usr/bin/perl
# bookstore.pl — Main CGI handler
use strict;
use warnings;
use CGI;
use CGI::Carp qw(fatalsToBrowser);

# Load models
do 'models.pm';  # In real app: use lib '.'; use BookstoreDB; etc.

# For this demo, we inline what's needed
use DBI;
use Digest::SHA qw(sha256_hex);
use JSON::PP;

my $cgi = CGI->new;

# =====================
# Init DB
# =====================

my $dbh = DBI->connect("dbi:SQLite:dbname=/tmp/bookstore.db","","",
    { RaiseError => 1, AutoCommit => 1, sqlite_unicode => 1 });

# =====================
# Utilities
# =====================

sub esc {
    my $s = shift // "";
    $s =~ s/&/&amp;/g;
    $s =~ s/</&lt;/g;
    $s =~ s/>/&gt;/g;
    $s =~ s/"/&quot;/g;
    return $s;
}

sub redirect {
    print "Status: 302\n";
    print "Location: bookstore.pl?page=" . $_[0] . "\n\n";
    exit;
}

sub stars {
    my $n = shift // 0;
    my $full  = int($n);
    my $empty = 5 - $full;
    return ("★" x $full) . ("☆" x $empty);
}

# =====================
# Session handling
# =====================

my $session_id = $cgi->cookie('bs_session') // '';
my $session_user = undef;

if ($session_id) {
    my $sess = $dbh->selectrow_hashref(
        "SELECT * FROM sessions WHERE id=? AND expires_at>?",
        undef, $session_id, time()
    );
    if ($sess && $sess->{user_id}) {
        $session_user = $dbh->selectrow_hashref("SELECT * FROM users WHERE id=?",
            undef, $sess->{user_id});
    }
}

sub new_session {
    my ($user_id) = @_;
    my $sid = sha256_hex(time() . $$ . rand());
    $dbh->do("INSERT OR REPLACE INTO sessions (id,user_id,expires_at) VALUES (?,?,?)",
        undef, $sid, $user_id, time() + 3600);
    return $sid;
}

sub destroy_session {
    $dbh->do("DELETE FROM sessions WHERE id=?", undef, $session_id);
}

# =====================
# Layout
# =====================

sub layout {
    my ($title, $content) = @_;
    
    my $cart_count = 0;
    if ($session_user) {
        ($cart_count) = $dbh->selectrow_array(
            "SELECT COALESCE(SUM(qty),0) FROM cart WHERE user_id=?",
            undef, $session_user->{id}
        );
    }
    
    my $nav_user = $session_user
        ? sprintf('<a href="?page=account">👤 %s</a> <a href="?page=logout">ออก</a>', esc($session_user->{username}))
        : '<a href="?page=login">เข้าสู่ระบบ</a> <a href="?page=register">สมัคร</a>';
    
    return <<HTML;
<!DOCTYPE html>
<html lang="th">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>$title — Perl Bookstore</title>
<style>
* { box-sizing: border-box; margin: 0; padding: 0; }
body { font-family: -apple-system, BlinkMacSystemFont, sans-serif; background: #f8f9fa; color: #333; }
header { background: #1a3a5c; color: white; padding: 0; }
.header-inner { max-width: 1000px; margin: 0 auto; padding: 12px 20px; display: flex; align-items: center; gap: 20px; }
.logo { font-size: 1.4em; font-weight: bold; color: white; text-decoration: none; }
.search-bar { flex: 1; display: flex; }
.search-bar input { flex: 1; padding: 8px 12px; border: 0; border-radius: 4px 0 0 4px; }
.search-bar button { padding: 8px 16px; background: #e67e22; color: white; border: 0; cursor: pointer; border-radius: 0 4px 4px 0; }
nav { background: #2c5aa0; }
.nav-inner { max-width: 1000px; margin: 0 auto; padding: 0 20px; display: flex; align-items: center; gap: 5px; }
nav a { color: rgba(255,255,255,.85); padding: 10px; text-decoration: none; font-size: .9em; }
nav a:hover { color: white; background: rgba(255,255,255,.1); }
.nav-right { margin-left: auto; }
main { max-width: 1000px; margin: 20px auto; padding: 0 20px; }
.book-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(180px,1fr)); gap: 20px; margin: 20px 0; }
.book-card { background: white; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,.08); overflow: hidden; transition: box-shadow .2s; }
.book-card:hover { box-shadow: 0 4px 12px rgba(0,0,0,.15); }
.book-cover { background: #2c5aa0; height: 140px; display: flex; align-items: center; justify-content: center; color: white; font-size: 2em; }
.book-info { padding: 12px; }
.book-title { font-weight: bold; font-size: .95em; margin-bottom: 4px; }
.book-author { color: #666; font-size: .85em; margin-bottom: 8px; }
.book-price { color: #e67e22; font-size: 1.1em; font-weight: bold; }
.stars { color: #f39c12; }
.btn { display: inline-block; padding: 8px 16px; border-radius: 4px; border: none; cursor: pointer; font-size: .9em; text-decoration: none; }
.btn-primary { background: #2c5aa0; color: white; }
.btn-orange  { background: #e67e22; color: white; }
.btn-sm { padding: 5px 10px; font-size: .8em; }
.card { background: white; border-radius: 8px; padding: 20px; margin: 15px 0; box-shadow: 0 2px 4px rgba(0,0,0,.05); }
.page-title { font-size: 1.5em; margin-bottom: 15px; color: #1a3a5c; }
table.data { width: 100%; border-collapse: collapse; }
table.data th, table.data td { padding: 10px; border-bottom: 1px solid #eee; text-align: left; }
table.data th { background: #f0f4f8; }
input, select, textarea { padding: 10px; border: 1px solid #ddd; border-radius: 4px; font-size: 1em; }
.form-group { margin: 12px 0; }
.form-group label { display: block; margin-bottom: 5px; font-weight: bold; font-size: .9em; }
.form-group input, .form-group textarea, .form-group select { width: 100%; }
.alert { padding: 12px 16px; border-radius: 4px; margin: 10px 0; }
.alert-error { background: #ffe5e5; color: #c0392b; border: 1px solid #e74c3c; }
.alert-success { background: #e8f8e8; color: #27ae60; border: 1px solid #2ecc71; }
footer { text-align: center; padding: 30px; color: #999; font-size: .85em; margin-top: 40px; border-top: 1px solid #eee; }
</style>
</head>
<body>
<header>
  <div class="header-inner">
    <a class="logo" href="?page=home">📚 Perl Bookstore</a>
    <form class="search-bar" method="get">
      <input type="hidden" name="page" value="search">
      <input type="text" name="q" placeholder="ค้นหาหนังสือ...">
      <button type="submit">ค้นหา</button>
    </form>
    <div style="white-space:nowrap;font-size:.9em">$nav_user</div>
    <a href="?page=cart" style="color:white;font-size:.9em">🛒 $cart_count</a>
  </div>
</header>
<nav>
  <div class="nav-inner">
    <a href="?page=home">หน้าแรก</a>
    <a href="?page=books">หนังสือทั้งหมด</a>
    <a href="?page=books&cat=Perl">Perl</a>
    <a href="?page=books&cat=Web">Web</a>
    <a href="?page=books&cat=Database">Database</a>
  </div>
</nav>
<main>
$content
</main>
<footer>© 2024 Perl Bookstore — Powered by Perl + CGI + SQLite</footer>
</body>
</html>
HTML
}

# =====================
# Page handlers
# =====================

my $page   = $cgi->param('page') // 'home';
my $method = $ENV{REQUEST_METHOD} // 'GET';
my $output = "";

# Handle auth POST
if ($page eq 'login' && $method eq 'POST') {
    my $username = $cgi->param('username');
    my $password = $cgi->param('password');
    my $user = $dbh->selectrow_hashref("SELECT * FROM users WHERE username=?", undef, $username);
    if ($user && $user->{password_hash} =~ /^([^:]+):(.+)$/) {
        if (sha256_hex($2 . $password . "bookstore_secret") eq $1) {
            my $sid = new_session($user->{id});
            print "Status: 302\n";
            print "Set-Cookie: bs_session=$sid; Path=/; HttpOnly\n";
            print "Location: ?page=home\n\n";
            exit;
        }
    }
    $output = "<div class='alert alert-error'>ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง</div>";
    $page = 'login';
}

if ($page eq 'logout') {
    destroy_session() if $session_id;
    print "Status: 302\n";
    print "Set-Cookie: bs_session=; Path=/; Expires=Thu, 01 Jan 1970 00:00:00 GMT\n";
    print "Location: ?page=home\n\n";
    exit;
}

if ($page eq 'add_to_cart' && $session_user) {
    my $book_id = int($cgi->param('book_id') // 0);
    my $qty     = int($cgi->param('qty') // 1);
    if ($book_id > 0 && $qty > 0) {
        $dbh->do("INSERT INTO cart (user_id,book_id,qty) VALUES (?,?,?) ON CONFLICT(user_id,book_id) DO UPDATE SET qty=qty+?",
            undef, $session_user->{id}, $book_id, $qty, $qty);
    }
    redirect('cart');
}

print "Content-Type: text/html; charset=utf-8\n\n";

# Home page
if ($page eq 'home') {
    my $featured = $dbh->selectall_arrayref(
        "SELECT * FROM books ORDER BY RANDOM() LIMIT 6", { Slice => {} }
    );
    
    my $cards = join "", map {
        my $b = $_;
        my $emoji = $b->{category} eq 'Perl' ? '🐪' :
                    $b->{category} eq 'Web'   ? '🌐' :
                    $b->{category} eq 'Database' ? '🗄️' : '📖';
        <<CARD;
<div class="book-card">
  <div class="book-cover">$emoji</div>
  <div class="book-info">
    <div class="book-title"><a href="?page=book&id=$b->{id}" style="color:inherit;text-decoration:none">@{[esc($b->{title})]}</a></div>
    <div class="book-author">@{[esc($b->{author})]}</div>
    <div class="book-price">\$$b->{price}</div>
    <form method="post" style="margin-top:8px">
      <input type="hidden" name="page" value="add_to_cart">
      <input type="hidden" name="book_id" value="$b->{id}">
      <button class="btn btn-orange btn-sm" type="submit">+ ตะกร้า</button>
    </form>
  </div>
</div>
CARD
    } @$featured;
    
    $output = <<HTML;
<h2 class="page-title">หนังสือแนะนำ</h2>
<div class="book-grid">$cards</div>
HTML

# Book list
} elsif ($page eq 'books' || $page eq 'search') {
    my $search = esc($cgi->param('q') // '');
    my $cat    = esc($cgi->param('cat') // '');
    my $offset = int($cgi->param('offset') // 0);
    
    my $sql    = "SELECT b.*, COALESCE(AVG(r.rating),0) AS avg_rating FROM books b LEFT JOIN reviews r ON b.id=r.book_id";
    my @params;
    my @where;
    
    if ($cat)    { push @where, "b.category=?"; push @params, $cat }
    if ($search) { push @where, "(b.title LIKE ? OR b.author LIKE ?)"; push @params, "%$search%", "%$search%" }
    
    $sql .= " WHERE " . join(" AND ", @where) if @where;
    $sql .= " GROUP BY b.id ORDER BY b.title LIMIT 12 OFFSET ?";
    push @params, $offset;
    
    my $books = $dbh->selectall_arrayref($sql, { Slice => {} }, @params);
    my $title = $search ? "ค้นหา: \"$search\"" : $cat ? "หมวด: $cat" : "หนังสือทั้งหมด";
    
    my $cards = join "", map {
        my $b = $_;
        <<CARD;
<div class="book-card">
  <div class="book-cover">📖</div>
  <div class="book-info">
    <div class="book-title"><a href="?page=book&id=$b->{id}" style="color:inherit;text-decoration:none">@{[esc($b->{title})]}</a></div>
    <div class="book-author">@{[esc($b->{author})]}</div>
    <div class="stars">@{[stars($b->{avg_rating})]}</div>
    <div class="book-price">\$@{[sprintf "%.2f", $b->{price}]}</div>
    <form method="post" style="margin-top:8px">
      <input type="hidden" name="page" value="add_to_cart">
      <input type="hidden" name="book_id" value="$b->{id}">
      <button class="btn btn-orange btn-sm">+ ตะกร้า</button>
    </form>
  </div>
</div>
CARD
    } @$books;
    
    $output = <<HTML;
<h2 class="page-title">$title</h2>
<div class="book-grid">$cards</div>
HTML

# Cart page
} elsif ($page eq 'cart') {
    if (!$session_user) {
        $output = "<div class='card'><p>กรุณา <a href='?page=login'>เข้าสู่ระบบ</a> เพื่อดูตะกร้าสินค้า</p></div>";
    } else {
        my $items = $dbh->selectall_arrayref(<<'SQL', { Slice => {} }, $session_user->{id});
SELECT c.*, b.title, b.author, b.price AS unit_price
FROM cart c JOIN books b ON c.book_id=b.id
WHERE c.user_id=? ORDER BY c.added_at
SQL
        my $total = 0;
        $total += $_->{unit_price} * $_->{qty} for @$items;
        
        my $rows = join "", map {
            my $i = $_;
            my $subtotal = $i->{unit_price} * $i->{qty};
            <<ROW;
<tr>
  <td>@{[esc($i->{title})]}</td>
  <td>@{[esc($i->{author})]}</td>
  <td>\$@{[sprintf "%.2f", $i->{unit_price}]}</td>
  <td>$i->{qty}</td>
  <td>\$@{[sprintf "%.2f", $subtotal]}</td>
</tr>
ROW
        } @$items;
        
        $output = <<HTML;
<div class="card">
<h2 class="page-title">🛒 ตะกร้าสินค้า</h2>
<table class="data">
<tr><th>หนังสือ</th><th>ผู้แต่ง</th><th>ราคา</th><th>จำนวน</th><th>รวม</th></tr>
$rows
</table>
<p style="text-align:right;font-size:1.3em;margin-top:15px"><strong>ยอดรวม: \$@{[sprintf "%.2f", $total]}</strong></p>
<div style="text-align:right">
  <a href="?page=checkout" class="btn btn-primary">ชำระเงิน →</a>
</div>
</div>
HTML
    }

# Login page
} elsif ($page eq 'login') {
    $output .= <<HTML;
<div class="card" style="max-width:400px;margin:40px auto">
<h2 class="page-title">เข้าสู่ระบบ</h2>
<form method="post">
<input type="hidden" name="page" value="login">
<div class="form-group"><label>Username</label><input type="text" name="username" required></div>
<div class="form-group"><label>Password</label><input type="password" name="password" required></div>
<button type="submit" class="btn btn-primary" style="width:100%">เข้าสู่ระบบ</button>
</form>
<p style="margin-top:15px;text-align:center">ยังไม่มีบัญชี? <a href="?page=register">สมัครสมาชิก</a></p>
</div>
HTML

} else {
    $output = "<div class='card'><h2>หน้านี้กำลังพัฒนา</h2></div>";
}

print layout("Perl Bookstore", $output);
$dbh->disconnect;
```

---

## Step 194: Admin Panel

```perl
#!/usr/bin/perl
# admin.pl — Admin interface
use strict;
use warnings;
use CGI;
use DBI;
use Digest::SHA qw(sha256_hex);

my $cgi = CGI->new;
my $dbh = DBI->connect("dbi:SQLite:dbname=/tmp/bookstore.db","","",
    { RaiseError => 1, AutoCommit => 1, sqlite_unicode => 1 });

sub esc { my $s=shift//""; $s=~s/&/&amp;/g; $s=~s/</&lt;/g; $s=~s/>/&gt;/g; $s }

# Check admin session
my $sid  = $cgi->cookie('bs_session') // '';
my $user = undef;

if ($sid) {
    my $sess = $dbh->selectrow_hashref("SELECT * FROM sessions WHERE id=? AND expires_at>?", undef, $sid, time());
    if ($sess) {
        $user = $dbh->selectrow_hashref("SELECT * FROM users WHERE id=? AND role='admin'", undef, $sess->{user_id});
    }
}

print "Content-Type: text/html; charset=utf-8\n\n";

unless ($user) {
    print "<h1>403 Forbidden</h1><p>Admin access required.</p>";
    exit;
}

my $action = $cgi->param('action') // 'dashboard';
my $method = $ENV{REQUEST_METHOD} // 'GET';

# Handle POST actions
if ($method eq 'POST') {
    if ($action eq 'add_book') {
        $dbh->do("INSERT INTO books (isbn,title,author,publisher,category,price,stock,description) VALUES (?,?,?,?,?,?,?,?)",
            undef,
            $cgi->param('isbn'), $cgi->param('title'), $cgi->param('author'),
            $cgi->param('publisher'), $cgi->param('category'),
            $cgi->param('price'), $cgi->param('stock'),
            $cgi->param('description'));
    } elsif ($action eq 'update_order_status') {
        $dbh->do("UPDATE orders SET status=? WHERE id=?",
            undef, $cgi->param('status'), $cgi->param('order_id'));
    }
}

print <<HTML;
<!DOCTYPE html>
<html lang="th">
<head>
<meta charset="utf-8">
<title>Admin Panel</title>
<style>
body { font-family: sans-serif; background: #f0f2f5; margin: 0; }
.sidebar { position: fixed; left: 0; top: 0; bottom: 0; width: 200px; background: #1a3a5c; color: white; padding: 20px 0; }
.sidebar a { display: block; padding: 10px 20px; color: rgba(255,255,255,.8); text-decoration: none; }
.sidebar a:hover { background: rgba(255,255,255,.1); color: white; }
.main { margin-left: 200px; padding: 20px; }
.card { background: white; border-radius: 8px; padding: 20px; margin: 15px 0; box-shadow: 0 2px 4px rgba(0,0,0,.05); }
.stat-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 15px; }
.stat { text-align: center; }
.stat-num { font-size: 2.5em; font-weight: bold; color: #2c5aa0; }
table { width: 100%; border-collapse: collapse; }
th,td { padding: 10px; border-bottom: 1px solid #eee; text-align: left; }
th { background: #f0f4f8; }
input, select, textarea { padding: 8px; border: 1px solid #ddd; border-radius: 4px; }
.btn { padding: 8px 16px; border: none; border-radius: 4px; cursor: pointer; color: white; }
.btn-blue { background: #2c5aa0; }
.btn-green { background: #27ae60; }
</style>
</head>
<body>
<div class="sidebar">
  <div style="padding: 20px; font-size: 1.1em; font-weight: bold; border-bottom: 1px solid rgba(255,255,255,.2)">
    📚 Admin
  </div>
  <a href="?action=dashboard">📊 Dashboard</a>
  <a href="?action=books">📖 Books</a>
  <a href="?action=orders">📦 Orders</a>
  <a href="?action=users">👥 Users</a>
  <a href="?action=add_book_form">+ Add Book</a>
  <a href="bookstore.pl">← Back to Store</a>
</div>
<div class="main">
HTML

if ($action eq 'dashboard') {
    my %stats = (
        books  => ($dbh->selectrow_array("SELECT COUNT(*) FROM books"))[0],
        users  => ($dbh->selectrow_array("SELECT COUNT(*) FROM users"))[0],
        orders => ($dbh->selectrow_array("SELECT COUNT(*) FROM orders"))[0],
        revenue => ($dbh->selectrow_array("SELECT COALESCE(SUM(total),0) FROM orders"))[0],
    );
    
    print <<HTML;
<h1>Dashboard</h1>
<div class="card">
<div class="stat-grid">
  <div class="stat"><div class="stat-num">$stats{books}</div><div>Books</div></div>
  <div class="stat"><div class="stat-num">$stats{users}</div><div>Users</div></div>
  <div class="stat"><div class="stat-num">$stats{orders}</div><div>Orders</div></div>
  <div class="stat"><div class="stat-num">\$@{[sprintf "%.0f", $stats{revenue}]}</div><div>Revenue</div></div>
</div>
</div>
HTML

    # Recent orders
    my $orders = $dbh->selectall_arrayref(<<'SQL', { Slice => {} });
SELECT o.*, u.username FROM orders o JOIN users u ON o.user_id=u.id ORDER BY o.created_at DESC LIMIT 5
SQL
    
    print "<div class='card'><h2>Recent Orders</h2><table><tr><th>#</th><th>User</th><th>Total</th><th>Status</th><th>Date</th></tr>";
    for my $o (@$orders) {
        printf "<tr><td>%d</td><td>%s</td><td>\$%.2f</td><td>%s</td><td>%s</td></tr>\n",
            $o->{id}, esc($o->{username}), $o->{total}, esc($o->{status}), $o->{created_at};
    }
    print "</table></div>";

} elsif ($action eq 'add_book_form') {
    print <<HTML;
<div class="card">
<h2>Add New Book</h2>
<form method="post">
<input type="hidden" name="action" value="add_book">
<div style="display:grid;grid-template-columns:1fr 1fr;gap:15px">
  <div><label>ISBN</label><br><input type="text" name="isbn" style="width:100%"></div>
  <div><label>Title *</label><br><input type="text" name="title" required style="width:100%"></div>
  <div><label>Author *</label><br><input type="text" name="author" required style="width:100%"></div>
  <div><label>Publisher</label><br><input type="text" name="publisher" style="width:100%"></div>
  <div><label>Category</label><br>
    <select name="category" style="width:100%">
      <option>Perl</option><option>Web</option><option>Database</option><option>Programming</option>
    </select>
  </div>
  <div><label>Price *</label><br><input type="number" name="price" step="0.01" required style="width:100%"></div>
  <div><label>Stock</label><br><input type="number" name="stock" value="0" style="width:100%"></div>
</div>
<div style="margin-top:10px"><label>Description</label><br><textarea name="description" rows="3" style="width:100%"></textarea></div>
<button type="submit" class="btn btn-green" style="margin-top:15px">Add Book</button>
</form>
</div>
HTML
}

print "</div></body></html>\n";
$dbh->disconnect;
```

---

## Step 195: Tests

```perl
#!/usr/bin/perl
# test_bookstore.pl — Unit tests
use strict;
use warnings;
use DBI;
use Digest::SHA qw(sha256_hex);

my ($pass, $fail) = (0, 0);

sub ok {
    my ($cond, $name) = @_;
    if ($cond) { print "ok - $name\n"; $pass++ }
    else { print "not ok - $name\n"; $fail++ }
}

sub is     { ok($_[0] eq $_[1], "$_[2] (got '$_[0]', expected '$_[1]')") }
sub is_num { ok($_[0] == $_[1], "$_[2] (got $_[0], expected $_[1])") }
sub like   { ok($_[0] =~ $_[1], "$_[2]") }

# In-memory test DB
my $dbh = DBI->connect("dbi:SQLite:dbname=:memory:","","",
    { RaiseError => 1, AutoCommit => 1 });

$dbh->do("CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT UNIQUE, email TEXT, password_hash TEXT, role TEXT DEFAULT 'customer')");
$dbh->do("CREATE TABLE books (id INTEGER PRIMARY KEY, title TEXT, author TEXT, price REAL, stock INTEGER)");
$dbh->do("CREATE TABLE cart (id INTEGER PRIMARY KEY, user_id INTEGER, book_id INTEGER, qty INTEGER, UNIQUE(user_id,book_id))");
$dbh->do("CREATE TABLE orders (id INTEGER PRIMARY KEY, user_id INTEGER, total REAL, status TEXT DEFAULT 'pending')");
$dbh->do("CREATE TABLE order_items (id INTEGER PRIMARY KEY, order_id INTEGER, book_id INTEGER, qty INTEGER, price REAL)");
$dbh->do("CREATE TABLE sessions (id TEXT PRIMARY KEY, user_id INTEGER, expires_at INTEGER)");

print "=== Bookstore Unit Tests ===\n\n";

# --- Password tests ---
print "-- Password hashing --\n";
my $salt = "testsalt12";
my $hash = sha256_hex($salt . "password123" . "bookstore_secret");
ok(length($hash) == 64, "hash is 64 chars");
ok(sha256_hex($salt . "password123" . "bookstore_secret") eq $hash, "same password produces same hash");
ok(sha256_hex($salt . "wrongpass" . "bookstore_secret") ne $hash, "wrong password produces different hash");

# --- User tests ---
print "\n-- User CRUD --\n";
my $salt2 = join "", map { ('a'..'z')[rand 26] } 1..8;
my $hash2 = sha256_hex($salt2 . "secret" . "bookstore_secret");
$dbh->do("INSERT INTO users (username,email,password_hash) VALUES (?,?,?)",
    undef, "alice", "alice\@test.com", "$hash2:$salt2");

my $uid = $dbh->last_insert_id;
my $user = $dbh->selectrow_hashref("SELECT * FROM users WHERE id=?", undef, $uid);
is($user->{username}, "alice",             "user username stored");
is($user->{email},    "alice\@test.com",   "user email stored");
is($user->{role},     "customer",          "default role is customer");

my $auth_user = $dbh->selectrow_hashref("SELECT * FROM users WHERE username=?", undef, "alice");
ok($auth_user && $auth_user->{password_hash} =~ /^([^:]+):(.+)$/, "password hash format correct");
my ($stored_hash, $stored_salt) = split /:/, $auth_user->{password_hash}, 2;
ok(sha256_hex($stored_salt . "secret" . "bookstore_secret") eq $stored_hash, "password verification works");

# --- Book tests ---
print "\n-- Book CRUD --\n";
$dbh->do("INSERT INTO books (title,author,price,stock) VALUES (?,?,?,?)",
    undef, "Learning Perl", "Schwartz", 39.99, 50);
$dbh->do("INSERT INTO books (title,author,price,stock) VALUES (?,?,?,?)",
    undef, "Programming Perl", "Wall", 54.99, 30);

my $books = $dbh->selectall_arrayref("SELECT * FROM books", { Slice => {} });
is_num(scalar @$books, 2, "two books inserted");

$dbh->do("UPDATE books SET stock = stock - ? WHERE id = ?", undef, 5, 1);
my ($stock) = $dbh->selectrow_array("SELECT stock FROM books WHERE id=?", undef, 1);
is_num($stock, 45, "stock reduced correctly");

# --- Cart tests ---
print "\n-- Cart operations --\n";
$dbh->do("INSERT INTO cart (user_id,book_id,qty) VALUES (?,?,?)", undef, $uid, 1, 2);
$dbh->do("INSERT INTO cart (user_id,book_id,qty) VALUES (?,?,?)", undef, $uid, 2, 1);

my $cart = $dbh->selectall_arrayref("SELECT * FROM cart WHERE user_id=?", { Slice => {} }, $uid);
is_num(scalar @$cart, 2, "two items in cart");

# Total calculation
my ($total) = $dbh->selectrow_array(
    "SELECT SUM(c.qty * b.price) FROM cart c JOIN books b ON c.book_id=b.id WHERE c.user_id=?",
    undef, $uid
);
is_num(sprintf("%.2f", $total), "134.97", "cart total calculated correctly (2×39.99 + 1×54.99)");

# Update cart
$dbh->do("UPDATE cart SET qty=? WHERE user_id=? AND book_id=?", undef, 3, $uid, 1);
my $item = $dbh->selectrow_hashref("SELECT * FROM cart WHERE user_id=? AND book_id=?", undef, $uid, 1);
is_num($item->{qty}, 3, "cart quantity updated");

# --- Order tests ---
print "\n-- Order creation --\n";
$dbh->do("INSERT INTO orders (user_id,total) VALUES (?,?)", undef, $uid, 134.97);
my $oid = $dbh->last_insert_id;
$dbh->do("INSERT INTO order_items (order_id,book_id,qty,price) VALUES (?,?,?,?)", undef, $oid, 1, 2, 39.99);
$dbh->do("INSERT INTO order_items (order_id,book_id,qty,price) VALUES (?,?,?,?)", undef, $oid, 2, 1, 54.99);

my $order = $dbh->selectrow_hashref("SELECT * FROM orders WHERE id=?", undef, $oid);
is($order->{status}, "pending", "new order status is pending");
is_num($order->{total}, 134.97, "order total correct");

$dbh->do("UPDATE orders SET status=? WHERE id=?", undef, "confirmed", $oid);
($order) = $dbh->selectrow_hashref("SELECT * FROM orders WHERE id=?", undef, $oid);
is($order->{status}, "confirmed", "order status updated");

my $items = $dbh->selectall_arrayref("SELECT * FROM order_items WHERE order_id=?", { Slice => {} }, $oid);
is_num(scalar @$items, 2, "two order items");

$dbh->disconnect;

print "\n---\n";
printf "%d tests passed, %d failed\n", $pass, $fail;
exit($fail > 0 ? 1 : 0);
```

---

## Step 196-200: Full Application Bundle

```perl
#!/usr/bin/perl
# run_bookstore.pl — Demo runner
#
# This script demonstrates all components working together
#

use strict;
use warnings;
use DBI;
use Digest::SHA qw(sha256_hex);
use JSON::PP;
use POSIX qw(strftime);

print "=" x 60 . "\n";
print "  PERL BOOKSTORE SYSTEM — Full Demo\n";
print "=" x 60 . "\n\n";

# =====================
# 1. Setup
# =====================

my $dbh = DBI->connect("dbi:SQLite:dbname=:memory:","","",
    { RaiseError => 1, AutoCommit => 1 });

for my $sql (
    "CREATE TABLE books (id INTEGER PRIMARY KEY AUTOINCREMENT, title TEXT, author TEXT, category TEXT, price REAL, stock INTEGER DEFAULT 0)",
    "CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT UNIQUE, email TEXT, password_hash TEXT, role TEXT DEFAULT 'customer')",
    "CREATE TABLE cart (id INTEGER PRIMARY KEY, user_id INTEGER, book_id INTEGER, qty INTEGER, UNIQUE(user_id,book_id))",
    "CREATE TABLE orders (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, total REAL, status TEXT DEFAULT 'pending')",
    "CREATE TABLE order_items (id INTEGER PRIMARY KEY, order_id INTEGER, book_id INTEGER, qty INTEGER, price REAL)",
    "CREATE TABLE reviews (id INTEGER PRIMARY KEY, book_id INTEGER, user_id INTEGER, rating INTEGER, comment TEXT)",
) { $dbh->do($sql) }

print "[1] Database schema created\n";

# =====================
# 2. Seed books
# =====================

my $ins_book = $dbh->prepare("INSERT INTO books (title,author,category,price,stock) VALUES (?,?,?,?,?)");
my @books = (
    ["Learning Perl",       "Schwartz",   "Perl",   39.99, 50],
    ["Programming Perl",    "Wall",       "Perl",   54.99, 30],
    ["Perl Cookbook",       "Christiansen","Perl",  49.99, 25],
    ["CGI Programming",     "Guelich",    "Web",    44.99, 20],
    ["MySQL Handbook",      "DuBois",     "Database",34.99, 40],
    ["Web Design Nutshell", "Niederst",   "Web",    39.99, 35],
);
$ins_book->execute(@$_) for @books;
printf "[2] Seeded %d books\n", scalar @books;

# =====================
# 3. Create users
# =====================

sub make_user {
    my ($username, $email, $password, $role) = @_;
    my $salt = join "", map { ('a'..'z')[rand 26] } 1..8;
    my $hash = sha256_hex($salt . $password . "secret");
    $dbh->do("INSERT INTO users (username,email,password_hash,role) VALUES (?,?,?,?)",
        undef, $username, $email, "$hash:$salt", $role//'customer');
    return $dbh->last_insert_id;
}

my $admin_id = make_user("admin", "admin\@store.com", "admin123", "admin");
my $alice_id = make_user("alice", "alice\@example.com", "alice123");
my $bob_id   = make_user("bob",   "bob\@example.com",   "bob123");
printf "[3] Created %d users (admin, alice, bob)\n", 3;

# =====================
# 4. Shopping simulation
# =====================

sub add_to_cart {
    my ($uid, $book_id, $qty) = @_;
    $dbh->do("INSERT INTO cart (user_id,book_id,qty) VALUES (?,?,?) ON CONFLICT(user_id,book_id) DO UPDATE SET qty=qty+?",
        undef, $uid, $book_id, $qty, $qty);
}

sub checkout {
    my ($uid) = @_;
    
    my $items = $dbh->selectall_arrayref(
        "SELECT c.book_id, c.qty, b.price, b.title, b.stock FROM cart c JOIN books b ON c.book_id=b.id WHERE c.user_id=?",
        { Slice => {} }, $uid
    );
    return undef unless @$items;
    
    my $total = 0;
    $total += $_->{price} * $_->{qty} for @$items;
    
    $dbh->{AutoCommit} = 0;
    my $oid;
    eval {
        $dbh->do("INSERT INTO orders (user_id,total) VALUES (?,?)", undef, $uid, $total);
        $oid = $dbh->last_insert_id;
        
        for my $item (@$items) {
            die "Out of stock: $item->{title}\n" if $item->{stock} < $item->{qty};
            $dbh->do("INSERT INTO order_items (order_id,book_id,qty,price) VALUES (?,?,?,?)",
                undef, $oid, $item->{book_id}, $item->{qty}, $item->{price});
            $dbh->do("UPDATE books SET stock = stock - ? WHERE id=?", undef, $item->{qty}, $item->{book_id});
        }
        $dbh->do("DELETE FROM cart WHERE user_id=?", undef, $uid);
        $dbh->commit;
    };
    if ($@) { $dbh->rollback; die $@ }
    $dbh->{AutoCommit} = 1;
    return $oid;
}

# Alice shops
add_to_cart($alice_id, 1, 2);   # 2x Learning Perl
add_to_cart($alice_id, 4, 1);   # 1x CGI Programming

my $a_cart = $dbh->selectall_arrayref(
    "SELECT b.title, c.qty, b.price FROM cart c JOIN books b ON c.book_id=b.id WHERE c.user_id=?",
    { Slice => {} }, $alice_id
);

printf "[4] Alice's cart:\n";
printf "    - %s x%d = \$%.2f\n", $_->{title}, $_->{qty}, $_->{price}*$_->{qty} for @$a_cart;

my $a_oid = checkout($alice_id);
printf "    Checkout → Order #%d\n", $a_oid;

# Bob shops
add_to_cart($bob_id, 2, 1);   # 1x Programming Perl
add_to_cart($bob_id, 5, 1);   # 1x MySQL Handbook
my $b_oid = checkout($bob_id);
printf "[5] Bob checked out → Order #%d\n", $b_oid;

# =====================
# 5. Reviews
# =====================

$dbh->do("INSERT INTO reviews (book_id,user_id,rating,comment) VALUES (?,?,?,?)",
    undef, 1, $alice_id, 5, "Best Perl book for beginners!");
$dbh->do("INSERT INTO reviews (book_id,user_id,rating,comment) VALUES (?,?,?,?)",
    undef, 2, $bob_id, 4, "Very comprehensive reference.");
printf "[6] Added reviews\n";

# =====================
# 6. Reports
# =====================

print "\n" . "=" x 60 . "\n";
print "SALES REPORT\n";
print "=" x 60 . "\n";

my $orders = $dbh->selectall_arrayref(
    "SELECT o.*, u.username FROM orders o JOIN users u ON o.user_id=u.id ORDER BY o.id",
    { Slice => {} }
);

for my $o (@$orders) {
    printf "Order #%d by %s — \$%.2f (%s)\n", $o->{id}, $o->{username}, $o->{total}, $o->{status};
    my $items = $dbh->selectall_arrayref(
        "SELECT oi.*, b.title FROM order_items oi JOIN books b ON oi.book_id=b.id WHERE oi.order_id=?",
        { Slice => {} }, $o->{id}
    );
    printf "  - %s x%d @ \$%.2f\n", $_->{title}, $_->{qty}, $_->{price} for @$items;
}

my ($revenue) = $dbh->selectrow_array("SELECT SUM(total) FROM orders");
printf "\nTotal Revenue: \$%.2f\n", $revenue;

print "\nBEST SELLERS:\n";
my $bestsellers = $dbh->selectall_arrayref(<<'SQL', { Slice => {} });
SELECT b.title, SUM(oi.qty) AS sold, SUM(oi.qty*oi.price) AS revenue
FROM order_items oi JOIN books b ON oi.book_id=b.id
GROUP BY b.id ORDER BY sold DESC
SQL
printf "  %-30s %4s %12s\n", "Title", "Sold", "Revenue";
printf "  %-30s %4d %12.2f\n", substr($_->{title},0,30), $_->{sold}, $_->{revenue}
    for @$bestsellers;

print "\nINVENTORY STATUS:\n";
my $inventory = $dbh->selectall_arrayref("SELECT title, stock FROM books ORDER BY stock", { Slice => {} });
for my $b (@$inventory) {
    my $status = $b->{stock} < 10 ? " ⚠ LOW" : "";
    printf "  %-30s %3d in stock%s\n", substr($b->{title},0,30), $b->{stock}, $status;
}

print "\nTOP RATED BOOKS:\n";
my $rated = $dbh->selectall_arrayref(<<'SQL', { Slice => {} });
SELECT b.title, AVG(r.rating) AS avg, COUNT(r.id) AS reviews
FROM reviews r JOIN books b ON r.book_id=b.id
GROUP BY b.id ORDER BY avg DESC
SQL
printf "  %-30s %s (%d reviews)\n",
    substr($_->{title},0,30), ("★" x int($_->{avg})) . ("☆" x (5-int($_->{avg}))), $_->{reviews}
for @$rated;

$dbh->disconnect;

print "\n" . "=" x 60 . "\n";
print "Bookstore demo complete!\n";
print "=" x 60 . "\n";
```

---

## สรุป Part 20 — Beginner Capstone

คุณได้สร้าง **Bookstore Management System** ที่สมบูรณ์ โดยใช้ทักษะที่เรียนมาทั้งหมด:

| Component | เทคโนโลยี |
|-----------|-----------|
| Web UI | CGI + HTML/CSS |
| Database | DBI + SQLite |
| Auth | SHA256 + Sessions |
| Cart & Orders | Transactions |
| Models | OOP Perl |
| Tests | Custom test framework |
| Reports | SQL aggregates |

**ยินดีด้วย!** คุณผ่านระดับ Beginner (Parts 1-20) แล้ว

**ถัดไป: [Part 21 — References และ Complex Data Structures](part_21.md)** — เข้าสู่ระดับ Intermediate!
