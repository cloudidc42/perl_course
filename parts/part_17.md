# Part 17: CGI Programming เบื้องต้น
## Steps 161-170: การสร้างเว็บด้วย CGI

---

## Step 161: CGI คืออะไร?

```
CGI (Common Gateway Interface) คือโปรโตคอลที่ให้เว็บเซิร์ฟเวอร์
เรียกใช้โปรแกรมภายนอก (เช่น Perl script) เพื่อสร้าง dynamic content

HTTP Request → Apache/Nginx → Perl CGI Script → HTML Response → Browser

Environment variables สำคัญ:
  REQUEST_METHOD   — GET หรือ POST
  QUERY_STRING     — ข้อมูลจาก URL (?name=value)
  CONTENT_LENGTH   — ขนาดของ POST data
  CONTENT_TYPE     — ประเภทของ POST data
  HTTP_COOKIE      — cookie header
  SERVER_NAME      — hostname ของ server
  REMOTE_ADDR      — IP ของ client
  SCRIPT_NAME      — path ของ script
  HTTP_USER_AGENT  — browser info
```

```perl
#!/usr/bin/perl
# File: /var/www/cgi-bin/hello.pl

use strict;
use warnings;

# =====================
# CGI header (ต้องส่งก่อน HTML)
# =====================

print "Content-Type: text/html; charset=utf-8\n";
print "\n";   # blank line = end of headers

# =====================
# HTML output
# =====================

print <<'HTML';
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="utf-8">
    <title>Hello Perl CGI</title>
    <style>
        body { font-family: sans-serif; padding: 20px; }
        h1   { color: #2c5aa0; }
    </style>
</head>
<body>
    <h1>สวัสดี Perl CGI!</h1>
    <p>นี่คือเว็บเพจแรกที่สร้างด้วย Perl CGI</p>
HTML

# Dynamic content
my $time = localtime;
print "<p>เวลาปัจจุบัน: <strong>$time</strong></p>\n";

# Server info
printf "<p>Server: %s</p>\n", $ENV{SERVER_NAME} // "localhost";
printf "<p>Client IP: %s</p>\n", $ENV{REMOTE_ADDR} // "unknown";
printf "<p>Method: %s</p>\n", $ENV{REQUEST_METHOD} // "CLI";

print "</body></html>\n";
```

---

## Step 162: CGI.pm Module

```perl
#!/usr/bin/perl
# File: /var/www/cgi-bin/cgi_module.pl

use strict;
use warnings;
use CGI qw(:standard);   # ใช้ CGI module
use CGI::Carp qw(fatalsToBrowser warningsToBrowser);

# warningsToBrowser(1);   # แสดง warnings ใน browser (dev only)

# =====================
# CGI object
# =====================

my $cgi = CGI->new;

# Print header
print $cgi->header(
    -type    => 'text/html',
    -charset => 'utf-8',
);

# HTML with helper functions
print $cgi->start_html(
    -title  => 'CGI.pm Demo',
    -style  => { -code => 'body { font-family: Arial; padding: 20px; }' },
);

print $cgi->h1("CGI.pm Demo");

# =====================
# Get parameters
# =====================

my $name  = $cgi->param('name')  // "World";
my $color = $cgi->param('color') // "blue";

# Sanitize (important!)
$name  =~ s/<[^>]*>//g;   # strip HTML tags
$color =~ s/[^a-zA-Z]//g; # only letters

print $cgi->p("Hello, ", $cgi->b($name), "!");
print $cgi->p("Your color is: ", 
    $cgi->font({-color => $color}, $color));

# =====================
# URL info
# =====================

print $cgi->h2("Request Info");
print $cgi->ul(
    $cgi->li("URL: "     . $cgi->url),
    $cgi->li("Method: "  . $cgi->request_method),
    $cgi->li("Script: "  . $cgi->script_name),
    $cgi->li("Query: "   . ($cgi->query_string || "(empty)")),
);

# =====================
# Form
# =====================

print $cgi->h2("Greeting Form");
print $cgi->start_form(-method => 'GET', -action => $cgi->script_name);
print $cgi->p("Your name: ", $cgi->textfield(-name => 'name', -size => 30));
print $cgi->p("Favorite color: ",
    $cgi->popup_menu(
        -name   => 'color',
        -values => [qw(red green blue purple orange)],
        -default => 'blue',
    )
);
print $cgi->submit("Submit");
print $cgi->end_form;

print $cgi->end_html;
```

---

## Step 163: HTML Helpers

```perl
#!/usr/bin/perl
use strict;
use warnings;

# =====================
# HTML generation functions
# =====================

sub html_tag {
    my ($tag, $attrs, $content) = @_;
    my $attr_str = "";
    if (ref $attrs eq 'HASH') {
        $attr_str = " " . join(" ", map { "$_=\"$attrs->{$_}\"" } sort keys %$attrs);
    }
    if (defined $content) {
        return "<$tag$attr_str>$content</$tag>";
    } else {
        return "<$tag$attr_str />";
    }
}

sub h1 { html_tag("h1", {}, shift) }
sub h2 { html_tag("h2", {}, shift) }
sub p  { html_tag("p",  {}, shift) }
sub b  { html_tag("b",  {}, shift) }
sub i  { html_tag("i",  {}, shift) }
sub a  { my ($href, $text) = @_; html_tag("a", { href => $href }, $text) }

sub table {
    my ($headers, @rows) = @_;
    my $html = "<table border='1' cellpadding='5'>\n";
    
    # Header row
    $html .= "<tr>";
    $html .= html_tag("th", {}, $_) for @$headers;
    $html .= "</tr>\n";
    
    # Data rows
    for my $row (@rows) {
        $html .= "<tr>";
        $html .= html_tag("td", {}, $_) for @$row;
        $html .= "</tr>\n";
    }
    
    $html .= "</table>\n";
    return $html;
}

sub ul_list {
    my @items = @_;
    my $html = "<ul>\n";
    $html .= html_tag("li", {}, $_) . "\n" for @items;
    $html .= "</ul>\n";
    return $html;
}

sub form_input {
    my %opts = @_;
    return html_tag("input", {
        type  => $opts{type}  // "text",
        name  => $opts{name}  // "",
        value => $opts{value} // "",
        size  => $opts{size}  // 30,
    });
}

# =====================
# Full page
# =====================

sub full_page {
    my ($title, $body) = @_;
    return <<HTML;
<!DOCTYPE html>
<html lang="th">
<head>
<meta charset="utf-8">
<title>$title</title>
<style>
  body { font-family: sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }
  table { border-collapse: collapse; width: 100%; }
  th { background: #2c5aa0; color: white; }
  tr:nth-child(even) { background: #f2f2f2; }
</style>
</head>
<body>
$body
</body>
</html>
HTML
}

# =====================
# Demo
# =====================

my @students = (
    ["Alice",  "A", 95, "Engineering"],
    ["Bob",    "B", 82, "Marketing"],
    ["Carol",  "A", 91, "Engineering"],
    ["Dave",   "C", 74, "Sales"],
);

my $table_html = table(
    ["Name", "Grade", "Score", "Department"],
    @students
);

my $body = h1("Student Report") .
           p("รายงานผลการเรียน") .
           $table_html .
           h2("Links") .
           ul_list(
               a("#", "Home"),
               a("#", "Students"),
               a("#", "Reports"),
           );

my $page = full_page("Student Report", $body);
print $page;
```

---

## Step 164: การรับข้อมูลจาก Form

```perl
#!/usr/bin/perl
# File: /var/www/cgi-bin/form_handler.pl

use strict;
use warnings;
use CGI;
use CGI::Carp qw(fatalsToBrowser);

my $cgi = CGI->new;

print $cgi->header(-type => 'text/html', -charset => 'utf-8');

# =====================
# Utility functions
# =====================

sub escape_html {
    my $str = shift // "";
    $str =~ s/&/&amp;/g;
    $str =~ s/</&lt;/g;
    $str =~ s/>/&gt;/g;
    $str =~ s/"/&quot;/g;
    $str =~ s/'/&#39;/g;
    return $str;
}

sub validate_email {
    my $email = shift;
    return $email =~ /^[^\@\s]+\@[^\@\s]+\.[^\@\s]{2,}$/;
}

sub validate_required {
    my $val = shift;
    return defined $val && $val =~ /\S/;
}

# =====================
# Form HTML
# =====================

my $form_html = <<'FORM';
<form method="post" action="">
  <fieldset>
    <legend>ข้อมูลส่วนตัว</legend>
    
    <p>
      <label>ชื่อ-นามสกุล: <input type="text" name="fullname" size="40" required></label>
    </p>
    <p>
      <label>อีเมล: <input type="email" name="email" size="40" required></label>
    </p>
    <p>
      <label>อายุ: <input type="number" name="age" min="1" max="120" size="5"></label>
    </p>
    <p>
      เพศ:
      <label><input type="radio" name="gender" value="M"> ชาย</label>
      <label><input type="radio" name="gender" value="F"> หญิง</label>
    </p>
    <p>
      <label>แผนก:
        <select name="dept">
          <option value="">-- เลือก --</option>
          <option value="it">IT</option>
          <option value="hr">HR</option>
          <option value="finance">Finance</option>
        </select>
      </label>
    </p>
    <p>
      ทักษะ:
      <label><input type="checkbox" name="skill" value="perl"> Perl</label>
      <label><input type="checkbox" name="skill" value="python"> Python</label>
      <label><input type="checkbox" name="skill" value="js"> JavaScript</label>
    </p>
    <p>
      <label>หมายเหตุ: <textarea name="notes" rows="3" cols="40"></textarea></label>
    </p>
    <p>
      <button type="submit">บันทึก</button>
      <button type="reset">ล้างข้อมูล</button>
    </p>
  </fieldset>
</form>
FORM

# =====================
# Process form
# =====================

my %errors;
my %data;

if ($cgi->request_method eq 'POST') {
    # Get all parameters
    $data{fullname} = $cgi->param('fullname') // "";
    $data{email}    = $cgi->param('email')    // "";
    $data{age}      = $cgi->param('age')      // "";
    $data{gender}   = $cgi->param('gender')   // "";
    $data{dept}     = $cgi->param('dept')     // "";
    $data{notes}    = $cgi->param('notes')    // "";
    
    # Multi-value param
    my @skills = $cgi->multi_param('skill');
    $data{skills} = \@skills;
    
    # Validate
    $errors{fullname} = "กรุณาใส่ชื่อ" unless validate_required($data{fullname});
    $errors{email}    = "อีเมลไม่ถูกต้อง" unless validate_email($data{email});
    $errors{age}      = "อายุไม่ถูกต้อง" if $data{age} && ($data{age} !~ /^\d+$/ || $data{age} > 120);
    $errors{dept}     = "กรุณาเลือกแผนก" unless $data{dept};
    
    # Sanitize
    $data{$_} = escape_html($data{$_}) for keys %data;
}

# =====================
# Output
# =====================

print <<HTML;
<!DOCTYPE html>
<html lang="th">
<head>
<meta charset="utf-8">
<title>Form Demo</title>
<style>
body { font-family: sans-serif; max-width: 600px; margin: 20px auto; }
.error { color: red; font-size: 0.9em; }
.success { background: #dff0d8; padding: 10px; border-radius: 4px; }
fieldset { padding: 15px; margin: 10px 0; }
p { margin: 8px 0; }
</style>
</head>
<body>
<h1>ฟอร์มลงทะเบียน</h1>
HTML

if ($cgi->request_method eq 'POST') {
    if (%errors) {
        print "<div class='error'><strong>กรุณาแก้ไข:</strong><ul>";
        print "<li>$_: $errors{$_}</li>" for sort keys %errors;
        print "</ul></div>\n";
        print $form_html;
    } else {
        print "<div class='success'>";
        print "<h2>บันทึกสำเร็จ!</h2>";
        print "<p>ชื่อ: $data{fullname}</p>";
        print "<p>อีเมล: $data{email}</p>";
        print "<p>อายุ: $data{age}</p>" if $data{age};
        print "<p>เพศ: $data{gender}</p>" if $data{gender};
        print "<p>แผนก: $data{dept}</p>";
        if (@{$data{skills}}) {
            print "<p>ทักษะ: ", join(", ", @{$data{skills}}), "</p>";
        }
        print "<p>หมายเหตุ: $data{notes}</p>" if $data{notes};
        print "</div>\n";
    }
} else {
    print $form_html;
}

print "</body></html>\n";
```

---

## Step 165: Cookies

```perl
#!/usr/bin/perl
use strict;
use warnings;
use CGI;

my $cgi = CGI->new;

# =====================
# Set cookie
# =====================

my $lang_cookie = $cgi->cookie(
    -name    => 'language',
    -value   => 'th',
    -expires => '+30d',    # 30 วัน
    -path    => '/',
    -secure  => 0,         # HTTPS only? set 1 in production
    -httponly => 1,        # ป้องกัน XSS
);

my $user_cookie = $cgi->cookie(
    -name   => 'username',
    -value  => 'alice',
    -expires => '+1d',
    -path   => '/',
);

# ส่ง header พร้อม cookies
print $cgi->header(
    -type    => 'text/html',
    -charset => 'utf-8',
    -cookie  => [$lang_cookie, $user_cookie],
);

# =====================
# Read cookies
# =====================

my $language = $cgi->cookie('language') // "en";
my $username = $cgi->cookie('username') // "Guest";

# =====================
# Delete cookie
# =====================

sub delete_cookie {
    my ($cgi, $name) = @_;
    return $cgi->cookie(
        -name    => $name,
        -value   => '',
        -expires => '-1d',   # ย้อนหลัง = ลบ
        -path    => '/',
    );
}

# =====================
# Session via cookie
# =====================

use Digest::MD5 qw(md5_hex);
use POSIX qw(strftime);

sub generate_session_id {
    my $data = join("", time(), $$, rand());
    return md5_hex($data);
}

# Simple in-memory session store (production: use file/DB/Redis)
my %SESSIONS;

sub create_session {
    my (%data) = @_;
    my $sid = generate_session_id();
    $SESSIONS{$sid} = {
        %data,
        created => time(),
        expires => time() + 3600,   # 1 hour
    };
    return $sid;
}

sub get_session {
    my $sid = shift;
    my $session = $SESSIONS{$sid} or return undef;
    return undef if time() > $session->{expires};
    return $session;
}

# Usage demo
my $sid = create_session(user => "Alice", role => "admin");
my $session = get_session($sid);

print <<HTML;
<!DOCTYPE html>
<html lang="th">
<head><meta charset="utf-8"><title>Cookie Demo</title></head>
<body>
<h1>Cookie Demo</h1>
<p>Language: $language</p>
<p>Username: $username</p>
<p>Session ID: $sid</p>
HTML

if ($session) {
    printf "<p>Session user: %s (role: %s)</p>\n",
        $session->{user}, $session->{role};
}

print <<HTML;
<h2>All cookies from request:</h2>
<ul>
HTML

for my $name ($cgi->cookie) {
    printf "<li>%s = %s</li>\n", $name, $cgi->cookie($name);
}

print "</ul></body></html>\n";
```

---

## Step 166: Session Management

```perl
#!/usr/bin/perl
use strict;
use warnings;
use CGI;
use CGI::Session;   # หรือสร้าง session เอง
use Digest::MD5 qw(md5_hex);
use JSON::PP;
use POSIX qw(strftime);

my $cgi = CGI->new;

# =====================
# Simple file-based session
# =====================

package Session;

my $SESSION_DIR = '/tmp/perl_sessions';
mkdir $SESSION_DIR unless -d $SESSION_DIR;

sub new {
    my ($class, $sid) = @_;
    
    $sid //= _generate_id();
    
    my $self = bless {
        id   => $sid,
        data => {},
        file => "$SESSION_DIR/$sid.json",
    }, $class;
    
    $self->_load if -e $self->{file};
    
    return $self;
}

sub id   { $_[0]->{id} }

sub get  { $_[0]->{data}{$_[1]} }
sub set  { $_[0]->{data}{$_[1]} = $_[2] }
sub del  { delete $_[0]->{data}{$_[1]} }
sub clear { $_[0]->{data} = {} }

sub save {
    my $self = shift;
    $self->{data}{_ts}  = time();
    my $json = JSON::PP->new->encode($self->{data});
    open(my $fh, '>', $self->{file}) or die $!;
    print $fh $json;
    close $fh;
}

sub _load {
    my $self = shift;
    open(my $fh, '<', $self->{file}) or return;
    local $/;
    my $json = <$fh>;
    close $fh;
    eval { $self->{data} = JSON::PP->new->decode($json) };
}

sub destroy {
    my $self = shift;
    unlink $self->{file} if -e $self->{file};
    $self->{data} = {};
}

sub _generate_id {
    return md5_hex(time() . $$ . rand());
}

sub is_expired {
    my ($self, $lifetime) = @_;
    $lifetime //= 3600;
    my $ts = $self->{data}{_ts} // 0;
    return time() - $ts > $lifetime;
}

package main;

# =====================
# Demo usage
# =====================

my $sess = Session->new;

$sess->set("user", "Alice");
$sess->set("role", "admin");
$sess->set("login_time", time());
$sess->save;

printf "Session ID: %s\n", $sess->id;
printf "User: %s\n", $sess->get("user");
printf "Role: %s\n", $sess->get("role");

# Simulate loading from existing session
my $loaded = Session->new($sess->id);
printf "Loaded user: %s\n", $loaded->get("user");
printf "Expired: %s\n", $loaded->is_expired(7200) ? "yes" : "no";

# Flash messages (one-time messages)
$sess->set("flash_success", "ลงทะเบียนสำเร็จ!");
$sess->save;

my $flash = $sess->get("flash_success");
$sess->del("flash_success");
$sess->save;

printf "Flash: %s\n", $flash // "(none)";
printf "Flash after read: %s\n", $sess->get("flash_success") // "(cleared)";

$sess->destroy;
```

---

## Step 167: Authentication

```perl
#!/usr/bin/perl
use strict;
use warnings;
use CGI;
use Digest::SHA qw(sha256_hex);
use MIME::Base64 qw(encode_base64url decode_base64url);
use JSON::PP;

my $cgi = CGI->new;

# =====================
# Password hashing
# =====================

sub hash_password {
    my ($password, $salt) = @_;
    $salt //= _random_salt();
    my $hash = sha256_hex($salt . $password . "secret_pepper");
    return ($hash, $salt);
}

sub verify_password {
    my ($password, $stored_hash, $salt) = @_;
    my ($hash) = hash_password($password, $salt);
    return $hash eq $stored_hash;
}

sub _random_salt {
    return join "", map { ("a".."z","A".."Z","0".."9")[rand 62] } 1..16;
}

# =====================
# Simple user store
# =====================

my %USERS;

sub register_user {
    my (%opts) = @_;
    
    die "Username taken\n" if exists $USERS{$opts{username}};
    
    my ($hash, $salt) = hash_password($opts{password});
    
    $USERS{$opts{username}} = {
        username  => $opts{username},
        email     => $opts{email},
        pw_hash   => $hash,
        pw_salt   => $salt,
        created   => time(),
        role      => $opts{role} // "user",
    };
    
    return 1;
}

sub login_user {
    my ($username, $password) = @_;
    
    my $user = $USERS{$username}
        or die "Invalid credentials\n";
    
    verify_password($password, $user->{pw_hash}, $user->{pw_salt})
        or die "Invalid credentials\n";
    
    return { %$user };
}

# =====================
# JWT-like token (simplified)
# =====================

my $SECRET = "my_jwt_secret_key_12345";

sub create_token {
    my (%payload) = @_;
    $payload{iat} = time();
    $payload{exp} = time() + 3600;
    
    my $json = JSON::PP->new->encode(\%payload);
    my $b64 = encode_base64url($json);
    my $sig = sha256_hex($b64 . $SECRET);
    
    return "$b64.$sig";
}

sub verify_token {
    my $token = shift;
    
    my ($b64, $sig) = split /\./, $token, 2;
    return undef unless $b64 && $sig;
    
    # Verify signature
    my $expected = sha256_hex($b64 . $SECRET);
    return undef unless $sig eq $expected;
    
    # Decode payload
    my $json = decode_base64url($b64);
    my $payload = eval { JSON::PP->new->decode($json) };
    return undef if $@;
    
    # Check expiry
    return undef if time() > $payload->{exp};
    
    return $payload;
}

# =====================
# Test
# =====================

print "=== Auth Demo ===\n";

# Register
register_user(
    username => "alice",
    password => "secure_password_123",
    email    => "alice\@example.com",
    role     => "admin",
);

# Login
my $user = eval { login_user("alice", "secure_password_123") };
if ($@) {
    print "Login failed: $@\n";
} else {
    printf "Logged in as: %s (role: %s)\n", $user->{username}, $user->{role};
    
    # Create token
    my $token = create_token(
        sub  => $user->{username},
        role => $user->{role},
    );
    printf "Token: %.40s...\n", $token;
    
    # Verify token
    my $claims = verify_token($token);
    if ($claims) {
        printf "Verified: sub=%s, role=%s\n", $claims->{sub}, $claims->{role};
    }
}

# Wrong password
eval { login_user("alice", "wrong_password") };
printf "Wrong password: %s\n", $@ if $@;
```

---

## Step 168: File Upload

```perl
#!/usr/bin/perl
use strict;
use warnings;
use CGI;
use CGI::Carp qw(fatalsToBrowser);
use File::Basename;
use POSIX qw(strftime);

my $cgi = CGI->new;
my $UPLOAD_DIR = '/tmp/uploads';
mkdir $UPLOAD_DIR unless -d $UPLOAD_DIR;

# =====================
# File upload HTML form
# =====================

my $form_html = <<'FORM';
<form method="post" enctype="multipart/form-data">
  <p>
    <label>เลือกไฟล์:
      <input type="file" name="upload" multiple accept=".jpg,.jpeg,.png,.gif,.pdf,.txt">
    </label>
  </p>
  <p>
    <label>คำอธิบาย:
      <input type="text" name="description" size="50">
    </label>
  </p>
  <p>
    <button type="submit">อัปโหลด</button>
  </p>
</form>
FORM

# =====================
# Handle upload
# =====================

print $cgi->header(-type => 'text/html', -charset => 'utf-8');

print <<HTML;
<!DOCTYPE html>
<html lang="th">
<head>
<meta charset="utf-8">
<title>File Upload</title>
</head>
<body>
<h1>อัปโหลดไฟล์</h1>
HTML

if ($cgi->request_method eq 'POST') {
    my @files = $cgi->upload('upload');
    my $desc  = $cgi->param('description') // "";
    
    if (@files) {
        print "<h2>ผลการอัปโหลด:</h2>\n<ul>\n";
        
        for my $fh (@files) {
            my $filename = basename($cgi->param('upload') // "unknown");
            $filename =~ s/[^\w\.\-]/_/g;  # sanitize filename
            
            my $info    = $cgi->uploadInfo($fh);
            my $type    = $info->{'Content-Type'} // "unknown";
            my $size    = 0;
            my $content = "";
            
            # Read file
            while (my $chunk = <$fh>) {
                $content .= $chunk;
                $size += length $chunk;
            }
            
            # Check size (max 5MB)
            if ($size > 5 * 1024 * 1024) {
                printf "<li>%s — ไฟล์ใหญ่เกินไป (%d bytes)</li>\n", 
                    $filename, $size;
                next;
            }
            
            # Check type
            my %allowed_types = (
                'image/jpeg' => 'jpg',
                'image/png'  => 'png',
                'image/gif'  => 'gif',
                'text/plain' => 'txt',
                'application/pdf' => 'pdf',
            );
            
            unless ($allowed_types{$type}) {
                printf "<li>%s — ประเภทไม่อนุญาต (%s)</li>\n",
                    $filename, $type;
                next;
            }
            
            # Save file
            my $ts = strftime "%Y%m%d%H%M%S", localtime;
            my $save_name = "${ts}_${filename}";
            my $save_path = "$UPLOAD_DIR/$save_name";
            
            open(my $out, '>', $save_path) or do {
                print "<li>$filename — บันทึกไม่ได้: $!</li>\n";
                next;
            };
            binmode $out;
            print $out $content;
            close $out;
            
            printf "<li>%s — สำเร็จ (%d bytes, %s)</li>\n",
                $filename, $size, $type;
        }
        
        print "</ul>\n";
    } else {
        print "<p>ไม่มีไฟล์ที่อัปโหลด</p>\n";
    }
    
    print "<p><a href=''>อัปโหลดอีกครั้ง</a></p>\n";
} else {
    print $form_html;
}

print "</body></html>\n";
```

---

## Step 169: Database กับ CGI

```perl
#!/usr/bin/perl
use strict;
use warnings;
use CGI;
use DBI;

my $cgi = CGI->new;

# =====================
# Database connection
# =====================

my $dbh = DBI->connect(
    "dbi:SQLite:dbname=/tmp/users.db",
    "", "",
    {
        RaiseError    => 1,
        AutoCommit    => 1,
        sqlite_unicode => 1,
    }
) or die "Cannot connect: $DBI::errstr";

# Create table if not exists
$dbh->do(<<'SQL');
CREATE TABLE IF NOT EXISTS users (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    username   TEXT UNIQUE NOT NULL,
    email      TEXT NOT NULL,
    fullname   TEXT,
    created_at TEXT DEFAULT (datetime('now'))
)
SQL

# =====================
# CRUD operations
# =====================

sub get_all_users {
    my $sth = $dbh->prepare("SELECT * FROM users ORDER BY created_at DESC");
    $sth->execute;
    return @{$sth->fetchall_arrayref({})};
}

sub get_user {
    my $id = shift;
    my $sth = $dbh->prepare("SELECT * FROM users WHERE id = ?");
    $sth->execute($id);
    return $sth->fetchrow_hashref;
}

sub create_user {
    my (%data) = @_;
    my $sth = $dbh->prepare(
        "INSERT INTO users (username, email, fullname) VALUES (?, ?, ?)"
    );
    $sth->execute($data{username}, $data{email}, $data{fullname});
    return $dbh->last_insert_id;
}

sub update_user {
    my ($id, %data) = @_;
    my $sth = $dbh->prepare(
        "UPDATE users SET username=?, email=?, fullname=? WHERE id=?"
    );
    $sth->execute($data{username}, $data{email}, $data{fullname}, $id);
    return $sth->rows;
}

sub delete_user {
    my $id = shift;
    $dbh->do("DELETE FROM users WHERE id = ?", undef, $id);
}

sub search_users {
    my $q = shift;
    my $sth = $dbh->prepare(
        "SELECT * FROM users WHERE username LIKE ? OR email LIKE ? OR fullname LIKE ?"
    );
    my $pattern = "%$q%";
    $sth->execute($pattern, $pattern, $pattern);
    return @{$sth->fetchall_arrayref({})};
}

# =====================
# CGI handler
# =====================

print $cgi->header(-type => 'text/html', -charset => 'utf-8');

my $action = $cgi->param('action') // "list";

sub escape_html {
    my $s = shift // "";
    $s =~ s/&/&amp;/g;
    $s =~ s/</&lt;/g;
    $s =~ s/>/&gt;/g;
    $s =~ s/"/&quot;/g;
    return $s;
}

print <<HTML;
<!DOCTYPE html><html lang="th">
<head><meta charset="utf-8"><title>User Manager</title>
<style>
body { font-family: sans-serif; max-width: 900px; margin: 20px auto; }
table { width: 100%; border-collapse: collapse; }
th, td { padding: 8px; border: 1px solid #ddd; }
th { background: #2c5aa0; color: white; }
tr:nth-child(even) { background: #f5f5f5; }
.btn { padding: 5px 10px; text-decoration: none; color: white; border-radius: 3px; }
.btn-blue { background: #2c5aa0; }
.btn-red  { background: #c0392b; }
</style>
</head>
<body>
<h1>User Manager</h1>
HTML

if ($action eq 'add' || $action eq 'create') {
    # Handle form submission
    if ($action eq 'create') {
        my $username = $cgi->param('username');
        my $email    = $cgi->param('email');
        my $fullname = $cgi->param('fullname');
        
        eval { create_user(username => $username, email => $email, fullname => $fullname) };
        if ($@) {
            print "<p style='color:red'>Error: $@</p>\n";
        } else {
            print "<p style='color:green'>สร้างผู้ใช้สำเร็จ</p>\n";
        }
    }
    
    # Show form
    print <<FORM;
<h2>เพิ่มผู้ใช้ใหม่</h2>
<form method="post">
<input type="hidden" name="action" value="create">
<p>Username: <input type="text" name="username" required></p>
<p>Email: <input type="email" name="email" required></p>
<p>Full Name: <input type="text" name="fullname"></p>
<p><button type="submit">บันทึก</button>
   <a href="?">ยกเลิก</a></p>
</form>
FORM
    
} elsif ($action eq 'delete') {
    my $id = $cgi->param('id');
    delete_user($id) if $id;
    print "<p>ลบแล้ว</p>\n";
}

# Show user list
my $search = $cgi->param('q') // "";
my @users = $search ? search_users($search) : get_all_users();

print "<p><a href='?action=add' class='btn btn-blue'>+ เพิ่มผู้ใช้</a></p>\n";
print "<form><input type='text' name='q' value='", escape_html($search), "' placeholder='ค้นหา...'>";
print " <button type='submit'>ค้นหา</button></form>\n";
printf "<p>พบ %d ผู้ใช้</p>\n", scalar @users;

print "<table><tr><th>ID</th><th>Username</th><th>Email</th><th>Full Name</th><th>Created</th><th>Action</th></tr>\n";

for my $u (@users) {
    printf "<tr><td>%s</td><td>%s</td><td>%s</td><td>%s</td><td>%s</td><td><a href='?action=delete&id=%s' class='btn btn-red' onclick='return confirm(\"ลบ?\")'>ลบ</a></td></tr>\n",
        map { escape_html($_ // "") } @{$u}{qw(id username email fullname created_at id)};
}

print "</table></body></html>\n";
$dbh->disconnect;
```

---

## Step 170: โปรแกรมสรุป — Mini Blog CGI

```perl
#!/usr/bin/perl
#
# blog.pl — Mini Blog ด้วย CGI + SQLite
#

use strict;
use warnings;
use CGI;
use DBI;
use POSIX qw(strftime);
use Digest::SHA qw(sha256_hex);

my $cgi = CGI->new;

# =====================
# Setup
# =====================

my $DB = DBI->connect("dbi:SQLite:dbname=/tmp/blog.db", "", "",
    { RaiseError => 1, AutoCommit => 1 }) or die $DBI::errstr;

$DB->do(<<'SQL');
CREATE TABLE IF NOT EXISTS posts (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    title      TEXT NOT NULL,
    content    TEXT NOT NULL,
    author     TEXT DEFAULT 'Anonymous',
    tags       TEXT DEFAULT '',
    created_at DATETIME DEFAULT (datetime('now')),
    views      INTEGER DEFAULT 0
)
SQL

$DB->do(<<'SQL');
CREATE TABLE IF NOT EXISTS comments (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    post_id    INTEGER NOT NULL,
    author     TEXT,
    content    TEXT NOT NULL,
    created_at DATETIME DEFAULT (datetime('now')),
    FOREIGN KEY (post_id) REFERENCES posts(id)
)
SQL

# Seed data
if (!($DB->selectrow_array("SELECT COUNT(*) FROM posts"))[0]) {
    $DB->do("INSERT INTO posts (title, content, author, tags) VALUES
        ('บทนำ Perl CGI', 'Perl CGI เป็นวิธีเก่าแก่ในการสร้างเว็บแบบ dynamic...', 'Admin', 'perl,cgi,web'),
        ('Regular Expressions', 'Regex ใน Perl เป็นหนึ่งในฟีเจอร์ที่ทรงพลังที่สุด...', 'Admin', 'perl,regex'),
        ('การจัดการไฟล์', 'Perl มีความสามารถด้านการจัดการไฟล์ที่ยอดเยี่ยม...', 'Admin', 'perl,files')
    ");
}

# =====================
# Helpers
# =====================

sub esc { my $s = shift // ""; $s =~ s/&/&amp;/g; $s =~ s/</&lt;/g; $s =~ s/>/&gt;/g; $s =~ s/"/&quot;/g; $s }
sub nl2br { my $s = shift; $s =~ s/\n/<br>/g; $s }

sub get_posts {
    my $tag = shift;
    my $sql = "SELECT * FROM posts";
    $sql .= " WHERE tags LIKE ?" if $tag;
    $sql .= " ORDER BY created_at DESC";
    my $sth = $DB->prepare($sql);
    $tag ? $sth->execute("%$tag%") : $sth->execute();
    return @{$sth->fetchall_arrayref({})};
}

sub get_post {
    my $id = shift;
    $DB->do("UPDATE posts SET views = views + 1 WHERE id = ?", undef, $id);
    return $DB->selectrow_hashref("SELECT * FROM posts WHERE id = ?", undef, $id);
}

sub get_comments {
    my $post_id = shift;
    return @{$DB->selectall_arrayref("SELECT * FROM comments WHERE post_id = ? ORDER BY created_at", {Slice=>{}}, $post_id)};
}

sub page_header {
    my $title = shift;
    return <<HTML;
<!DOCTYPE html>
<html lang="th">
<head>
<meta charset="utf-8">
<title>$title — Perl Blog</title>
<style>
* { box-sizing: border-box; }
body { font-family: sans-serif; max-width: 900px; margin: 0 auto; padding: 20px; background: #f5f5f5; }
.card { background: white; padding: 20px; margin: 15px 0; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,.1); }
header { background: #2c5aa0; color: white; padding: 20px; border-radius: 8px; margin-bottom: 20px; }
header a { color: #adf; }
h1, h2, h3 { margin-top: 0; }
.tag { background: #e0e0ff; padding: 2px 8px; border-radius: 12px; font-size: 0.85em; margin: 2px; display: inline-block; }
.meta { color: #666; font-size: 0.9em; }
form input[type=text], form textarea { width: 100%; padding: 8px; margin: 5px 0 10px; border: 1px solid #ddd; border-radius: 4px; }
form button { background: #2c5aa0; color: white; padding: 10px 20px; border: none; border-radius: 4px; cursor: pointer; }
</style>
</head>
<body>
<header>
<h1><a href="?">📝 Perl Blog</a></h1>
<p>บล็อกเกี่ยวกับ Perl Programming</p>
</header>
HTML
}

# =====================
# Routing
# =====================

print $cgi->header(-type => 'text/html', -charset => 'utf-8');

my $action  = $cgi->param('action') // 'list';
my $post_id = $cgi->param('id');
my $tag     = $cgi->param('tag');

if ($action eq 'view' && $post_id) {
    my $post = get_post($post_id);
    
    unless ($post) {
        print page_header("Not Found");
        print "<div class='card'><h2>ไม่พบโพสต์</h2></div>";
        print "</body></html>";
        exit;
    }
    
    print page_header(esc $post->{title});
    print "<div class='card'>";
    printf "<h2>%s</h2>\n", esc $post->{title};
    printf "<p class='meta'>โดย %s | %s | 👁 %d views</p>\n",
        esc($post->{author}), $post->{created_at}, $post->{views};
    
    if ($post->{tags}) {
        print "<p>";
        print "<span class='tag'>" . esc($_) . "</span> " 
            for split /,/, $post->{tags};
        print "</p>\n";
    }
    
    print "<div>" . nl2br(esc $post->{content}) . "</div>\n";
    print "</div>\n";
    
    # Comments
    my @comments = get_comments($post_id);
    printf "<h3>%d ความคิดเห็น</h3>\n", scalar @comments;
    
    for my $c (@comments) {
        print "<div class='card'>";
        printf "<strong>%s</strong> <span class='meta'>%s</span>\n",
            esc($c->{author} // "Anonymous"), $c->{created_at};
        print "<p>" . nl2br(esc $c->{content}) . "</p>\n";
        print "</div>\n";
    }
    
    # Add comment form
    if ($cgi->request_method eq 'POST' && $cgi->param('add_comment')) {
        my $content = $cgi->param('comment_content');
        my $author  = $cgi->param('comment_author') // "Anonymous";
        if ($content) {
            $DB->do("INSERT INTO comments (post_id,author,content) VALUES (?,?,?)",
                undef, $post_id, $author, $content);
            print "<p style='color:green'>เพิ่มความคิดเห็นแล้ว</p>\n";
        }
    }
    
    print <<FORM;
<div class='card'>
<h3>เพิ่มความคิดเห็น</h3>
<form method="post">
<input type="hidden" name="action" value="view">
<input type="hidden" name="id" value="$post_id">
<input type="hidden" name="add_comment" value="1">
<input type="text" name="comment_author" placeholder="ชื่อของคุณ">
<textarea name="comment_content" rows="4" placeholder="ความคิดเห็น..."></textarea>
<button type="submit">โพสต์</button>
</form>
</div>
FORM

    print "<p><a href='?'>← กลับ</a></p>\n";

} elsif ($action eq 'new') {
    print page_header("เขียนโพสต์ใหม่");
    
    if ($cgi->request_method eq 'POST') {
        my $title   = $cgi->param('title');
        my $content = $cgi->param('content');
        my $author  = $cgi->param('author') // "Anonymous";
        my $tags    = $cgi->param('tags');
        
        if ($title && $content) {
            my $id = $DB->do(
                "INSERT INTO posts (title,content,author,tags) VALUES (?,?,?,?)",
                undef, $title, $content, $author, $tags
            );
            print "<p style='color:green'>สร้างโพสต์แล้ว! <a href='?action=view&id=" . $DB->{mysql_insertid} . "'>ดูโพสต์</a></p>\n";
        }
    }
    
    print <<FORM;
<div class='card'>
<h2>เขียนโพสต์ใหม่</h2>
<form method="post">
<input type="hidden" name="action" value="new">
<label>หัวข้อ: <input type="text" name="title" placeholder="หัวข้อโพสต์" required></label>
<label>ผู้เขียน: <input type="text" name="author" placeholder="ชื่อผู้เขียน"></label>
<label>แท็ก: <input type="text" name="tags" placeholder="perl,cgi,web (คั่นด้วย ,)"></label>
<label>เนื้อหา:<textarea name="content" rows="10" placeholder="เนื้อหาโพสต์..." required></textarea></label>
<button type="submit">เผยแพร่</button>
<a href="?" style="margin-left:10px">ยกเลิก</a>
</form>
</div>
FORM

} else {
    # List posts
    print page_header("หน้าหลัก");
    
    print "<p><a href='?action=new' style='background:#2c5aa0;color:white;padding:8px 16px;border-radius:4px;text-decoration:none'>✏️ เขียนโพสต์</a></p>\n";
    
    if ($tag) {
        printf "<p>กรองตามแท็ก: <strong>%s</strong> | <a href='?'>ดูทั้งหมด</a></p>\n", esc $tag;
    }
    
    my @posts = get_posts($tag);
    
    for my $post (@posts) {
        my $preview = substr($post->{content}, 0, 150) . "...";
        print "<div class='card'>";
        printf "<h2><a href='?action=view&id=%d'>%s</a></h2>\n",
            $post->{id}, esc $post->{title};
        printf "<p class='meta'>%s | %s | 👁 %d</p>\n",
            esc($post->{author}), $post->{created_at}, $post->{views};
        print "<p>" . esc($preview) . "</p>\n";
        if ($post->{tags}) {
            print "<p>";
            for my $t (split /,/, $post->{tags}) {
                $t =~ s/^\s+|\s+$//g;
                printf "<a href='?tag=%s' class='tag'>%s</a> ", esc($t), esc($t);
            }
            print "</p>\n";
        }
        printf "<a href='?action=view&id=%d'>อ่านต่อ →</a>\n", $post->{id};
        print "</div>\n";
    }
}

print "</body></html>\n";
$DB->disconnect;
```

---

## สรุป Part 17

ใน Part นี้คุณได้เรียนรู้:
- ✅ CGI พื้นฐาน
- ✅ CGI.pm module
- ✅ HTML generation
- ✅ Form handling (GET, POST)
- ✅ Cookies
- ✅ Session management
- ✅ Authentication
- ✅ File upload
- ✅ Database + CGI
- ✅ Mini Blog application

**ถัดไป: [Part 18 — Advanced CGI และ Web Applications](part_18.md)**
