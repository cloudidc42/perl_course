# Part 40: Authentication Systems & OAuth2
## Steps 391-400: Security, Sessions, JWT, OAuth2

---

## Step 391: Session Management Deep Dive

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(sha256_hex hmac_sha256_hex);
use MIME::Base64 qw(encode_base64url decode_base64url);

{
package Session::Store;

my %store;

sub new {
    my ($class, %opts) = @_;
    return bless {
        ttl        => $opts{ttl}        // 3600,
        max_size   => $opts{max_size}   // 1000,
        secret     => $opts{secret}     // "change-me-in-prod",
        prefix     => $opts{prefix}     // "sess_",
    }, $class;
}

sub create {
    my ($self, %data) = @_;
    my $id  = $self->{prefix} . _random_hex(32);
    my $now = time();
    $store{$id} = {
        id         => $id,
        data       => \%data,
        created_at => $now,
        updated_at => $now,
        expires_at => $now + $self->{ttl},
    };
    return $id;
}

sub get {
    my ($self, $id) = @_;
    my $sess = $store{$id} or return undef;
    return undef if $sess->{expires_at} < time();
    return $sess;
}

sub update {
    my ($self, $id, %data) = @_;
    my $sess = $self->get($id) or return 0;
    $sess->{data}{$_} = $data{$_} for keys %data;
    $sess->{updated_at} = time();
    return 1;
}

sub refresh {
    my ($self, $id) = @_;
    my $sess = $self->get($id) or return 0;
    $sess->{expires_at} = time() + $self->{ttl};
    $sess->{updated_at} = time();
    return 1;
}

sub destroy {
    my ($self, $id) = @_;
    return delete $store{$id} ? 1 : 0;
}

sub sign {
    my ($self, $id) = @_;
    my $sig = substr(hmac_sha256_hex($id, $self->{secret}), 0, 16);
    return "$id.$sig";
}

sub verify_signed {
    my ($self, $signed) = @_;
    my ($id, $sig) = split /\./, $signed, 2;
    return undef unless $id && $sig;
    my $expected = substr(hmac_sha256_hex($id, $self->{secret}), 0, 16);
    return $sig eq $expected ? $id : undef;
}

sub cleanup {
    my $self = shift;
    my $now  = time();
    my $removed = 0;
    for my $id (keys %store) {
        if ($store{$id}{expires_at} < $now) {
            delete $store{$id};
            $removed++;
        }
    }
    return $removed;
}

sub count { scalar keys %store }

sub _random_hex { join "", map { sprintf "%02x", int(rand(256)) } 1..$_[0] }
}

package main;

printf "=== Session Management ===\n\n";

my $sessions = Session::Store->new(ttl => 3600, secret => "super-secret-key");

# Create sessions
my $sid1 = $sessions->create(user_id => 1, username => "alice", role => "admin");
my $sid2 = $sessions->create(user_id => 2, username => "bob",   role => "user");
my $sid3 = $sessions->create(user_id => 3, username => "carol", role => "user");

printf "Created sessions:\n";
printf "  Alice: %s\n", substr($sid1, 0, 20) . "...";
printf "  Bob:   %s\n", substr($sid2, 0, 20) . "...";
printf "  Carol: %s\n", substr($sid3, 0, 20) . "...";

# Sign for cookie
my $signed1 = $sessions->sign($sid1);
printf "\nSigned cookie (Alice): %s\n", substr($signed1, 0, 40) . "...";

# Verify
my $verified_id = $sessions->verify_signed($signed1);
printf "Verified session id: %s\n", $verified_id ? "OK" : "FAILED";

my $tampered = $signed1 . "x";
my $tampered_result = $sessions->verify_signed($tampered);
printf "Tampered cookie: %s\n", $tampered_result ? "PASS (bad!)" : "Rejected OK";

# Read session
my $sess = $sessions->get($sid1);
printf "\nAlice session data:\n";
printf "  user_id=%d username=%s role=%s\n",
    $sess->{data}{user_id}, $sess->{data}{username}, $sess->{data}{role};

# Update
$sessions->update($sid1, last_login => time(), ip => "192.168.1.1");
my $updated = $sessions->get($sid1);
printf "\nUpdated: last_login=%d ip=%s\n",
    $updated->{data}{last_login}, $updated->{data}{ip};

# Stats
printf "\nActive sessions: %d\n", $sessions->count;

# Destroy
$sessions->destroy($sid3);
printf "After destroying Carol: %d sessions\n", $sessions->count;
```

---

## Step 392: JWT — JSON Web Tokens

```perl
#!/usr/bin/perl
use strict;
use warnings;
use MIME::Base64 qw(encode_base64url decode_base64url);
use Digest::SHA qw(hmac_sha256);
use JSON::PP;

{
package JWT;

my $JSON = JSON::PP->new->utf8->canonical;

sub new {
    my ($class, %opts) = @_;
    return bless {
        secret    => $opts{secret}    // die("secret required"),
        algorithm => $opts{algorithm} // "HS256",
        issuer    => $opts{issuer}    // "perl_app",
        ttl       => $opts{ttl}       // 3600,
    }, $class;
}

sub generate {
    my ($self, %claims) = @_;
    my $now = time();
    
    my $header = { typ => "JWT", alg => $self->{algorithm} };
    my $payload = {
        iss => $self->{issuer},
        iat => $now,
        exp => $now + ($claims{ttl} // $self->{ttl}),
        %claims,
    };
    delete $payload{ttl};
    
    my $h = _b64u($JSON->encode($header));
    my $p = _b64u($JSON->encode($payload));
    my $s = _b64u(hmac_sha256("$h.$p", $self->{secret}));
    
    return "$h.$p.$s";
}

sub verify {
    my ($self, $token) = @_;
    
    my @parts = split /\./, $token;
    return (undef, "invalid_format") unless @parts == 3;
    
    my ($h, $p, $s) = @parts;
    
    # Verify signature
    my $expected = _b64u(hmac_sha256("$h.$p", $self->{secret}));
    return (undef, "invalid_signature") unless $expected eq $s;
    
    # Decode payload
    my $payload = eval { $JSON->decode(_b64d($p)) };
    return (undef, "invalid_payload") if $@;
    
    # Check expiry
    if ($payload->{exp} && $payload->{exp} < time()) {
        return (undef, "token_expired");
    }
    
    # Check issuer
    if ($self->{issuer} && $payload->{iss} && $payload->{iss} ne $self->{issuer}) {
        return (undef, "invalid_issuer");
    }
    
    return ($payload, undef);
}

sub decode_unverified {
    my ($self, $token) = @_;
    my @parts = split /\./, $token;
    return undef unless @parts == 3;
    return eval { $JSON->decode(_b64d($parts[1])) };
}

sub refresh {
    my ($self, $token, %extra) = @_;
    my ($claims, $err) = $self->verify($token);
    return (undef, $err) unless $claims;
    
    delete $claims->{$_} for qw(iat exp iss);
    return ($self->generate(%$claims, %extra), undef);
}

sub _b64u { encode_base64url($_[0], "") }
sub _b64d { decode_base64url($_[0]) }
}

package main;

printf "=== JWT (JSON Web Tokens) ===\n\n";

my $jwt = JWT->new(
    secret  => "super-secret-key-2024",
    issuer  => "perl_course",
    ttl     => 3600,
);

# Generate tokens
my $access_token = $jwt->generate(
    sub     => 1,
    username=> "alice",
    role    => "admin",
    ttl     => 900,  # 15 minutes
);
my $refresh_token = $jwt->generate(
    sub     => 1,
    type    => "refresh",
    ttl     => 86400 * 7,  # 7 days
);

printf "Access token:\n  %s\n\n", $access_token;
printf "Token length: %d chars\n\n", length($access_token);

# Verify
my ($claims, $err) = $jwt->verify($access_token);
if ($claims) {
    printf "Valid token:\n";
    printf "  sub=%s username=%s role=%s\n", $claims->{sub}, $claims->{username}, $claims->{role};
    printf "  iss=%s exp=%d (in %ds)\n", $claims->{iss}, $claims->{exp}, $claims->{exp}-time();
} else {
    printf "Invalid: %s\n", $err;
}

# Invalid signature
printf "\nTampered token: ";
my $bad = $access_token;
$bad =~ s/.$//;  # Remove last char
my (undef, $bad_err) = $jwt->verify($bad);
printf "%s\n", $bad_err;

# Expired token
my $expired = $jwt->generate(sub=>99, username=>"ghost", ttl=>-1);
my (undef, $exp_err) = $jwt->verify($expired);
printf "Expired token: %s\n", $exp_err;

# Refresh
my ($new_token, $ref_err) = $jwt->refresh($access_token);
if ($new_token) {
    my ($new_claims) = $jwt->verify($new_token);
    printf "\nRefreshed token: exp=%d (new exp in %ds)\n",
        $new_claims->{exp}, $new_claims->{exp}-time();
} else {
    printf "Refresh failed: %s\n", $ref_err;
}

# Decode without verify (for inspection)
my $raw = $jwt->decode_unverified($access_token);
printf "\nRaw payload: sub=%s username=%s\n", $raw->{sub}, $raw->{username};
```

---

## Step 393: Password Hashing & bcrypt-style

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(sha512 sha256_hex);
use MIME::Base64 qw(encode_base64 decode_base64);

{
package Password;

# PBKDF2-like with SHA-512
sub hash {
    my ($class, $password, %opts) = @_;
    my $rounds  = $opts{rounds}  // 100_000;
    my $salt    = $opts{salt}    // _random_bytes(16);
    my $keylen  = $opts{keylen}  // 32;
    
    # Simplified PBKDF2 with SHA-512
    my $u = sha512($salt . $password);
    my $result = $u;
    for my $i (2..$rounds) {
        $u = sha512($u . $password);
        $result ^= $u;
    }
    
    my $hash = encode_base64($salt . substr($result, 0, $keylen), "");
    return sprintf '$pbkdf2-sha512$%d$%s', $rounds, $hash;
}

sub verify {
    my ($class, $password, $stored) = @_;
    return 0 unless $stored =~ /^\$pbkdf2-sha512\$(\d+)\$(.+)$/;
    my ($rounds, $encoded) = ($1, $2);
    my $data = decode_base64($encoded);
    my $salt = substr($data, 0, 16);
    my $expected = $class->hash($password, rounds => $rounds+0, salt => $salt);
    return _secure_compare($stored, $expected);
}

sub needs_rehash {
    my ($class, $stored, $current_rounds) = @_;
    $current_rounds //= 100_000;
    return 1 unless $stored =~ /\$pbkdf2-sha512\$(\d+)\$/;
    return $1 < $current_rounds;
}

sub strength {
    my ($class, $password) = @_;
    my $score = 0;
    my @issues;
    
    $score += 10 if length($password) >= 8;
    $score += 10 if length($password) >= 12;
    $score += 10 if length($password) >= 16;
    $score += 10 if $password =~ /[a-z]/;
    $score += 10 if $password =~ /[A-Z]/;
    $score += 10 if $password =~ /[0-9]/;
    $score += 10 if $password =~ /[^a-zA-Z0-9]/;
    $score += 20 if length($password) >= 20;
    
    push @issues, "Too short (min 8)" if length($password) < 8;
    push @issues, "Add uppercase" unless $password =~ /[A-Z]/;
    push @issues, "Add numbers"   unless $password =~ /[0-9]/;
    push @issues, "Add symbols"   unless $password =~ /[^a-zA-Z0-9]/;
    
    my $level = $score >= 70 ? "strong" : $score >= 40 ? "medium" : "weak";
    return { score => $score, level => $level, issues => \@issues };
}

sub _random_bytes { join "", map { chr(int rand 256) } 1..$_[0] }

sub _secure_compare {
    my ($a, $b) = @_;
    return 0 unless length($a) == length($b);
    my $diff = 0;
    $diff |= ord(substr($a,$_,1)) ^ ord(substr($b,$_,1)) for 0..length($a)-1;
    return $diff == 0;
}
}

package main;

printf "=== Password Hashing ===\n\n";

# Hash passwords
my $pass = "MyS3cur3P@ss!";
printf "Hashing '%s' (this may take a moment with high rounds)...\n", $pass;

# Use low rounds for demo
my $hash = Password->hash($pass, rounds => 1000);
printf "Hash: %s\n\n", substr($hash, 0, 60) . "...";

# Verify
my $ok = Password->verify($pass, $hash);
printf "Correct password: %s\n", $ok ? "PASS" : "FAIL";

my $bad = Password->verify("wrong-password", $hash);
printf "Wrong password:   %s\n\n", $bad ? "PASS (bad!)" : "FAIL (correct)";

# Needs rehash?
my $old_hash = Password->hash($pass, rounds => 500);
printf "Needs rehash (rounds 500 vs 1000): %s\n",
    Password->needs_rehash($old_hash, 1000) ? "YES" : "NO";
printf "Needs rehash (rounds 1000 vs 500): %s\n\n",
    Password->needs_rehash($hash, 500) ? "YES" : "NO";

# Password strength
printf "Password strength:\n";
for my $p ("abc", "password", "P@ssw0rd", "MyS3cur3P\@ss!", "Th1sIsAVeryLongAndSecurePassword!") {
    my $s = Password->strength($p);
    printf "  %-35s score=%d level=%-6s %s\n",
        "\"$p\"", $s->{score}, $s->{level},
        @{$s->{issues}} ? "issues: " . join(", ", @{$s->{issues}}) : "";
}
```

---

## Step 394: OAuth2 Authorization Server

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(sha256_hex hmac_sha256_hex);
use MIME::Base64 qw(encode_base64url decode_base64url);
use JSON::PP;

my $JSON = JSON::PP->new->utf8->canonical;

{
package OAuth2::Server;

my %clients;
my %auth_codes;
my %access_tokens;
my %refresh_tokens;

sub new {
    my ($class, %opts) = @_;
    return bless {
        issuer       => $opts{issuer}    // "https://auth.example.com",
        secret       => $opts{secret}    // die("secret required"),
        code_ttl     => $opts{code_ttl}  // 600,
        access_ttl   => $opts{access_ttl}// 3600,
        refresh_ttl  => $opts{refresh_ttl}// 86400*30,
    }, $class;
}

sub register_client {
    my ($self, %opts) = @_;
    my $id     = "client_" . _random_hex(8);
    my $secret = _random_hex(32);
    $clients{$id} = {
        id           => $id,
        secret       => $secret,
        name         => $opts{name}         // "Unknown App",
        redirect_uris=> $opts{redirect_uris}// [],
        scopes       => $opts{scopes}       // ["read"],
        type         => $opts{type}         // "confidential",
    };
    return ($id, $secret);
}

sub authorize {
    my ($self, %params) = @_;
    my $client = $clients{$params{client_id}} or return (undef, "invalid_client");
    
    # Validate redirect_uri
    my $redir = $params{redirect_uri};
    return (undef, "invalid_redirect_uri")
        unless grep { $_ eq $redir } @{$client->{redirect_uris}};
    
    # Validate scopes
    my @req_scopes = split /\s+/, $params{scope}//"";
    for my $s (@req_scopes) {
        return (undef, "invalid_scope") unless grep { $_ eq $s } @{$client->{scopes}};
    }
    
    # Create auth code (PKCE: code_challenge)
    my $code = _random_hex(32);
    $auth_codes{$code} = {
        client_id      => $params{client_id},
        redirect_uri   => $redir,
        scope          => $params{scope},
        user_id        => $params{user_id},
        code_challenge => $params{code_challenge},
        expires_at     => time() + $self->{code_ttl},
    };
    
    return ($code, undef);
}

sub token {
    my ($self, %params) = @_;
    my $grant = $params{grant_type} // "";
    
    if ($grant eq "authorization_code") {
        return $self->_token_auth_code(%params);
    } elsif ($grant eq "refresh_token") {
        return $self->_token_refresh(%params);
    } elsif ($grant eq "client_credentials") {
        return $self->_token_client_creds(%params);
    }
    return (undef, "unsupported_grant_type");
}

sub _token_auth_code {
    my ($self, %p) = @_;
    my $ac = $auth_codes{$p{code}} or return (undef, "invalid_grant");
    return (undef, "expired_code") if $ac->{expires_at} < time();
    return (undef, "client_mismatch") if $ac->{client_id} ne ($p{client_id}//"");
    
    # PKCE verification
    if ($ac->{code_challenge}) {
        my $verifier = $p{code_verifier} // return (undef, "missing_verifier");
        my $challenge = encode_base64url(Digest::SHA::sha256($verifier), "");
        return (undef, "pkce_failed") unless $challenge eq $ac->{code_challenge};
    }
    
    delete $auth_codes{$p{code}};
    return $self->_issue_tokens($ac->{user_id}, $ac->{client_id}, $ac->{scope});
}

sub _token_refresh {
    my ($self, %p) = @_;
    my $rt = $refresh_tokens{$p{refresh_token}} or return (undef, "invalid_grant");
    return (undef, "expired_refresh_token") if $rt->{expires_at} < time();
    delete $refresh_tokens{$p{refresh_token}};
    return $self->_issue_tokens($rt->{user_id}, $rt->{client_id}, $rt->{scope});
}

sub _token_client_creds {
    my ($self, %p) = @_;
    my $client = $clients{$p{client_id}} or return (undef, "invalid_client");
    return (undef, "invalid_client_secret") unless $client->{secret} eq ($p{client_secret}//"");
    return $self->_issue_tokens(undef, $p{client_id}, $p{scope}//"read");
}

sub _issue_tokens {
    my ($self, $user_id, $client_id, $scope) = @_;
    
    my $at = _random_hex(32);
    my $rt = _random_hex(32);
    
    $access_tokens{$at} = {
        user_id    => $user_id,
        client_id  => $client_id,
        scope      => $scope,
        expires_at => time() + $self->{access_ttl},
    };
    $refresh_tokens{$rt} = {
        user_id    => $user_id,
        client_id  => $client_id,
        scope      => $scope,
        expires_at => time() + $self->{refresh_ttl},
    };
    
    return ({
        access_token  => $at,
        refresh_token => $rt,
        token_type    => "Bearer",
        expires_in    => $self->{access_ttl},
        scope         => $scope,
    }, undef);
}

sub introspect {
    my ($self, $token) = @_;
    my $t = $access_tokens{$token};
    return { active => 0 } unless $t && $t->{expires_at} > time();
    return {
        active     => 1,
        client_id  => $t->{client_id},
        username   => $t->{user_id} ? "user_$t->{user_id}" : undef,
        scope      => $t->{scope},
        exp        => $t->{expires_at},
    };
}

sub revoke {
    my ($self, $token) = @_;
    delete $access_tokens{$token};
    delete $refresh_tokens{$token};
    return 1;
}

sub _random_hex { join "", map { sprintf "%02x", int rand 256 } 1..$_[0] }
}

package main;

printf "=== OAuth2 Authorization Server ===\n\n";

my $auth = OAuth2::Server->new(secret => "oauth2-server-secret");

# Register a client
my ($client_id, $client_secret) = $auth->register_client(
    name          => "My Web App",
    redirect_uris => ["https://app.example.com/callback"],
    scopes        => [qw(read write admin)],
    type          => "confidential",
);

printf "Client registered:\n  id=%s\n  secret=%s\n\n",
    $client_id, substr($client_secret, 0, 16) . "...";

# Authorization Code flow with PKCE
printf "--- Authorization Code Flow (PKCE) ---\n";

# Generate PKCE
my $code_verifier  = join "", map { ("a".."z","A".."Z","0".."9")[rand 62] } 1..64;
my $code_challenge = MIME::Base64::encode_base64url(Digest::SHA::sha256($code_verifier), "");

printf "PKCE verifier:  %s...\n", substr($code_verifier, 0, 20);
printf "PKCE challenge: %s...\n\n", substr($code_challenge, 0, 20);

# User authorizes
my ($code, $err) = $auth->authorize(
    client_id      => $client_id,
    redirect_uri   => "https://app.example.com/callback",
    scope          => "read write",
    user_id        => 42,
    code_challenge => $code_challenge,
);

printf "Auth code: %s (err=%s)\n\n", $code ? substr($code,0,16)."..." : "NONE", $err//"none";

# Exchange code for token
my ($tokens, $token_err) = $auth->token(
    grant_type    => "authorization_code",
    code          => $code,
    client_id     => $client_id,
    redirect_uri  => "https://app.example.com/callback",
    code_verifier => $code_verifier,
);

if ($tokens) {
    printf "Token response:\n";
    printf "  access_token:  %s...\n", substr($tokens->{access_token}, 0, 16);
    printf "  refresh_token: %s...\n", substr($tokens->{refresh_token}, 0, 16);
    printf "  token_type:    %s\n", $tokens->{token_type};
    printf "  expires_in:    %d\n", $tokens->{expires_in};
    printf "  scope:         %s\n\n", $tokens->{scope};
    
    # Introspect
    my $info = $auth->introspect($tokens->{access_token});
    printf "Introspect:\n  active=%s scope=%s client_id=%s\n\n",
        $info->{active} ? "yes" : "no", $info->{scope}, $info->{client_id};
    
    # Refresh
    my ($new_tokens, $rt_err) = $auth->token(
        grant_type    => "refresh_token",
        refresh_token => $tokens->{refresh_token},
        client_id     => $client_id,
    );
    printf "Refresh: %s\n\n", $new_tokens ? "new access token issued" : "error: $rt_err";
    
    # Revoke
    $auth->revoke($tokens->{access_token});
    my $revoked = $auth->introspect($tokens->{access_token});
    printf "After revoke: active=%s\n", $revoked->{active} ? "yes" : "no";
} else {
    printf "Token error: %s\n", $token_err;
}

# Client credentials flow
printf "\n--- Client Credentials Flow ---\n";
my ($cc_tokens, $cc_err) = $auth->token(
    grant_type    => "client_credentials",
    client_id     => $client_id,
    client_secret => $client_secret,
    scope         => "read",
);

printf "Service-to-service token: %s\n",
    $cc_tokens ? substr($cc_tokens->{access_token},0,16)."..." : "error: $cc_err";
```

---

## Step 395: OpenID Connect

```perl
#!/usr/bin/perl
use strict;
use warnings;
use MIME::Base64 qw(encode_base64url decode_base64url);
use Digest::SHA qw(sha256_hex hmac_sha256_hex);
use JSON::PP;

my $JSON = JSON::PP->new->utf8->canonical;

{
package OIDC;

sub new {
    my ($class, %opts) = @_;
    return bless {
        issuer    => $opts{issuer}    // "https://auth.example.com",
        secret    => $opts{secret}    // die("secret required"),
        ttl       => $opts{ttl}       // 3600,
        users     => $opts{users}     // {},
    }, $class;
}

sub discovery {
    my $self = shift;
    my $base = $self->{issuer};
    return {
        issuer                   => $base,
        authorization_endpoint   => "$base/authorize",
        token_endpoint           => "$base/token",
        userinfo_endpoint        => "$base/userinfo",
        jwks_uri                 => "$base/.well-known/jwks.json",
        response_types_supported => ["code", "token", "id_token"],
        subject_types_supported  => ["public"],
        id_token_signing_alg     => ["HS256"],
        scopes_supported         => [qw(openid profile email phone address)],
        claims_supported         => [qw(sub name email email_verified picture given_name family_name)],
    };
}

sub create_id_token {
    my ($self, %opts) = @_;
    my $user = $self->{users}{$opts{sub}} // {};
    my $now  = time();
    
    my %claims = (
        iss         => $self->{issuer},
        sub         => $opts{sub},
        aud         => $opts{client_id},
        iat         => $now,
        exp         => $now + $self->{ttl},
        nonce       => $opts{nonce},
        at_hash     => $opts{access_token} ? substr(sha256_hex($opts{access_token}),0,16) : undef,
    );
    
    # Scope-based claim inclusion
    my %scopes = map { $_ => 1 } split /\s+/, $opts{scope}//"openid";
    if ($scopes{profile}) {
        @claims{qw(name given_name family_name picture)} =
            @{$user}{qw(name given_name family_name picture)};
    }
    if ($scopes{email}) {
        @claims{qw(email email_verified)} = @{$user}{qw(email email_verified)};
    }
    
    # Remove undefs
    delete $claims{$_} for grep { !defined $claims{$_} } keys %claims;
    
    # Build JWT
    my $h = encode_base64url($JSON->encode({typ=>"JWT",alg=>"HS256"}), "");
    my $p = encode_base64url($JSON->encode(\%claims), "");
    my $s = encode_base64url(Digest::SHA::hmac_sha256("$h.$p", $self->{secret}), "");
    return "$h.$p.$s";
}

sub verify_id_token {
    my ($self, $token) = @_;
    my @parts = split /\./, $token;
    return (undef, "invalid_format") unless @parts == 3;
    my ($h, $p, $s) = @parts;
    
    my $expected = encode_base64url(Digest::SHA::hmac_sha256("$h.$p", $self->{secret}), "");
    return (undef, "invalid_signature") unless $expected eq $s;
    
    my $claims = eval { $JSON->decode(decode_base64url($p)) };
    return (undef, "invalid_payload") if $@;
    return (undef, "expired") if ($claims->{exp}//"") < time();
    return ($claims, undef);
}

sub userinfo {
    my ($self, $access_token, $scope) = @_;
    # In real OIDC, lookup user from token
    # Simplified: return mock data
    my %scopes = map { $_ => 1 } split /\s+/, $scope//"";
    my $user = { sub => "42" };
    $user->{name} = "Alice Smith" if $scopes{profile};
    $user->{email} = "alice\@example.com" if $scopes{email};
    return $user;
}
}

package main;

printf "=== OpenID Connect ===\n\n";

my $oidc = OIDC->new(
    issuer => "https://auth.myapp.com",
    secret => "oidc-secret-key",
    users  => {
        "42" => {
            name       => "Alice Smith",
            given_name => "Alice",
            family_name=> "Smith",
            email      => "alice\@example.com",
            email_verified => 1,
            picture    => "https://example.com/alice.jpg",
        },
    },
);

# Discovery document
my $disc = $oidc->discovery;
printf "OIDC Discovery:\n";
printf "  issuer: %s\n", $disc->{issuer};
printf "  auth:   %s\n", $disc->{authorization_endpoint};
printf "  token:  %s\n", $disc->{token_endpoint};
printf "  scopes: %s\n\n", join(", ", @{$disc->{scopes_supported}});

# Create ID token
my $id_token = $oidc->create_id_token(
    sub        => "42",
    client_id  => "my_client",
    scope      => "openid profile email",
    nonce      => "random_nonce_123",
);

printf "ID Token: %s...\n\n", substr($id_token, 0, 50);

# Verify
my ($claims, $err) = $oidc->verify_id_token($id_token);
if ($claims) {
    printf "ID Token claims:\n";
    for my $k (sort keys %$claims) {
        printf "  %s = %s\n", $k, defined $claims->{$k} ? $claims->{$k} : "(undef)";
    }
} else {
    printf "Verify failed: %s\n", $err;
}

# UserInfo endpoint
my $ui = $oidc->userinfo("access_token_xyz", "openid profile email");
printf "\nUserInfo:\n";
printf "  %s = %s\n", $_, $ui->{$_} for sort keys %$ui;
```

---

## Step 396: Two-Factor Authentication (TOTP)

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(hmac_sha1);
use MIME::Base64 qw(decode_base64);

{
package TOTP;

# RFC 6238 — TOTP implementation
sub new {
    my ($class, %opts) = @_;
    return bless {
        digits  => $opts{digits}  // 6,
        period  => $opts{period}  // 30,
        algo    => $opts{algo}    // "SHA1",
    }, $class;
}

sub generate_secret {
    my ($class, $len) = @_;
    $len //= 20;
    my @chars = ('A'..'Z', '2'..'7');  # Base32 chars
    return join "", map { $chars[rand @chars] } 1..$len;
}

sub hotp {
    my ($self, $secret, $counter) = @_;
    
    # Decode base32 secret
    my $key = _base32_decode($secret);
    
    # Pack counter as 8-byte big-endian
    my $msg = pack("NN", 0, $counter);
    
    # HMAC-SHA1
    my $hmac = hmac_sha1($msg, $key);
    
    # Dynamic truncation
    my $offset = ord(substr($hmac, -1)) & 0x0F;
    my $trunc   = substr($hmac, $offset, 4);
    my $code    = (unpack("N", $trunc) & 0x7FFFFFFF) % (10 ** $self->{digits});
    
    return sprintf "%0*d", $self->{digits}, $code;
}

sub totp {
    my ($self, $secret, $time) = @_;
    $time //= time();
    my $counter = int($time / $self->{period});
    return $self->hotp($secret, $counter);
}

sub verify {
    my ($self, $secret, $code, %opts) = @_;
    my $time    = $opts{time}    // time();
    my $window  = $opts{window}  // 1;  # Allow 1 step drift
    
    my $counter = int($time / $self->{period});
    for my $c ($counter - $window .. $counter + $window) {
        return 1 if $self->hotp($secret, $c) eq $code;
    }
    return 0;
}

sub provisioning_uri {
    my ($self, %opts) = @_;
    my $label  = $opts{account} // "user\@example.com";
    my $issuer = $opts{issuer}  // "MyApp";
    my $secret = $opts{secret}  // die "secret required";
    
    my $uri = "otpauth://totp/" . _url_encode("$issuer:$label")
            . "?secret=$secret"
            . "&issuer=" . _url_encode($issuer)
            . "&digits=$self->{digits}"
            . "&period=$self->{period}";
    return $uri;
}

sub _base32_decode {
    my $input = uc(shift);
    $input =~ s/=//g;
    my $alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZ234567";
    my %map = map { substr($alphabet,$_,1) => $_ } 0..31;
    
    my $bits = "";
    for my $c (split //, $input) {
        $bits .= sprintf "%05b", $map{$c} // 0;
    }
    
    my $result = "";
    while (length($bits) >= 8) {
        $result .= chr(oct("0b" . substr($bits, 0, 8)));
        $bits = substr($bits, 8);
    }
    return $result;
}

sub _url_encode {
    my $s = shift;
    $s =~ s/([^A-Za-z0-9\-_.~])/ sprintf "%%%02X", ord($1) /ge;
    return $s;
}
}

package main;

printf "=== TOTP Two-Factor Authentication ===\n\n";

my $totp = TOTP->new(digits => 6, period => 30);

# Generate a secret
my $secret = "JBSWY3DPEHPK3PXP";  # Example secret (base32)
printf "Secret: %s\n", $secret;

# Generate current code
my $fake_time = 1700000000;  # Fixed time for deterministic output
my $code = $totp->totp($secret, $fake_time);
printf "TOTP code at t=%d: %s\n\n", $fake_time, $code;

# Verify
printf "Verification:\n";
printf "  Correct code:  %s\n", $totp->verify($secret, $code, time => $fake_time) ? "PASS" : "FAIL";
printf "  Wrong code:    %s\n", $totp->verify($secret, "000000", time => $fake_time) ? "PASS" : "FAIL";

# Window verification (clock drift)
my $prev_code = $totp->totp($secret, $fake_time - 30);
printf "  Previous step: %s\n", $totp->verify($secret, $prev_code, time => $fake_time, window => 1) ? "PASS (in window)" : "FAIL";

# Provisioning URI (for QR code)
my $uri = $totp->provisioning_uri(
    account => "alice\@example.com",
    issuer  => "MyApp",
    secret  => $secret,
);
printf "\nProvisioning URI:\n  %s\n\n", $uri;

# Generate a new secret
my $new_secret = TOTP->generate_secret;
printf "New random secret: %s\n", $new_secret;

# Show multiple time windows
printf "\nCodes for adjacent windows:\n";
for my $offset (-2..2) {
    my $t = $fake_time + ($offset * 30);
    my $c = $totp->totp($secret, $t);
    printf "  t+%3ds (window %d): %s%s\n",
        $offset*30, int($t/30), $c, $offset==0 ? " <-- current" : "";
}
```

---

## Step 397: API Key Management

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(sha256_hex hmac_sha256_hex);

{
package APIKey;

my %keys;
my $key_num = 0;

sub new {
    my ($class, %opts) = @_;
    return bless {
        secret     => $opts{secret} // die("secret required"),
        prefix     => $opts{prefix} // "sk",
        version    => $opts{version}// "v1",
    }, $class;
}

sub create {
    my ($self, %opts) = @_;
    my $raw_key = $self->{prefix} . "_" . $self->{version} . "_" . _random_hex(32);
    my $key_id  = "kid_" . ++$key_num;
    my $hash    = sha256_hex($raw_key);
    
    $keys{$hash} = {
        id         => $key_id,
        hash       => $hash,
        prefix     => substr($raw_key, 0, 10),
        name       => $opts{name}    // "API Key",
        user_id    => $opts{user_id},
        scopes     => $opts{scopes}  // ["read"],
        created_at => time(),
        expires_at => $opts{expires} ? time() + $opts{expires} : undef,
        last_used  => undef,
        usage      => 0,
        rate_limit => $opts{rate_limit} // 1000,
        active     => 1,
    };
    
    return ($raw_key, $key_id);
}

sub verify {
    my ($self, $raw_key) = @_;
    my $hash = sha256_hex($raw_key);
    my $meta = $keys{$hash} or return (undef, "invalid_key");
    
    return (undef, "key_revoked")  unless $meta->{active};
    return (undef, "key_expired")
        if $meta->{expires_at} && $meta->{expires_at} < time();
    
    $meta->{last_used} = time();
    $meta->{usage}++;
    
    return ($meta, undef);
}

sub has_scope {
    my ($self, $raw_key, $scope) = @_;
    my ($meta) = $self->verify($raw_key);
    return 0 unless $meta;
    return grep { $_ eq $scope || $_ eq "*" } @{$meta->{scopes}};
}

sub revoke {
    my ($self, $key_id) = @_;
    for my $h (keys %keys) {
        if ($keys{$h}{id} eq $key_id) {
            $keys{$h}{active} = 0;
            return 1;
        }
    }
    return 0;
}

sub rotate {
    my ($self, $old_key) = @_;
    my ($meta, $err) = $self->verify($old_key);
    return (undef, $err) unless $meta;
    
    # Revoke old, create new
    $self->revoke($meta->{id});
    return $self->create(
        name      => $meta->{name} . " (rotated)",
        user_id   => $meta->{user_id},
        scopes    => $meta->{scopes},
        rate_limit=> $meta->{rate_limit},
    );
}

sub list_for_user {
    my ($self, $user_id) = @_;
    return [
        map { { id => $_->{id}, prefix => $_->{prefix}, name => $_->{name},
                active => $_->{active}, scopes => $_->{scopes}, usage => $_->{usage} } }
        grep { $_->{user_id} == $user_id }
        values %keys
    ];
}

sub _random_hex { join "", map { sprintf "%02x", int rand 256 } 1..$_[0] }
}

package main;

printf "=== API Key Management ===\n\n";

my $km = APIKey->new(secret => "key-signing-secret", prefix => "pk");

# Create keys
printf "Creating API keys:\n";
my ($key1, $kid1) = $km->create(user_id => 1, name => "Production Key",
    scopes => ["read", "write"], rate_limit => 5000);
my ($key2, $kid2) = $km->create(user_id => 1, name => "Read-only Key",
    scopes => ["read"], expires => 86400*30);
my ($key3, $kid3) = $km->create(user_id => 2, name => "Admin Key",
    scopes => ["read", "write", "admin"]);

printf "  %s: %s...\n", $kid1, substr($key1, 0, 20);
printf "  %s: %s...\n", $kid2, substr($key2, 0, 20);
printf "  %s: %s...\n\n", $kid3, substr($key3, 0, 20);

# Verify
printf "Verifying:\n";
my ($meta, $err) = $km->verify($key1);
if ($meta) {
    printf "  Valid: id=%s name='%s' usage=%d\n", $meta->{id}, $meta->{name}, $meta->{usage};
} else {
    printf "  Invalid: %s\n", $err;
}

(undef, my $bad_err) = $km->verify("invalid_key_12345");
printf "  Invalid key: %s\n\n", $bad_err;

# Scope check
printf "Scope checks:\n";
printf "  key1 has 'write': %s\n", $km->has_scope($key1, "write") ? "YES" : "NO";
printf "  key2 has 'write': %s\n", $km->has_scope($key2, "write") ? "YES" : "NO";
printf "  key3 has 'admin': %s\n\n", $km->has_scope($key3, "admin") ? "YES" : "NO";

# Revoke
$km->revoke($kid2);
my (undef, $rev_err) = $km->verify($key2);
printf "After revoke key2: %s\n\n", $rev_err;

# Rotate
my ($new_key, $nkid) = $km->rotate($key1);
printf "Rotated key1: new_id=%s key=%s...\n", $nkid, substr($new_key, 0, 20);
my (undef, $old_err) = $km->verify($key1);
printf "Old key1 after rotate: %s\n\n", $old_err;

# List for user
my $user1_keys = $km->list_for_user(1);
printf "User 1 keys (%d total):\n", scalar @$user1_keys;
printf "  %s: '%s' active=%s\n", $_->{id}, $_->{name}, $_->{active}?"yes":"no"
    for @$user1_keys;
```

---

## Step 398: Permission & RBAC System

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package RBAC;

sub new {
    my ($class) = @_;
    return bless {
        roles       => {},
        permissions => {},
        users       => {},
        _inheritance=> {},
    }, $class;
}

sub define_permission {
    my ($self, $name, $description) = @_;
    $self->{permissions}{$name} = { name => $name, desc => $description };
}

sub define_role {
    my ($self, $name, %opts) = @_;
    $self->{roles}{$name} = {
        name        => $name,
        permissions => $opts{permissions} // [],
        inherits    => $opts{inherits}    // [],
    };
    push @{$self->{_inheritance}{$_}}, $name for @{$opts{inherits}//[]};
}

sub assign_role {
    my ($self, $user_id, @roles) = @_;
    push @{$self->{users}{$user_id}{roles}}, @roles;
    # Deduplicate
    my %seen;
    @{$self->{users}{$user_id}{roles}} = grep { !$seen{$_}++ }
        @{$self->{users}{$user_id}{roles}};
}

sub revoke_role {
    my ($self, $user_id, $role) = @_;
    @{$self->{users}{$user_id}{roles}} =
        grep { $_ ne $role } @{$self->{users}{$user_id}{roles}//[]};
}

sub can {
    my ($self, $user_id, $permission) = @_;
    my %perms = $self->effective_permissions($user_id);
    return exists $perms{$permission} || exists $perms{"*"};
}

sub effective_permissions {
    my ($self, $user_id) = @_;
    my @roles = $self->effective_roles($user_id);
    my %perms;
    for my $role_name (@roles) {
        my $role = $self->{roles}{$role_name} or next;
        $perms{$_} = 1 for @{$role->{permissions}};
    }
    return %perms;
}

sub effective_roles {
    my ($self, $user_id) = @_;
    my %seen;
    my @result;
    my @queue = @{$self->{users}{$user_id}{roles}//[]};
    while (my $role = shift @queue) {
        next if $seen{$role}++;
        push @result, $role;
        push @queue, @{($self->{roles}{$role}//{inherits=>[]})->{inherits}};
    }
    return @result;
}

sub roles_for {
    my ($self, $user_id) = @_;
    return @{$self->{users}{$user_id}{roles}//[]};
}
}

{
package Policy;
# Resource-based policies (ABAC)

sub new { bless { rules => [] }, $_[0] }

sub allow {
    my ($self, %rule) = @_;
    push @{$self->{rules}}, { effect => "allow", %rule };
    return $self;
}

sub deny {
    my ($self, %rule) = @_;
    push @{$self->{rules}}, { effect => "deny", %rule };
    return $self;
}

sub evaluate {
    my ($self, %ctx) = @_;
    # Deny takes precedence
    for my $rule (@{$self->{rules}}) {
        next unless _matches($rule, \%ctx);
        return 0 if $rule->{effect} eq "deny";
    }
    for my $rule (@{$self->{rules}}) {
        next unless _matches($rule, \%ctx);
        return 1 if $rule->{effect} eq "allow";
    }
    return 0;  # Default deny
}

sub _matches {
    my ($rule, $ctx) = @_;
    for my $k (qw(action resource role)) {
        next unless defined $rule->{$k};
        my $v = $ctx->{$k} // return 0;
        if (ref $rule->{$k} eq "ARRAY") {
            return 0 unless grep { $_ eq $v } @{$rule->{$k}};
        } else {
            return 0 unless $rule->{$k} eq $v || $rule->{$k} eq "*";
        }
    }
    return 1;
}
}

package main;

printf "=== RBAC & Policy System ===\n\n";

my $rbac = RBAC->new;

# Define permissions
$rbac->define_permission("posts.read",   "Read posts");
$rbac->define_permission("posts.write",  "Create/edit posts");
$rbac->define_permission("posts.delete", "Delete posts");
$rbac->define_permission("users.manage", "Manage users");
$rbac->define_permission("admin.*",      "All admin");

# Define roles with inheritance
$rbac->define_role("viewer",    permissions => ["posts.read"]);
$rbac->define_role("author",    permissions => ["posts.write"],  inherits => ["viewer"]);
$rbac->define_role("moderator", permissions => ["posts.delete"], inherits => ["author"]);
$rbac->define_role("admin",     permissions => ["users.manage", "admin.*"], inherits => ["moderator"]);

# Assign
$rbac->assign_role(1, "viewer");
$rbac->assign_role(2, "author");
$rbac->assign_role(3, "moderator");
$rbac->assign_role(4, "admin");

printf "Users and permissions:\n";
for my $uid (1..4) {
    my @roles = $rbac->roles_for($uid);
    my @eff   = $rbac->effective_roles($uid);
    printf "  User %d roles=%s effective_roles=%s\n", $uid,
        join(",",@roles), join(",",@eff);
}

printf "\nPermission checks:\n";
my @checks = (
    [1, "posts.read",   "viewer"],
    [1, "posts.write",  "viewer"],
    [2, "posts.write",  "author"],
    [2, "posts.delete", "author"],
    [3, "posts.delete", "moderator"],
    [4, "users.manage", "admin"],
    [4, "admin.*",      "admin"],
);

for my $c (@checks) {
    my ($uid, $perm, $role) = @$c;
    printf "  User %d (%s) can '%s': %s\n",
        $uid, $role, $perm, $rbac->can($uid, $perm) ? "YES" : "NO";
}

# Policy-based
printf "\n--- ABAC Policy ---\n";
my $policy = Policy->new;
$policy->allow(action => "read",   role => ["viewer","author","moderator","admin"]);
$policy->allow(action => "write",  role => ["author","moderator","admin"]);
$policy->allow(action => "delete", role => ["moderator","admin"]);
$policy->deny( action => "delete", role => "author");  # Explicit deny for safety

printf "Author read:   %s\n", $policy->evaluate(action=>"read",   role=>"author") ?"ALLOW":"DENY";
printf "Author write:  %s\n", $policy->evaluate(action=>"write",  role=>"author") ?"ALLOW":"DENY";
printf "Author delete: %s\n", $policy->evaluate(action=>"delete", role=>"author") ?"ALLOW":"DENY";
printf "Mod delete:    %s\n", $policy->evaluate(action=>"delete", role=>"moderator")?"ALLOW":"DENY";
```

---

## Step 399: Secure Token Storage

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(sha256_hex hmac_sha256_hex);
use MIME::Base64 qw(encode_base64 decode_base64 encode_base64url);

{
package SecureStore;

# Envelope encryption: each value encrypted with a data key,
# data key encrypted with a master key

sub new {
    my ($class, %opts) = @_;
    return bless {
        master_key => $opts{master_key} // die("master_key required"),
        store      => {},
    }, $class;
}

sub put {
    my ($self, $namespace, $key, $value) = @_;
    
    # Generate data key
    my $data_key = _random_bytes(32);
    
    # Encrypt value (XOR stream cipher simulation)
    my $iv      = _random_bytes(16);
    my $ciphertext = _stream_encrypt($value, $data_key, $iv);
    
    # Encrypt data key with master key
    my $master_iv   = _random_bytes(16);
    my $enc_datakey = _stream_encrypt($data_key, sha256_hex($self->{master_key}), $master_iv);
    
    # MAC
    my $mac = hmac_sha256_hex("$namespace:$key:$ciphertext:$enc_datakey", $self->{master_key});
    
    $self->{store}{"$namespace:$key"} = {
        ciphertext  => encode_base64($ciphertext, ""),
        enc_datakey => encode_base64($enc_datakey, ""),
        iv          => encode_base64($iv, ""),
        master_iv   => encode_base64($master_iv, ""),
        mac         => $mac,
    };
}

sub get {
    my ($self, $namespace, $key) = @_;
    my $entry = $self->{store}{"$namespace:$key"} or return (undef, "not_found");
    
    my $ciphertext  = decode_base64($entry->{ciphertext});
    my $enc_datakey = decode_base64($entry->{enc_datakey});
    
    # Verify MAC
    my $expected_mac = hmac_sha256_hex(
        "$namespace:$key:$ciphertext:$enc_datakey", $self->{master_key}
    );
    return (undef, "integrity_failed") unless $expected_mac eq $entry->{mac};
    
    # Decrypt data key
    my $master_iv = decode_base64($entry->{master_iv});
    my $data_key  = _stream_encrypt($enc_datakey, sha256_hex($self->{master_key}), $master_iv);
    
    # Decrypt value
    my $iv    = decode_base64($entry->{iv});
    my $value = _stream_encrypt($ciphertext, $data_key, $iv);
    
    return ($value, undef);
}

sub delete { delete $_[0]->{store}{"$_[1]:$_[2]"}; 1 }

sub list {
    my ($self, $namespace) = @_;
    return [map { (split /:/, $_, 2)[1] }
            grep { /^\Q$namespace\E:/ }
            keys %{$self->{store}}];
}

sub _stream_encrypt {
    my ($data, $key, $iv) = @_;
    # Simplified stream cipher using SHA-256 keystream
    my $keystream = "";
    my $counter   = 0;
    while (length($keystream) < length($data)) {
        $keystream .= sha256_hex($key . $iv . pack("N", $counter++));
    }
    my $ks_bytes = pack("H*", substr($keystream, 0, length($data)*2));
    return $data ^ $ks_bytes;
}

sub _random_bytes { join "", map { chr(int rand 256) } 1..$_[0] }
}

package main;

printf "=== Secure Token Storage ===\n\n";

my $store = SecureStore->new(master_key => "my-master-key-2024");

# Store sensitive data
printf "Storing secrets:\n";
$store->put("tokens", "access_token",   "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...");
$store->put("tokens", "refresh_token",  "eyRefreshTokenABC123...");
$store->put("creds",  "db_password",    "supersecret_db_pass!");
$store->put("creds",  "api_key",        "sk_live_abcdefghij1234567890");
$store->put("config", "smtp_password",  "mail_password_xyz");

for my $ns (qw(tokens creds config)) {
    printf "  %s: %s\n", $ns, join(", ", @{$store->list($ns)});
}

# Retrieve
printf "\nRetrieving:\n";
for my $item (["tokens","access_token"], ["creds","db_password"], ["config","smtp_password"]) {
    my ($ns, $k) = @$item;
    my ($val, $err) = $store->get($ns, $k);
    printf "  %s/%s: %s\n", $ns, $k,
        $val ? substr($val, 0, 30) . "..." : "ERROR: $err";
}

# Integrity check
printf "\nIntegrity:\n";
my ($v, $e) = $store->get("tokens", "access_token");
printf "  Valid fetch: %s\n", $v ? "OK" : "FAIL: $e";

# Delete
$store->delete("tokens", "refresh_token");
my (undef, $del_err) = $store->get("tokens", "refresh_token");
printf "  After delete: %s\n\n", $del_err;

printf "Remaining keys:\n";
for my $ns (qw(tokens creds config)) {
    my @keys = @{$store->list($ns)};
    printf "  %s: [%s]\n", $ns, join(", ", @keys) if @keys;
}
```

---

## Step 400: Capstone — Complete Authentication Service

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;
use Digest::SHA qw(sha256_hex hmac_sha256_hex);
use MIME::Base64 qw(encode_base64url decode_base64url);

my $JSON = JSON::PP->new->utf8->canonical;

# Complete auth service with: registration, login, JWT, 2FA, API keys, RBAC
my %db_users;
my %db_sessions;
my %db_api_keys;
my $uid_seq = 0;

# --- JWT helpers ---
my $JWT_SECRET = "jwt-secret-2024";
sub jwt_sign {
    my %p = @_;
    $p{iat} //= time(); $p{exp} //= time()+3600;
    my $h = encode_base64url($JSON->encode({typ=>"JWT",alg=>"HS256"}), "");
    my $pay = encode_base64url($JSON->encode(\%p), "");
    my $sig = encode_base64url(Digest::SHA::hmac_sha256("$h.$pay", $JWT_SECRET), "");
    return "$h.$pay.$sig";
}

sub jwt_verify {
    my $tok = shift;
    my @p = split /\./, $tok; return undef unless @p==3;
    my $expected = encode_base64url(Digest::SHA::hmac_sha256("$p[0].$p[1]", $JWT_SECRET), "");
    return undef unless $expected eq $p[2];
    my $payload = eval { $JSON->decode(decode_base64url($p[1])) };
    return undef if !$payload || $payload->{exp} < time();
    return $payload;
}

# --- Password helpers ---
sub hash_password {
    my ($pw, $salt) = @_;
    $salt //= sprintf "%08x", rand(0xFFFFFFFF);
    my $h = $pw;
    $h = sha256_hex($h . $salt) for 1..10000;
    return "$salt:$h";
}

sub verify_password {
    my ($pw, $stored) = @_;
    my ($salt, $hash) = split /:/, $stored, 2;
    return hash_password($pw, $salt) eq $stored;
}

# --- Service functions ---
sub auth_register {
    my (%data) = @_;
    return { error => "email_taken" } if grep { $_->{email} eq $data{email} } values %db_users;
    my $id = ++$uid_seq;
    $db_users{$id} = {
        id       => $id,
        email    => $data{email},
        username => $data{username},
        password => hash_password($data{password}),
        role     => $data{role} // "user",
        totp_secret => undef,
        created  => time(),
    };
    return { user_id => $id, message => "registered" };
}

sub auth_login {
    my (%creds) = @_;
    my ($user) = grep { $_->{email} eq $creds{email} } values %db_users;
    return { error => "invalid_credentials" } unless $user;
    return { error => "invalid_credentials" } unless verify_password($creds{password}, $user->{password});
    
    if ($user->{totp_secret} && !$creds{totp_code}) {
        return { requires_2fa => 1, user_id => $user->{id} };
    }
    
    return _issue_tokens($user);
}

sub _issue_tokens {
    my $user = shift;
    my $access = jwt_sign(sub => $user->{id}, role => $user->{role},
                          email => $user->{email}, type => "access", exp => time()+900);
    my $refresh = jwt_sign(sub => $user->{id}, type => "refresh", exp => time()+86400*7);
    
    my $sid = sprintf "sess_%08x", rand(0xFFFFFFFF);
    $db_sessions{$sid} = { user_id => $user->{id}, created => time(), expires => time()+86400 };
    
    return { access_token => $access, refresh_token => $refresh, session_id => $sid,
             token_type => "Bearer", expires_in => 900 };
}

sub auth_me {
    my $token = shift;
    my $claims = jwt_verify($token) or return { error => "invalid_token" };
    my $user   = $db_users{$claims->{sub}} or return { error => "user_not_found" };
    return { id => $user->{id}, email => $user->{email},
             username => $user->{username}, role => $user->{role} };
}

sub auth_refresh {
    my $rt = shift;
    my $claims = jwt_verify($rt) or return { error => "invalid_refresh_token" };
    return { error => "wrong_token_type" } unless ($claims->{type}//"") eq "refresh";
    my $user = $db_users{$claims->{sub}} or return { error => "user_not_found" };
    return _issue_tokens($user);
}

sub create_api_key {
    my ($user_id, %opts) = @_;
    my $raw = "pk_" . sprintf "%064s", join "", map { sprintf "%x", rand(16) } 1..64;
    my $hash = sha256_hex($raw);
    $db_api_keys{$hash} = { user_id => $user_id, name => $opts{name}//"Key",
                             scopes => $opts{scopes}//["read"], created => time() };
    return $raw;
}

sub verify_api_key {
    my $key = shift;
    my $meta = $db_api_keys{sha256_hex($key)} or return undef;
    return { %$meta, user => $db_users{$meta->{user_id}} };
}

# --- Simulation ---
printf "=== Complete Authentication Service ===\n\n";

# Register
printf "1. Registration\n";
my $r1 = auth_register(email=>"alice\@test.com", username=>"alice", password=>"P@ss1234!", role=>"admin");
my $r2 = auth_register(email=>"bob\@test.com",   username=>"bob",   password=>"S3cur3!pass");
my $r3 = auth_register(email=>"alice\@test.com", username=>"dup",   password=>"foo");  # Duplicate

printf "  Alice: %s\n", $r1->{user_id} ? "OK id=$r1->{user_id}" : "ERR: $r1->{error}";
printf "  Bob:   %s\n", $r2->{user_id} ? "OK id=$r2->{user_id}" : "ERR: $r2->{error}";
printf "  Dup:   %s\n\n", $r3->{error} // "unexpected success";

# Login
printf "2. Login\n";
my $login = auth_login(email=>"alice\@test.com", password=>"P@ss1234!");
if ($login->{access_token}) {
    printf "  Login OK: access_token=%s...\n", substr($login->{access_token},0,30);
    printf "  expires_in=%ds\n\n", $login->{expires_in};
} else {
    printf "  Login FAIL: %s\n\n", $login->{error};
}

my $bad_login = auth_login(email=>"alice\@test.com", password=>"wrong!");
printf "  Wrong password: %s\n\n", $bad_login->{error} // "unexpected success";

# Auth me
printf "3. Auth /me\n";
my $me = auth_me($login->{access_token});
printf "  User: id=%d email=%s role=%s\n\n", $me->{id}, $me->{email}, $me->{role};

# Refresh
printf "4. Token Refresh\n";
my $refreshed = auth_refresh($login->{refresh_token});
printf "  New access token: %s...\n\n",
    $refreshed->{access_token} ? substr($refreshed->{access_token},0,30) : "ERR: $refreshed->{error}";

# API Keys
printf "5. API Keys\n";
my $api_key = create_api_key($r1->{user_id}, name => "CI/CD Key", scopes => ["read","deploy"]);
printf "  Created: %s...\n", substr($api_key, 0, 20);

my $key_meta = verify_api_key($api_key);
printf "  Valid: user=%s scopes=%s\n\n",
    $key_meta->{user}{username}, join(",",@{$key_meta->{scopes}});

printf "Auth service running: %d users, %d sessions, %d API keys\n",
    scalar(keys %db_users), scalar(keys %db_sessions), scalar(keys %db_api_keys);
```

---

## สรุป Part 40 — Authentication Systems & OAuth2

### สิ่งที่เรียนรู้:
- **Session Management** — Signed sessions, TTL, refresh, cleanup
- **JWT** — HS256 signing, verify, refresh, PKCE
- **Password Hashing** — PBKDF2-SHA512, needs_rehash, strength check
- **OAuth2 Server** — Auth code + PKCE, refresh token, client credentials
- **OpenID Connect** — Discovery, ID token, userinfo endpoint
- **TOTP** — HOTP/TOTP RFC 6238, PKCE, QR provisioning
- **API Key Management** — Scoped keys, rotation, revocation
- **RBAC** — Role inheritance, effective permissions, Policy ABAC
- **Secure Storage** — Envelope encryption, integrity MAC
- **Capstone** — Complete auth service with all components

**ถัดไป: [Part 41 — Caching Strategies & Performance](part_41.md)**
