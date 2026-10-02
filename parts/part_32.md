# Part 32: Web Application Security
## Steps 311-320: ความปลอดภัยของ Web Application — Input Validation, SQL Injection, XSS, CSRF

---

## Step 311: Input Validation

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Scalar::Util qw(looks_like_number);

{
package Validator;

sub new {
    my ($class, %data) = @_;
    return bless { data => \%data, errors => {} }, $class;
}

sub required {
    my ($self, $field) = @_;
    my $val = $self->{data}{$field};
    unless (defined $val && $val =~ /\S/) {
        $self->{errors}{$field} = "$field is required";
    }
    return $self;
}

sub min_length {
    my ($self, $field, $min) = @_;
    my $val = $self->{data}{$field} // "";
    if (length($val) < $min) {
        $self->{errors}{$field} = "$field must be at least $min characters";
    }
    return $self;
}

sub max_length {
    my ($self, $field, $max) = @_;
    my $val = $self->{data}{$field} // "";
    if (length($val) > $max) {
        $self->{errors}{$field} = "$field must be at most $max characters";
    }
    return $self;
}

sub matches {
    my ($self, $field, $regex, $msg) = @_;
    my $val = $self->{data}{$field} // "";
    unless ($val =~ $regex) {
        $self->{errors}{$field} = $msg // "$field is invalid";
    }
    return $self;
}

sub email {
    my ($self, $field) = @_;
    return $self->matches($field, qr/^[^\s\@]+\@[^\s\@]+\.[^\s\@]+$/, "$field must be a valid email");
}

sub numeric {
    my ($self, $field) = @_;
    my $val = $self->{data}{$field} // "";
    unless (looks_like_number($val)) {
        $self->{errors}{$field} = "$field must be numeric";
    }
    return $self;
}

sub range {
    my ($self, $field, $min, $max) = @_;
    my $val = $self->{data}{$field} // 0;
    unless (looks_like_number($val) && $val >= $min && $val <= $max) {
        $self->{errors}{$field} = "$field must be between $min and $max";
    }
    return $self;
}

sub in_list {
    my ($self, $field, @allowed) = @_;
    my $val = $self->{data}{$field} // "";
    unless (grep { $_ eq $val } @allowed) {
        $self->{errors}{$field} = "$field must be one of: " . join(", ", @allowed);
    }
    return $self;
}

sub custom {
    my ($self, $field, $code, $msg) = @_;
    my $val = $self->{data}{$field};
    unless ($code->($val)) {
        $self->{errors}{$field} = $msg;
    }
    return $self;
}

sub valid  { !%{$_[0]->{errors}} }
sub errors { %{$_[0]->{errors}} }
sub error  { $_[0]->{errors}{$_[1]} }
}

package main;

# Test validation
my %form_data = (
    username => "al",
    email    => "not-an-email",
    age      => "abc",
    role     => "superuser",
    password => "short",
);

my $v = Validator->new(%form_data)
    ->required("username")
    ->min_length("username", 3)
    ->required("email")
    ->email("email")
    ->required("age")
    ->numeric("age")
    ->range("age", 1, 120)
    ->in_list("role", "admin", "member", "moderator")
    ->min_length("password", 8);

if ($v->valid) {
    print "Valid!\n";
} else {
    print "Validation errors:\n";
    my %errs = $v->errors;
    printf "  %s: %s\n", $_, $errs{$_} for sort keys %errs;
}

# Good data
my $v2 = Validator->new(
    username => "alice_99",
    email    => "alice\@example.com",
    age      => "25",
    role     => "member",
    password => "securePassword1",
)->required("username")->min_length("username", 3)
 ->required("email")->email("email")
 ->numeric("age")->range("age", 1, 120)
 ->in_list("role", "admin", "member", "moderator")
 ->min_length("password", 8);

printf "\nGood data valid: %s\n", $v2->valid ? "YES" : "NO";
```

---

## Step 312: SQL Injection Prevention

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

my $dbh = DBI->connect("dbi:SQLite::memory:", "", "", {
    RaiseError => 1, AutoCommit => 1 });
$dbh->do("CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT, password TEXT, admin INTEGER DEFAULT 0)");
$dbh->do("INSERT INTO users VALUES (1, 'alice', 'hash1', 1)");
$dbh->do("INSERT INTO users VALUES (2, 'bob',   'hash2', 0)");
$dbh->do("INSERT INTO users VALUES (3, 'carol', 'hash3', 0)");

# VULNERABLE — NEVER DO THIS
sub vulnerable_login {
    my ($username, $password) = @_;
    # Direct string interpolation — SQL injection!
    my $sql = "SELECT * FROM users WHERE username='$username' AND password='$password'";
    printf "  SQL: %s\n", $sql;
    return $dbh->selectrow_hashref($sql);
}

# SAFE — Always use placeholders
sub safe_login {
    my ($username, $password) = @_;
    return $dbh->selectrow_hashref(
        "SELECT * FROM users WHERE username=? AND password=?",
        undef, $username, $password
    );
}

printf "=== SQL Injection Demo ===\n\n";

# Normal login
printf "Normal login (vulnerable):\n";
my $user1 = vulnerable_login("alice", "hash1");
printf "  Result: %s\n\n", $user1 ? "logged_in as $user1->{username}" : "denied";

# SQL injection attempt: ' OR '1'='1
printf "Injection attempt (vulnerable):\n";
my $injected = eval { vulnerable_login("' OR '1'='1", "anything") };
printf "  Result: %s\n\n", $injected ? "BYPASSED as $injected->{username}" : "failed ($@)";

# Same injection with safe method
printf "Injection attempt (safe):\n";
my $safe_result = safe_login("' OR '1'='1", "anything");
printf "  Result: %s\n\n", $safe_result ? "BYPASSED" : "correctly_denied";

# LIKE injection
sub safe_search {
    my ($term) = @_;
    # Escape LIKE wildcards
    $term =~ s/([%_\\])/\\$1/g;
    return @{$dbh->selectall_arrayref(
        "SELECT * FROM users WHERE username LIKE ? ESCAPE '\\'",
        {Slice=>{}}, "%$term%"
    )};
}

printf "Safe LIKE search for 'ali':\n";
my @found = safe_search("ali");
printf "  Found: %s\n", $_->{username} for @found;

printf "\nSafe LIKE with wildcards in input '%_alice%':\n";
my @found2 = safe_search("%_alice%");
printf "  Found: %d results (correctly 0 because escaped)\n", scalar @found2;

# Whitelist for dynamic ORDER BY (can't parameterize column names)
my %allowed_sort_cols = map { $_ => 1 } qw(id username admin);

sub safe_order_by {
    my ($col, $dir) = @_;
    die "Invalid sort column\n" unless $allowed_sort_cols{$col};
    $dir = uc($dir//"ASC");
    die "Invalid direction\n"   unless $dir eq "ASC" || $dir eq "DESC";
    return $dbh->selectall_arrayref("SELECT * FROM users ORDER BY $col $dir", {Slice=>{}});
}

printf "\nSafe dynamic ORDER BY (username ASC):\n";
my $sorted = eval { safe_order_by("username", "asc") };
printf "  %s\n", $_->{username} for @$sorted;

eval { safe_order_by("DROP TABLE users; --", "asc") };
printf "Malicious column: correctly rejected (%s)\n", $@ ? "yes" : "no";
```

---

## Step 313: XSS Prevention

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package XSS;

# HTML encode — stops XSS in HTML context
sub encode_html {
    my $s = shift // "";
    $s =~ s/&/&amp;/g;
    $s =~ s/</&lt;/g;
    $s =~ s/>/&gt;/g;
    $s =~ s/"/&quot;/g;
    $s =~ s/'/&#x27;/g;
    $s =~ s{/}{&#x2F;}g;
    return $s;
}

# JS encode — for values in <script> context
sub encode_js {
    my $s = shift // "";
    $s =~ s/\\/\\\\/g;
    $s =~ s/"/\\"/g;
    $s =~ s/'/\\'/g;
    $s =~ s/\n/\\n/g;
    $s =~ s/\r/\\r/g;
    $s =~ s/\t/\\t/g;
    $s =~ s/</\\x3C/g;
    $s =~ s/>/\\x3E/g;
    return $s;
}

# URL encode — for values in URL context
sub encode_url {
    my $s = shift // "";
    $s =~ s/([^A-Za-z0-9\-_.~])/sprintf "%%%02X", ord($1)/ge;
    return $s;
}

# Strip HTML tags — for plain text extraction
sub strip_tags {
    my $s = shift // "";
    $s =~ s/<[^>]+>//g;
    $s =~ s/&amp;/&/g;
    $s =~ s/&lt;/</g;
    $s =~ s/&gt;/>/g;
    $s =~ s/&quot;/"/g;
    return $s;
}

# Content Security Policy header
sub csp_header {
    return "Content-Security-Policy: " . join("; ",
        "default-src 'self'",
        "script-src 'self'",
        "style-src 'self' 'unsafe-inline'",
        "img-src 'self' data:",
        "font-src 'self'",
        "connect-src 'self'",
        "frame-ancestors 'none'",
    );
}
}

package main;

my @attack_strings = (
    q{<script>alert('xss')</script>},
    q{"><img src=x onerror=alert(1)>},
    q{javascript:alert(1)},
    q{' OR '1'='1},
    q{<svg onload=alert(1)>},
);

printf "=== XSS Prevention ===\n\n";

for my $attack (@attack_strings) {
    printf "Input:    %s\n", $attack;
    printf "HTML enc: %s\n", XSS::encode_html($attack);
    printf "JS enc:   %s\n", XSS::encode_js($attack);
    printf "URL enc:  %s\n", XSS::encode_url($attack);
    print  "\n";
}

# HTML generation example
sub render_comment {
    my ($author, $body) = @_;
    return sprintf(
        q{<div class="comment"><strong>%s</strong><p>%s</p></div>},
        XSS::encode_html($author),
        XSS::encode_html($body)
    );
}

printf "Safe comment HTML:\n%s\n\n",
    render_comment(
        '<script>evil()</script>',
        'Hello <b>world</b>! <img src=x onerror=alert(1)>'
    );

printf "CSP Header:\n%s\n", XSS::csp_header();
```

---

## Step 314: CSRF Protection

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(sha256_hex hmac_sha256_hex);

{
package CSRF;

my $SECRET = "csrf_secret_key_change_in_production";

sub generate_token {
    my ($class, $session_id) = @_;
    my $timestamp = time();
    my $random    = sprintf "%08x", rand(0xFFFFFFFF);
    my $data      = "$session_id:$timestamp:$random";
    my $hmac      = hmac_sha256_hex($data, $SECRET);
    return "$data:$hmac";
}

sub validate_token {
    my ($class, $token, $session_id, $max_age) = @_;
    $max_age //= 3600;
    
    return 0 unless $token;
    
    my ($sid, $ts, $rand, $hmac) = split /:/, $token, 4;
    
    # Verify all parts present
    return 0 unless $sid && $ts && $rand && $hmac;
    
    # Verify session matches
    return 0 unless $sid eq $session_id;
    
    # Verify not expired
    return 0 if (time() - $ts) > $max_age;
    
    # Verify HMAC
    my $data     = "$sid:$ts:$rand";
    my $expected = hmac_sha256_hex($data, $SECRET);
    
    # Constant-time comparison
    return _constant_compare($hmac, $expected);
}

sub _constant_compare {
    my ($a, $b) = @_;
    return 0 if length($a) != length($b);
    my $diff = 0;
    $diff |= ord(substr($a,$_,1)) ^ ord(substr($b,$_,1)) for 0..length($a)-1;
    return $diff == 0 ? 1 : 0;
}

# Render CSRF hidden input
sub hidden_field {
    my ($class, $session_id) = @_;
    my $token = $class->generate_token($session_id);
    return sprintf '<input type="hidden" name="csrf_token" value="%s">', $token;
}
}

package main;

printf "=== CSRF Protection ===\n\n";

my $session_id = "user_session_abc123";

# Generate token
my $token = CSRF->generate_token($session_id);
printf "Generated token: %.40s...\n", $token;

# Valid check
my $valid = CSRF->validate_token($token, $session_id);
printf "Valid token: %s\n", $valid ? "YES" : "NO";

# Wrong session
my $wrong_session = CSRF->validate_token($token, "other_session");
printf "Wrong session: %s\n", $wrong_session ? "PASSED (BAD)" : "correctly_rejected";

# Tampered token
my $tampered = $token;
$tampered =~ s/a/b/;
my $tampered_check = CSRF->validate_token($tampered, $session_id);
printf "Tampered token: %s\n", $tampered_check ? "PASSED (BAD)" : "correctly_rejected";

# HTML form
printf "\nForm HTML:\n";
printf "<form method='POST' action='/change-password'>\n";
printf "  %s\n", CSRF->hidden_field($session_id);
printf "  <input type='password' name='new_password'>\n";
printf "  <button>Change</button>\n</form>\n";
```

---

## Step 315: Password Hashing

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(sha256_hex sha512_hex);
use MIME::Base64 qw(encode_base64url decode_base64url);

{
package Password;

# Bcrypt-like: PBKDF2 with SHA-512, multiple iterations
sub hash_password {
    my ($class, $password, $iterations, $salt) = @_;
    $iterations //= 10000;
    $salt       //= do {
        my $raw = "";
        for (1..16) { $raw .= chr(int(rand(256))) }
        encode_base64url($raw)
    };
    
    # PBKDF2-like iteration
    my $hash = $password;
    for (1..$iterations) {
        $hash = sha512_hex($hash . $salt . $_);
    }
    
    return "$iterations\$$salt\$$hash";
}

sub verify {
    my ($class, $password, $stored) = @_;
    my ($iterations, $salt, $expected) = split /\$/, $stored;
    my $computed = $class->hash_password($password, $iterations, $salt);
    my (undef, undef, $computed_hash) = split /\$/, $computed;
    
    # Constant-time comparison
    return _const_cmp($expected, $computed_hash);
}

sub _const_cmp {
    my ($a, $b) = @_;
    return 0 if length($a) != length($b);
    my $diff = 0;
    $diff |= ord(substr($a,$_,1)) ^ ord(substr($b,$_,1)) for 0..length($a)-1;
    return $diff == 0;
}

sub needs_rehash {
    my ($class, $stored, $min_iterations) = @_;
    my ($iters) = split /\$/, $stored;
    return $iters < $min_iterations;
}
}

{
package PasswordPolicy;

sub check {
    my ($class, $password) = @_;
    my @errors;
    push @errors, "at least 8 characters"   if length($password) < 8;
    push @errors, "at least one uppercase"   unless $password =~ /[A-Z]/;
    push @errors, "at least one lowercase"   unless $password =~ /[a-z]/;
    push @errors, "at least one digit"       unless $password =~ /\d/;
    push @errors, "at least one special char"unless $password =~ /[^A-Za-z0-9]/;
    
    return @errors ? (0, \@errors) : (1, []);
}

sub strength {
    my ($class, $password) = @_;
    my $score = 0;
    $score += length($password) >= 8  ? 1 : 0;
    $score += length($password) >= 12 ? 1 : 0;
    $score += length($password) >= 16 ? 1 : 0;
    $score += $password =~ /[A-Z]/    ? 1 : 0;
    $score += $password =~ /[a-z]/    ? 1 : 0;
    $score += $password =~ /\d/       ? 1 : 0;
    $score += $password =~ /[^A-Za-z0-9]/ ? 2 : 0;
    
    return $score <= 2 ? "weak" : $score <= 5 ? "medium" : "strong";
}
}

package main;

printf "=== Password Hashing ===\n\n";

my $password = "MySecureP\@ss1";

# Hash with 1000 iterations (fast for demo; use 10000+ in production)
my $t0   = time;
my $hash = Password->hash_password($password, 1000);
printf "Hashed (1000 iter): %s...\n", substr($hash, 0, 40);

# Verify
my $ok = Password->verify($password, $hash);
printf "Verify correct:     %s\n", $ok ? "OK" : "FAIL";

my $bad = Password->verify("wrong_password", $hash);
printf "Verify wrong:       %s\n", $bad ? "FAIL (bypass!)" : "correctly_denied";

# Needs rehash
printf "Needs rehash (min 5000): %s\n", Password->needs_rehash($hash, 5000) ? "yes" : "no";

# Password policy
printf "\n=== Password Policy ===\n";
my @test_passwords = (
    "short",
    "alllowercase",
    "ALLUPPERCASE",
    "NoNumbers!",
    "N0Spec1alCh",
    "Str0ng\@Pass!",
);

for my $pw (@test_passwords) {
    my ($valid, $errors) = PasswordPolicy->check($pw);
    my $strength = PasswordPolicy->strength($pw);
    printf "%-20s  %-6s  %-6s  %s\n",
        $pw, $valid ? "PASS" : "FAIL", $strength,
        $valid ? "" : join(", ", @$errors);
}
```

---

## Step 316: Rate Limiting and Brute-Force Protection

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Security::RateLimit;

my %attempts;   # key => [timestamps]
my %locked;     # key => lock_until

sub check_allowed {
    my ($class, $key, %opts) = @_;
    my $window   = $opts{window}   // 300;   # 5 minutes
    my $max_attempts = $opts{max}  // 5;
    my $lockout  = $opts{lockout}  // 900;   # 15 minutes
    
    my $now = time();
    
    # Check if locked
    if (my $until = $locked{$key}) {
        if ($now < $until) {
            return (0, "Account locked. Try again in " . ($until - $now) . "s");
        }
        delete $locked{$key};
        delete $attempts{$key};
    }
    
    # Clean old attempts
    $attempts{$key} //= [];
    @{$attempts{$key}} = grep { $now - $_ < $window } @{$attempts{$key}};
    
    my $count = scalar @{$attempts{$key}};
    
    if ($count >= $max_attempts) {
        $locked{$key} = $now + $lockout;
        return (0, "Too many attempts. Account locked for ${lockout}s");
    }
    
    return (1, undef, $max_attempts - $count - 1);  # remaining attempts
}

sub record_attempt {
    my ($class, $key) = @_;
    push @{$attempts{$key}}, time();
}

sub record_success {
    my ($class, $key) = @_;
    delete $attempts{$key};
    delete $locked{$key};
}

sub attempt_count {
    my ($class, $key, $window) = @_;
    $window //= 300;
    my $now = time();
    return scalar grep { $now - $_ < $window } @{$attempts{$key}//[]};
}
}

package main;

printf "=== Rate Limiting / Brute-Force Protection ===\n\n";

sub simulate_login {
    my ($username, $password) = @_;
    my $key = "login:$username";
    
    my ($allowed, $err, $remaining) = Security::RateLimit->check_allowed($key,
        max => 3, window => 60, lockout => 120);
    
    unless ($allowed) {
        printf "  [BLOCKED] $err\n";
        return 0;
    }
    
    # Simulate auth check
    my $success = ($username eq "alice" && $password eq "correct");
    
    if ($success) {
        Security::RateLimit->record_success($key);
        printf "  [SUCCESS] Login OK\n";
        return 1;
    } else {
        Security::RateLimit->record_attempt($key);
        my $count = Security::RateLimit->attempt_count($key);
        printf "  [FAILED] Wrong password (attempt %d/3)\n", $count;
        return 0;
    }
}

printf "Simulating login attempts for alice:\n";
simulate_login("alice", "wrong1");
simulate_login("alice", "wrong2");
simulate_login("alice", "wrong3");
simulate_login("alice", "wrong4");  # Should be blocked
simulate_login("alice", "correct"); # Still blocked
```

---

## Step 317: HTTPS and Secure Headers

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Security::Headers;

# All security headers for production
sub all_headers {
    return (
        # HTTPS enforcement
        "Strict-Transport-Security" => "max-age=31536000; includeSubDomains; preload",
        
        # XSS protection
        "X-XSS-Protection"        => "1; mode=block",
        "X-Content-Type-Options"  => "nosniff",
        
        # Clickjacking prevention
        "X-Frame-Options"         => "DENY",
        
        # CSP
        "Content-Security-Policy" => join("; ",
            "default-src 'self'",
            "script-src 'self'",
            "style-src 'self' 'unsafe-inline'",
            "img-src 'self' data: https:",
            "font-src 'self'",
            "connect-src 'self'",
            "frame-ancestors 'none'",
            "base-uri 'self'",
            "form-action 'self'",
        ),
        
        # Referrer control
        "Referrer-Policy"         => "strict-origin-when-cross-origin",
        
        # Permissions Policy
        "Permissions-Policy"      => join(", ",
            "geolocation=()",
            "camera=()",
            "microphone=()",
            "payment=()",
        ),
        
        # Remove server info
        "Server"                  => "webserver",
    );
}

sub cgi_headers {
    my %headers = all_headers();
    my $out = "";
    while (my ($name, $val) = each %headers) {
        $out .= "$name: $val\n";
    }
    return $out;
}

sub cookie_attributes {
    my ($class, %opts) = @_;
    my @parts;
    push @parts, "HttpOnly"                           unless $opts{allow_js};
    push @parts, "Secure"                             unless $opts{insecure};
    push @parts, "SameSite=" . ($opts{samesite}//"Strict");
    push @parts, "Path=" . ($opts{path}//"/");
    push @parts, "Max-Age=" . $opts{max_age}          if $opts{max_age};
    push @parts, "Domain=" . $opts{domain}            if $opts{domain};
    return join("; ", @parts);
}
}

package main;

printf "=== Secure HTTP Headers ===\n\n";

my %headers = Security::Headers::all_headers();
while (my ($name, $val) = each %headers) {
    printf "%-35s %s\n", $name . ":", $val;
}

printf "\n=== Secure Cookie Attributes ===\n";
printf "Session cookie: %s\n",  Security::Headers->cookie_attributes(max_age => 86400);
printf "API cookie:     %s\n",  Security::Headers->cookie_attributes(max_age => 3600, samesite => "Lax");
printf "Tracking:       %s\n",  Security::Headers->cookie_attributes(allow_js => 1, samesite => "None", insecure => 0);
```

---

## Step 318: Secure File Uploads

```perl
#!/usr/bin/perl
use strict;
use warnings;
use File::Basename qw(basename);
use Digest::SHA qw(sha256_hex);

{
package Upload::Validator;

my %ALLOWED_TYPES = (
    "image/jpeg" => { ext => [qw(jpg jpeg)], magic => "\xFF\xD8\xFF" },
    "image/png"  => { ext => [qw(png)],      magic => "\x89PNG" },
    "image/gif"  => { ext => [qw(gif)],      magic => "GIF8" },
    "application/pdf" => { ext => [qw(pdf)], magic => "%PDF" },
);

sub validate {
    my ($class, %args) = @_;
    my @errors;
    
    my $filename  = $args{filename}  // "";
    my $content   = $args{content}   // "";
    my $mime_type = $args{mime_type} // "";
    my $max_size  = $args{max_size}  // 5 * 1024 * 1024;  # 5MB
    
    # Check size
    push @errors, "File too large (max " . int($max_size/1024/1024) . "MB)"
        if length($content) > $max_size;
    
    # Sanitize filename
    my $safe_name = _safe_filename($filename);
    push @errors, "Invalid filename" unless $safe_name;
    
    # Check MIME type is allowed
    unless ($ALLOWED_TYPES{$mime_type}) {
        push @errors, "File type '$mime_type' not allowed";
        return { valid => 0, errors => \@errors };
    }
    
    # Verify magic bytes (don't trust MIME from client)
    my $magic_ok = 0;
    if (my $info = $ALLOWED_TYPES{$mime_type}) {
        my $file_start = substr($content, 0, 8);
        $magic_ok = index($file_start, $info->{magic}) == 0;
    }
    push @errors, "File content doesn't match claimed type" unless $magic_ok;
    
    # Verify extension matches MIME type
    my ($ext) = $safe_name =~ /\.([^.]+)$/;
    $ext = lc($ext // "");
    my $ext_ok = grep { $_ eq $ext } @{ $ALLOWED_TYPES{$mime_type}{ext} };
    push @errors, "File extension doesn't match MIME type" unless $ext_ok;
    
    # Generate safe storage name
    my $storage_name = sha256_hex(time() . $$ . rand()) . ".$ext";
    
    return {
        valid        => scalar(@errors) == 0 ? 1 : 0,
        errors       => \@errors,
        safe_name    => $safe_name,
        storage_name => $storage_name,
        size         => length($content),
    };
}

sub _safe_filename {
    my $name = shift;
    $name = basename($name);
    $name =~ s/[^A-Za-z0-9.\-_]/_/g;
    $name =~ s/\.{2,}/./g;
    $name =~ s/^\.//;
    return length($name) >= 3 && length($name) <= 255 ? $name : undef;
}
}

package main;

printf "=== Secure File Upload Validation ===\n\n";

# Fake PNG content (PNG magic bytes)
my $fake_png = "\x89PNG\r\n\x1a\n" . ("A" x 100);

# Fake JPEG
my $fake_jpg = "\xFF\xD8\xFF\xE0" . ("B" x 100);

my @upload_tests = (
    {
        desc      => "Valid PNG",
        filename  => "photo.png",
        mime_type => "image/png",
        content   => $fake_png,
    },
    {
        desc      => "Valid JPEG",
        filename  => "image.jpg",
        mime_type => "image/jpeg",
        content   => $fake_jpg,
    },
    {
        desc      => "Path traversal",
        filename  => "../../etc/passwd",
        mime_type => "image/png",
        content   => $fake_png,
    },
    {
        desc      => "PHP disguised as PNG",
        filename  => "evil.php.png",
        mime_type => "image/png",
        content   => "<?php system(\$_GET['cmd']); ?>",
    },
    {
        desc      => "Wrong extension",
        filename  => "document.exe",
        mime_type => "image/png",
        content   => $fake_png,
    },
    {
        desc      => "Disallowed type",
        filename  => "virus.exe",
        mime_type => "application/x-executable",
        content   => "MZ" . "A" x 100,
    },
);

for my $test (@upload_tests) {
    my $result = Upload::Validator->validate(%$test);
    printf "%s: %s\n", $test->{desc}, $result->{valid} ? "ALLOWED" : "REJECTED";
    printf "  %s\n", $_ for @{$result->{errors}};
    printf "  Storage: %s\n", $result->{storage_name} if $result->{valid};
}
```

---

## Step 319: Session Security

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(sha256_hex);

{
package Session::Store;

my %sessions;

sub create {
    my ($class, %data) = @_;
    my $id      = sha256_hex(rand() . time() . $$ . rand());
    my $expires = time() + ($data{ttl} // 3600);
    
    $sessions{$id} = {
        %data,
        id         => $id,
        created_at => time(),
        expires_at => $expires,
        last_seen  => time(),
        ip         => $data{ip} // "127.0.0.1",
        ua         => $data{ua} // "",
    };
    delete $sessions{$id}{ttl};
    
    return $id;
}

sub get {
    my ($class, $id) = @_;
    return undef unless $id && $sessions{$id};
    
    my $s = $sessions{$id};
    
    # Check expiry
    if (time() > $s->{expires_at}) {
        delete $sessions{$id};
        return undef;
    }
    
    # Update last_seen
    $s->{last_seen} = time();
    return $s;
}

sub validate_context {
    my ($class, $id, $ip, $ua) = @_;
    my $s = $class->get($id) or return (0, "Session not found");
    
    # IP binding (optional but more secure)
    # return (0, "IP mismatch") if $s->{ip} ne $ip;
    
    return (1, undef, $s);
}

sub destroy   { delete $sessions{$_[1]} }
sub rotate_id {
    my ($class, $old_id) = @_;
    my $s = $sessions{$old_id} or return undef;
    my $new_id = sha256_hex(rand() . time() . $$);
    $sessions{$new_id} = { %$s, id => $new_id };
    delete $sessions{$old_id};
    return $new_id;
}

sub gc {
    my ($class) = @_;
    my $now     = time();
    my $count   = 0;
    for my $id (keys %sessions) {
        if ($now > $sessions{$id}{expires_at}) {
            delete $sessions{$id};
            $count++;
        }
    }
    return $count;
}
}

package main;

printf "=== Session Security ===\n\n";

# Create session
my $sid = Session::Store->create(
    user_id  => 42,
    username => "alice",
    role     => "admin",
    ip       => "192.168.1.1",
    ttl      => 3600,
);

printf "Session created: %.16s...\n", $sid;

# Get session
my $s = Session::Store->get($sid);
printf "Session valid: %s\n", $s ? "yes" : "no";
printf "  user_id: %d\n",  $s->{user_id};
printf "  username: %s\n", $s->{username};

# Validate context
my ($ok, $err) = Session::Store->validate_context($sid, "192.168.1.1", "Mozilla/5.0");
printf "Context valid: %s\n", $ok ? "yes" : "no [$err]";

# Session rotation (do this after login)
my $new_sid = Session::Store->rotate_id($sid);
printf "\nRotated session: %.16s...\n", $new_sid;
printf "Old session still valid: %s\n",
    Session::Store->get($sid) ? "yes (BAD)" : "no (correct)";
printf "New session valid: %s\n",
    Session::Store->get($new_sid) ? "yes (correct)" : "no (BAD)";

# Destroy (logout)
Session::Store->destroy($new_sid);
printf "\nAfter destroy: %s\n",
    Session::Store->get($new_sid) ? "still_valid (BAD)" : "destroyed (correct)";
```

---

## Step 320: Capstone — Secure CGI Application

```perl
#!/usr/bin/perl
# secure_app.pl — Security-hardened CGI application demo
use strict;
use warnings;
use Digest::SHA qw(sha256_hex hmac_sha256_hex);

# ---- CGI simulation ----
{
package MockCGI;
sub new       { bless {%{$_[1]}}, $_[0] }
sub param     { $_[0]->{params}{$_[1]} }
sub remote_ip { $_[0]->{ip} // "127.0.0.1" }
sub method    { $_[0]->{method} // "GET" }
sub path_info { $_[0]->{path} // "/" }
sub cookie    { $_[0]->{cookies}{$_[1]} }
}

# ---- Secure Request Handler ----
{
package SecureApp;

my $SECRET = "app_secret_key_change_this";

sub handle_request {
    my ($class, $cgi) = @_;
    my %response = (status => 200, headers => {}, body => "");
    
    # Add security headers to every response
    %{$response{headers}} = (
        "X-Content-Type-Options"  => "nosniff",
        "X-Frame-Options"         => "DENY",
        "X-XSS-Protection"        => "1; mode=block",
        "Referrer-Policy"         => "strict-origin-when-cross-origin",
    );
    
    my $path   = $cgi->path_info;
    my $method = $cgi->method;
    
    # Route
    if ($path eq "/login" && $method eq "POST") {
        return $class->handle_login($cgi, \%response);
    } elsif ($path eq "/api/data" && $method eq "GET") {
        return $class->handle_api($cgi, \%response);
    } elsif ($path eq "/submit" && $method eq "POST") {
        return $class->handle_form($cgi, \%response);
    }
    
    $response{status} = 404;
    $response{body}   = "Not found";
    return %response;
}

sub handle_login {
    my ($class, $cgi, $res) = @_;
    
    my $username = $cgi->param("username") // "";
    my $password = $cgi->param("password") // "";
    
    # Sanitize inputs
    $username =~ s/[^A-Za-z0-9_]//g;
    
    # Rate limit check
    my ($allowed) = Security::RateLimit->check_allowed("login:$username", max => 5, window => 300);
    unless ($allowed) {
        $res->{status} = 429;
        $res->{body}   = "Too many attempts";
        return %$res;
    }
    
    # Validate
    my $v = Validator->new(username => $username, password => $password)
        ->required("username")->required("password")
        ->min_length("username", 3)->min_length("password", 4);
    
    unless ($v->valid) {
        $res->{status} = 400;
        $res->{body}   = join(", ", values %{{$v->errors}});
        return %$res;
    }
    
    # Auth
    my $valid_credentials = ($username eq "alice" && sha256_hex("tms_salt_$password") eq sha256_hex("tms_salt_password123"));
    
    if ($valid_credentials) {
        Security::RateLimit->record_success("login:$username");
        my $token = Session::Store->create(username => $username, user_id => 1, ttl => 3600);
        $res->{headers}{"Set-Cookie"} = "session=$token; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=3600";
        $res->{body}   = "Login OK";
    } else {
        Security::RateLimit->record_attempt("login:$username");
        $res->{status} = 401;
        $res->{body}   = "Invalid credentials";
    }
    
    return %$res;
}

sub handle_api {
    my ($class, $cgi, $res) = @_;
    
    # Auth via cookie
    my $token   = $cgi->cookie("session") // "";
    my $session = Session::Store->get($token);
    
    unless ($session) {
        $res->{status} = 401;
        $res->{body}   = '{"error":"Unauthorized"}';
        return %$res;
    }
    
    $res->{headers}{"Content-Type"} = "application/json";
    $res->{body} = '{"data":"secret","user":"' . $session->{username} . '"}';
    return %$res;
}

sub handle_form {
    my ($class, $cgi, $res) = @_;
    
    # CSRF check
    my $token   = $cgi->cookie("session") // "";
    my $csrf    = $cgi->param("csrf_token") // "";
    
    my $valid_csrf = CSRF->validate_token($csrf, $token);
    unless ($valid_csrf) {
        $res->{status} = 403;
        $res->{body}   = "CSRF validation failed";
        return %$res;
    }
    
    # Input sanitization
    my $comment = $cgi->param("comment") // "";
    my $safe    = XSS::encode_html($comment);
    
    $res->{body} = "Saved: $safe";
    return %$res;
}
}

package main;

printf "=== Secure CGI App Demo ===\n\n";

# Test 1: Valid login
my %r1 = SecureApp->handle_request(MockCGI->new({
    method => "POST", path => "/login",
    params => { username => "alice", password => "password123" }
}));
printf "Login alice: %d — %s\n", $r1{status}, $r1{body};

# Test 2: Invalid login
my %r2 = SecureApp->handle_request(MockCGI->new({
    method => "POST", path => "/login",
    params => { username => "alice", password => "wrong" }
}));
printf "Login wrong: %d — %s\n", $r2{status}, $r2{body};

# Test 3: API without auth
my %r3 = SecureApp->handle_request(MockCGI->new({
    method => "GET", path => "/api/data",
    cookies => {}
}));
printf "API no auth: %d — %s\n", $r3{status}, $r3{body};

# Get session from login
my $cookie_hdr = $r1{headers}{"Set-Cookie"} // "";
my ($session_token) = $cookie_hdr =~ /session=([^;]+)/;
printf "\nSession token: %.16s...\n", $session_token // "none";

# Test 4: API with auth
if ($session_token) {
    my %r4 = SecureApp->handle_request(MockCGI->new({
        method  => "GET",
        path    => "/api/data",
        cookies => { session => $session_token },
    }));
    printf "API with auth: %d — %s\n", $r4{status}, $r4{body};
}

# Test 5: Form with CSRF
if ($session_token) {
    my $csrf_token = CSRF->generate_token($session_token);
    my %r5 = SecureApp->handle_request(MockCGI->new({
        method  => "POST",
        path    => "/submit",
        cookies => { session => $session_token },
        params  => {
            csrf_token => $csrf_token,
            comment    => "<script>alert('xss')</script>",
        },
    }));
    printf "Form submit: %d — %s\n", $r5{status}, $r5{body};
}

printf "\nSecurity headers on response:\n";
my %check_r = SecureApp->handle_request(MockCGI->new({ method => "GET", path => "/" }));
printf "  %s: %s\n", $_, $check_r{headers}{$_} for sort keys %{$check_r{headers}};
```

---

## สรุป Part 32 — Web Application Security

ใน Part นี้คุณได้เรียนรู้:

### Security Concepts
- **Input Validation** — Required/length/email/numeric/range/in_list/custom
- **SQL Injection Prevention** — Parameterized queries, whitelist for column names
- **XSS Prevention** — encode_html/encode_js/encode_url, CSP headers
- **CSRF Protection** — HMAC-signed tokens, constant-time comparison
- **Password Hashing** — Iterative SHA-512 (PBKDF2-like), salted, policy checking
- **Rate Limiting** — Sliding window, lockout on excessive attempts
- **Secure Headers** — HSTS/CSP/X-Frame-Options/Permissions-Policy
- **Secure File Uploads** — Magic byte verification, extension whitelist, storage name randomization
- **Session Security** — HttpOnly/Secure/SameSite cookies, session rotation, GC
- **Capstone** — Complete secure CGI request lifecycle

**ถัดไป: [Part 33 — CGI Web Development](part_33.md)**
