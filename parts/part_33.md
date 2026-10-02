# Part 33: CGI Web Development
## Steps 321-330: การพัฒนาเว็บด้วย CGI — Forms, Cookies, Sessions, Templates

---

## Step 321: CGI Basics

```perl
#!/usr/bin/perl
# hello.cgi — Basic CGI script
use strict;
use warnings;

# CGI environment variables
print "Content-Type: text/plain\n\n";

printf "Server: %s\n",   $ENV{SERVER_NAME}    // "(not set)";
printf "Method: %s\n",   $ENV{REQUEST_METHOD} // "(not set)";
printf "Path:   %s\n",   $ENV{PATH_INFO}      // "/";
printf "Query:  %s\n",   $ENV{QUERY_STRING}   // "";
printf "Remote: %s\n",   $ENV{REMOTE_ADDR}    // "127.0.0.1";
```

```perl
#!/usr/bin/perl
# Demo: simulate CGI processing
use strict;
use warnings;

{
package CGI::Simple;

sub new {
    my ($class, %opts) = @_;
    my $self = bless { params => {}, cookies => {} }, $class;
    
    # Simulate parsing — normally reads STDIN and ENV
    if ($opts{params}) {
        $self->{params} = $opts{params};
    } else {
        $self->_parse_query_string($ENV{QUERY_STRING}//"");
    }
    
    $self->_parse_cookies($ENV{HTTP_COOKIE}//"");
    return $self;
}

sub _parse_query_string {
    my ($self, $qs) = @_;
    for my $pair (split /&/, $qs) {
        my ($key, $val) = split /=/, $pair, 2;
        next unless $key;
        $key = _urldecode($key);
        $val = _urldecode($val // "");
        if (exists $self->{params}{$key}) {
            push @{$self->{params}{$key}}, $val;
        } else {
            $self->{params}{$key} = [$val];
        }
    }
}

sub _parse_cookies {
    my ($self, $cookie_str) = @_;
    for my $pair (split /;\s*/, $cookie_str) {
        my ($name, $val) = split /=/, $pair, 2;
        $self->{cookies}{$name} = $val if $name;
    }
}

sub _urldecode {
    my $s = shift // "";
    $s =~ s/\+/ /g;
    $s =~ s/%([0-9A-Fa-f]{2})/chr(hex($1))/ge;
    return $s;
}

sub param {
    my ($self, $name) = @_;
    return unless exists $self->{params}{$name};
    return wantarray ? @{$self->{params}{$name}} : $self->{params}{$name}[0];
}

sub cookie  { $_[0]->{cookies}{$_[1]} }
sub method  { $ENV{REQUEST_METHOD} // "GET" }
sub referer { $ENV{HTTP_REFERER} // "" }
sub user_agent { $ENV{HTTP_USER_AGENT} // "" }
sub remote_addr { $ENV{REMOTE_ADDR} // "" }
sub path_info   { $ENV{PATH_INFO} // "/" }
}

{
package CGI::Response;

sub new { bless { status => 200, headers => {}, body => "", cookies => [] }, $_[0] }

sub content_type { $_[0]->{headers}{"Content-Type"} = $_[1]; $_[0] }
sub header       { $_[0]->{headers}{$_[1]} = $_[2]; $_[0] }
sub status       { $_[0]->{status} = $_[1]; $_[0] }

sub set_cookie {
    my ($self, %c) = @_;
    my $cookie = "$c{name}=$c{value}";
    $cookie .= "; Path=" . ($c{path}//"/");
    $cookie .= "; Max-Age=$c{max_age}"  if $c{max_age};
    $cookie .= "; HttpOnly"             unless $c{allow_js};
    $cookie .= "; Secure"              if $c{secure};
    $cookie .= "; SameSite=" . ($c{samesite}//"Lax");
    push @{$self->{cookies}}, $cookie;
    return $self;
}

sub body { if (@_ > 1) { $_[0]->{body} = $_[1]; return $_[0] } return $_[0]->{body} }

sub to_http {
    my $self = shift;
    my $out  = "HTTP/1.1 " . $self->{status} . "\r\n";
    $out .= "$_: " . $self->{headers}{$_} . "\r\n" for keys %{$self->{headers}};
    $out .= "Set-Cookie: $_\r\n" for @{$self->{cookies}};
    $out .= "\r\n";
    $out .= $self->{body};
    return $out;
}
}

package main;

# Simulate a CGI request
local %ENV = (%ENV,
    REQUEST_METHOD => "GET",
    QUERY_STRING   => "name=Alice+Smith&age=25&color=blue&color=green",
    HTTP_COOKIE    => "session=abc123; theme=dark",
);

my $cgi = CGI::Simple->new;

printf "Params:\n";
printf "  name:  %s\n",      $cgi->param("name");
printf "  age:   %s\n",      $cgi->param("age");
printf "  color: %s\n",      join(", ", $cgi->param("color"));

printf "\nCookies:\n";
printf "  session: %s\n",    $cgi->cookie("session");
printf "  theme:   %s\n",    $cgi->cookie("theme");

# Build response
my $res = CGI::Response->new
    ->content_type("text/html; charset=utf-8")
    ->set_cookie(name => "test", value => "hello", max_age => 3600)
    ->body("<h1>Hello CGI</h1>");

printf "\nResponse:\n%.100s...\n", $res->to_http;
```

---

## Step 322: HTML Form Handling

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package FormHandler;

sub render_form {
    my ($class, %args) = @_;
    my $action  = $args{action}  // "";
    my $method  = $args{method}  // "POST";
    my %values  = %{$args{values}//{}};
    my %errors  = %{$args{errors}//{}};
    
    my $html = <<FORM_START;
<form method="$method" action="$action" novalidate>
FORM_START

    $html .= $class->field_group(
        label       => "Full Name",
        name        => "full_name",
        type        => "text",
        value       => $values{full_name}//"",
        placeholder => "Enter your full name",
        error       => $errors{full_name},
        required    => 1,
    );
    
    $html .= $class->field_group(
        label    => "Email",
        name     => "email",
        type     => "email",
        value    => $values{email}//"",
        error    => $errors{email},
        required => 1,
    );
    
    $html .= $class->field_group(
        label   => "Role",
        name    => "role",
        type    => "select",
        value   => $values{role}//"member",
        options => [
            { value => "",          label => "-- Choose role --" },
            { value => "member",    label => "Member" },
            { value => "moderator", label => "Moderator" },
            { value => "admin",     label => "Admin" },
        ],
    );
    
    $html .= $class->field_group(
        label   => "Newsletter",
        name    => "newsletter",
        type    => "checkbox",
        checked => $values{newsletter},
    );
    
    $html .= $class->field_group(
        label => "Message",
        name  => "message",
        type  => "textarea",
        value => $values{message}//"",
        rows  => 5,
    );
    
    $html .= '<button type="submit">Submit</button>' . "\n";
    $html .= "</form>\n";
    return $html;
}

sub field_group {
    my ($class, %f) = @_;
    my $id    = $f{name};
    my $req   = $f{required} ? " *" : "";
    my $err   = $f{error} ? qq{<span class="error">$f{error}</span>} : "";
    my $html  = qq{<div class="form-group">\n};
    $html    .= qq{  <label for="$id">$f{label}$req</label>\n};
    
    if ($f{type} eq "select") {
        $html .= qq{  <select name="$f{name}" id="$id">\n};
        for my $opt (@{$f{options}}) {
            my $sel = ($f{value} eq $opt->{value}) ? " selected" : "";
            $html  .= qq{    <option value="$opt->{value}"$sel>$opt->{label}</option>\n};
        }
        $html .= "  </select>\n";
    } elsif ($f{type} eq "checkbox") {
        my $checked = $f{checked} ? " checked" : "";
        $html .= qq{  <input type="checkbox" name="$f{name}" id="$id" value="1"$checked>\n};
    } elsif ($f{type} eq "textarea") {
        my $rows = $f{rows} // 3;
        $html .= qq{  <textarea name="$f{name}" id="$id" rows="$rows">} . ($f{value}//"") . "</textarea>\n";
    } else {
        my $ph = $f{placeholder} ? qq{ placeholder="$f{placeholder}"} : "";
        my $req_attr = $f{required} ? " required" : "";
        $html .= qq{  <input type="$f{type}" name="$f{name}" id="$id" value="} . ($f{value}//"") . qq{"$ph$req_attr>\n};
    }
    
    $html .= "  $err\n" if $err;
    $html .= "</div>\n";
    return $html;
}
}

package main;

# Render empty form
my $form = FormHandler->render_form(action => "/submit");
printf "Empty form lines: %d\n", scalar(my @l = split /\n/, $form);

# Render form with values and errors
my $form_with_errors = FormHandler->render_form(
    action => "/submit",
    values => { full_name => "Al", email => "not-email", role => "admin" },
    errors => { full_name => "Too short", email => "Invalid email format" },
);

printf "Form with errors:\n%s\n", $form_with_errors;
```

---

## Step 323: Cookie Management

```perl
#!/usr/bin/perl
use strict;
use warnings;
use POSIX qw(strftime);

{
package Cookie;

sub new {
    my ($class, %args) = @_;
    return bless {
        name     => $args{name}     // die("name required"),
        value    => $args{value}    // "",
        path     => $args{path}     // "/",
        domain   => $args{domain},
        max_age  => $args{max_age},
        secure   => $args{secure}   // 0,
        httponly => $args{httponly} // 1,
        samesite => $args{samesite} // "Lax",
        expires  => $args{expires},
    }, $class;
}

sub to_header {
    my $self = shift;
    my $h    = "$self->{name}=$self->{value}";
    $h .= "; Path=$self->{path}";
    $h .= "; Domain=$self->{domain}"           if $self->{domain};
    $h .= "; Max-Age=$self->{max_age}"          if defined $self->{max_age};
    $h .= "; Expires=" . _gmdate($self->{expires}) if $self->{expires};
    $h .= "; Secure"                            if $self->{secure};
    $h .= "; HttpOnly"                          if $self->{httponly};
    $h .= "; SameSite=$self->{samesite}";
    return $h;
}

sub _gmdate {
    return strftime("%a, %d %b %Y %H:%M:%S GMT", gmtime(shift));
}

# Parse Set-Cookie header
sub parse {
    my ($class, $header) = @_;
    my %c;
    my @parts = split /;\s*/, $header;
    
    my ($name, $value) = split /=/, shift(@parts), 2;
    $c{name}  = $name;
    $c{value} = $value;
    
    for my $part (@parts) {
        my ($k, $v) = split /=/, $part, 2;
        $k = lc $k;
        $k =~ s/^\s+|\s+$//g;
        if    ($k eq "path")     { $c{path}     = $v }
        elsif ($k eq "domain")   { $c{domain}   = $v }
        elsif ($k eq "max-age")  { $c{max_age}  = $v }
        elsif ($k eq "samesite") { $c{samesite} = $v }
        elsif ($k eq "secure")   { $c{secure}   = 1  }
        elsif ($k eq "httponly") { $c{httponly} = 1  }
    }
    
    return $class->new(%c);
}

# Cookie jar (client-side)
package CookieJar;

sub new { bless { cookies => {} }, $_[0] }

sub set {
    my ($self, $cookie) = @_;
    $self->{cookies}{ $cookie->{name} } = $cookie;
}

sub get { $_[0]->{cookies}{$_[1]} }

sub delete {
    my ($self, $name) = @_;
    $self->{cookies}{$name} = Cookie->new(name => $name, value => "", max_age => 0);
}

sub to_request_header {
    my $self = shift;
    return join("; ", map { "$_=" . $self->{cookies}{$_}{value} }
        grep { ($_[0]->{cookies}{$_}{max_age}//"X") ne 0 } keys %{$self->{cookies}});
}
}

package main;

printf "=== Cookie Management ===\n\n";

# Create cookies
my $session_cookie = Cookie->new(
    name     => "session",
    value    => "abc123xyz",
    max_age  => 86400,
    secure   => 1,
    httponly => 1,
    samesite => "Strict",
);

my $pref_cookie = Cookie->new(
    name     => "theme",
    value    => "dark",
    max_age  => 30 * 86400,
    httponly => 0,
    samesite => "Lax",
);

printf "Session cookie:\n  %s\n\n", $session_cookie->to_header;
printf "Preference cookie:\n  %s\n\n", $pref_cookie->to_header;

# Parse cookie header
my $parsed = Cookie->parse("user_id=42; Path=/; SameSite=Lax; HttpOnly; Secure");
printf "Parsed cookie: name=%s value=%s secure=%s httponly=%s\n",
    $parsed->{name}, $parsed->{value}, $parsed->{secure}, $parsed->{httponly};

# Delete cookie
my $delete_cookie = Cookie->new(
    name    => "session",
    value   => "",
    max_age => 0,
);
printf "\nDelete cookie: %s\n", $delete_cookie->to_header;
```

---

## Step 324: Session Management with CGI

```perl
#!/usr/bin/perl
use strict;
use warnings;
use Digest::SHA qw(sha256_hex);

{
package SessionManager;

my %store;

sub start {
    my ($class, $existing_id) = @_;
    
    if ($existing_id && $store{$existing_id}) {
        my $s = $store{$existing_id};
        if (time() < $s->{expires}) {
            $s->{last_active} = time();
            return $existing_id;
        }
        delete $store{$existing_id};
    }
    
    # New session
    my $id = sha256_hex(rand() . time() . $$ . rand());
    $store{$id} = {
        data       => {},
        created    => time(),
        last_active => time(),
        expires    => time() + 3600,
    };
    return $id;
}

sub get    { $store{$_[1]}{data}{$_[2]} }
sub set    { $store{$_[1]}{data}{$_[2]} = $_[3] }
sub delete_key { delete $store{$_[1]}{data}{$_[2]} }

sub destroy {
    my ($class, $id) = @_;
    delete $store{$id};
}

sub all_data { %{$store{$_[1]}{data}//{}} }

sub regenerate {
    my ($class, $old_id) = @_;
    my $data    = $store{$old_id} or return undef;
    my $new_id  = sha256_hex(rand() . time() . $$);
    $store{$new_id} = { %$data, created => time() };
    delete $store{$old_id};
    return $new_id;
}
}

{
package FlashMessage;

sub add {
    my ($class, $sid, $type, $msg) = @_;
    my @flash = @{ SessionManager->get($sid, "__flash") // [] };
    push @flash, { type => $type, message => $msg };
    SessionManager->set($sid, "__flash", \@flash);
}

sub get_all {
    my ($class, $sid) = @_;
    my $msgs = SessionManager->get($sid, "__flash") // [];
    SessionManager->delete_key($sid, "__flash");
    return @$msgs;
}
}

package main;

printf "=== Session Management ===\n\n";

# New session
my $sid = SessionManager->start(undef);
printf "Session ID: %.16s...\n", $sid;

# Store data
SessionManager->set($sid, "user_id",  42);
SessionManager->set($sid, "username", "alice");
SessionManager->set($sid, "role",     "admin");

# Read data
printf "User: %s (id=%d, role=%s)\n",
    SessionManager->get($sid, "username"),
    SessionManager->get($sid, "user_id"),
    SessionManager->get($sid, "role");

# Flash messages
FlashMessage->add($sid, "success", "Profile updated!");
FlashMessage->add($sid, "info",    "Welcome back, alice.");

# Read flashes (one-time)
my @msgs = FlashMessage->get_all($sid);
printf "\nFlash messages:\n";
printf "  [%s] %s\n", $_->{type}, $_->{message} for @msgs;

# Second read — should be empty
my @msgs2 = FlashMessage->get_all($sid);
printf "Flash read again: %d msgs (should be 0)\n", scalar @msgs2;

# Regenerate session ID (after login)
my $new_sid = SessionManager->regenerate($sid);
printf "\nAfter regenerate:\n";
printf "  Old valid: %s\n", SessionManager->get($sid, "username") ? "yes (BAD)" : "no (OK)";
printf "  New valid: %s\n", SessionManager->get($new_sid, "username") ? "yes (OK)" : "no (BAD)";

# Destroy (logout)
SessionManager->destroy($new_sid);
printf "After destroy: %s\n", SessionManager->get($new_sid, "username") ? "still valid" : "destroyed";
```

---

## Step 325: CGI Template System

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Web::Template;

sub new {
    my ($class, %opts) = @_;
    return bless {
        layouts => { default => $opts{default_layout} // '' },
        helpers => {},
    }, $class;
}

sub register_helper {
    my ($self, $name, $code) = @_;
    $self->{helpers}{$name} = $code;
}

sub render_string {
    my ($self, $template, %vars) = @_;
    
    # Process includes: {{> partial_name }}
    $template =~ s/\{\{>\s*(\w+)\s*\}\}/
        exists $vars{_partials}{$1} ? $vars{_partials}{$1} : ""
    /ge;
    
    # Helpers: {{helper arg}}
    my %helpers = %{$self->{helpers}};
    $template =~ s/\{\{(\w+)\s+([^}]*)\}\}/
        exists $helpers{$1} ? $helpers{$1}->($2, \%vars) : "{{$1 $2}}"
    /ge;
    
    # Loops: {{#each items}}<item>{{/each}}
    $template =~ s/\{\{#each\s+(\w+)\}\}(.*?)\{\{\/each\}\}/
        _render_loop($1, $2, \%vars)
    /gse;
    
    # Conditionals: {{#if cond}}...{{#else}}...{{/if}}
    $template =~ s/\{\{#if\s+(\w+)\}\}(.*?)\{\{#else\}\}(.*?)\{\{\/if\}\}/
        _get_var($1, \%vars) ? $2 : $3
    /gse;
    
    $template =~ s/\{\{#if\s+(\w+)\}\}(.*?)\{\{\/if\}\}/
        _get_var($1, \%vars) ? $2 : ""
    /gse;
    
    # Variables with formatting: {{var|format}}
    $template =~ s/\{\{(\w+)\|(\w+)\}\}/
        _format_var(_get_var($1, \%vars), $2)
    /ge;
    
    # Plain variables: {{var}}
    $template =~ s/\{\{(\w+)\}\}/
        defined(_get_var($1, \%vars)) ? _html_escape(_get_var($1, \%vars)) : ""
    /ge;
    
    return $template;
}

sub _get_var  { my ($k, $v) = @_; $v->{$k} }
sub _html_escape {
    my $s = shift // ""; $s =~ s/&/&amp;/g; $s =~ s/</&lt;/g; $s =~ s/>/&gt;/g; return $s }

sub _format_var {
    my ($val, $fmt) = @_;
    return "" unless defined $val;
    return sprintf("%.2f", $val)   if $fmt eq "money";
    return sprintf("%d", $val)     if $fmt eq "int";
    return uc($val)                if $fmt eq "upper";
    return lc($val)                if $fmt eq "lower";
    return substr($val, 0, 100)    if $fmt eq "truncate";
    return $val;
}

sub _render_loop {
    my ($key, $body, $vars) = @_;
    my $list = $vars->{$key};
    return "" unless ref $list eq 'ARRAY';
    
    my $out = "";
    my $i   = 0;
    for my $item (@$list) {
        my %item_vars = ref $item eq 'HASH'
            ? (%$vars, %$item, _index => $i, _first => ($i == 0 ? 1 : 0),
               _last => ($i == $#$list ? 1 : 0))
            : (%$vars, item => $item, _index => $i);
        
        my $tmpl = Web::Template->new;
        $out .= $tmpl->render_string($body, %item_vars);
        $i++;
    }
    return $out;
}
}

package main;

my $tmpl = Web::Template->new;

# Register helpers
$tmpl->register_helper("date", sub { my $ts = shift; POSIX::strftime("%Y-%m-%d", localtime($ts)) });
$tmpl->register_helper("pluralize", sub {
    my ($n, $vars) = @_; my ($num, $word) = split /\s+/, $n, 2;
    $num = ($vars->{$num}//0); return "$num " . ($num == 1 ? $word : "${word}s");
});

my $page_tmpl = q{
<!DOCTYPE html>
<html>
<head><title>{{title}}</title></head>
<body>
<h1>{{title}}</h1>
<p>User: {{username|upper}} | Items: {{pluralize count item}}</p>
{{#if is_admin}}
<p class="admin">Admin panel visible</p>
{{#else}}
<p>Regular user view</p>
{{/if}}
<ul>
{{#each products}}
  <li>{{name}} - ${{price|money}} {{#if _first}}(first!){{/if}}</li>
{{/each}}
</ul>
</body>
</html>
};

my $rendered = $tmpl->render_string($page_tmpl,
    title    => "Product List",
    username => "alice",
    count    => 3,
    is_admin => 1,
    products => [
        { name => "Widget",  price => 9.99  },
        { name => "Gadget",  price => 24.5  },
        { name => "Doohickey", price => 3.0 },
    ],
);

printf "Rendered template:\n%s\n", $rendered;
```

---

## Step 326: File Upload Handler

```perl
#!/usr/bin/perl
use strict;
use warnings;
use File::Path qw(make_path);
use File::Basename qw(basename);
use Digest::SHA qw(sha256_hex);

{
package Upload::Handler;

my $UPLOAD_DIR = "/tmp/uploads_demo";

sub new {
    my ($class, $dir) = @_;
    my $upload_dir = $dir // $UPLOAD_DIR;
    make_path($upload_dir) unless -d $upload_dir;
    return bless { dir => $upload_dir }, $class;
}

sub save {
    my ($self, %args) = @_;
    my $filename    = $args{filename}  // die "filename required";
    my $content     = $args{content}   // die "content required";
    my $mime_type   = $args{mime_type} // "application/octet-stream";
    
    # Validate
    my %allowed = map { $_ => 1 } qw(image/jpeg image/png image/gif application/pdf text/plain);
    die "File type '$mime_type' not allowed\n" unless $allowed{$mime_type};
    die "File too large\n" if length($content) > 10 * 1024 * 1024;
    
    # Sanitize filename
    my $safe = basename($filename);
    $safe =~ s/[^A-Za-z0-9.\-_]/_/g;
    $safe =~ s/\.{2,}/./g;
    
    # Extract extension
    my ($ext) = $safe =~ /\.([^.]+)$/;
    $ext = lc($ext//"bin");
    
    # Generate unique storage name
    my $storage = sha256_hex(time() . $$ . rand()) . ".$ext";
    my $path    = "$self->{dir}/$storage";
    
    # Write file
    open my $fh, ">", $path or die "Cannot write: $!";
    binmode $fh;
    print $fh $content;
    close $fh;
    
    return {
        original  => $safe,
        storage   => $storage,
        path      => $path,
        size      => length($content),
        mime_type => $mime_type,
        url       => "/uploads/$storage",
    };
}

sub delete_file {
    my ($self, $storage) = @_;
    $storage =~ s/[^A-Za-z0-9._-]//g;  # Sanitize
    my $path = "$self->{dir}/$storage";
    unlink $path if -f $path;
}

sub list_files {
    my ($self) = @_;
    opendir my $dh, $self->{dir} or return ();
    my @files = grep { -f "$self->{dir}/$_" } readdir $dh;
    closedir $dh;
    return @files;
}
}

package main;

printf "=== File Upload Handler ===\n\n";

my $handler = Upload::Handler->new("/tmp/test_uploads_$$");

# Fake PNG magic bytes
my $png_content = "\x89PNG\r\n\x1a\n" . ("X" x 1024);

# Valid upload
my $result = eval { $handler->save(
    filename  => "photo.png",
    content   => $png_content,
    mime_type => "image/png",
) };
if ($result) {
    printf "Uploaded: original=%s storage=%s size=%d\n",
        $result->{original}, $result->{storage}, $result->{size};
    printf "URL: %s\n", $result->{url};
} else {
    printf "Upload failed: %s\n", $@;
}

# Invalid type
eval { $handler->save(
    filename  => "script.php",
    content   => "<?php echo 1; ?>",
    mime_type => "application/x-php",
) };
printf "PHP rejected: %s\n", $@ ? "yes (correct)" : "no (BAD)";

# List files
my @files = $handler->list_files;
printf "Files in upload dir: %d\n", scalar @files;

# Cleanup
unlink "/tmp/test_uploads_$$/$$" if -d "/tmp/test_uploads_$$";
eval { File::Path::remove_tree("/tmp/test_uploads_$$") };
```

---

## Step 327: Pagination and Navigation

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Web::Pagination;

sub new {
    my ($class, %args) = @_;
    return bless {
        total    => $args{total}    // 0,
        page     => $args{page}     // 1,
        per_page => $args{per_page} // 10,
        base_url => $args{base_url} // "?",
    }, $class;
}

sub pages    { my $s=shift; int(($s->{total} + $s->{per_page} - 1) / $s->{per_page}) || 1 }
sub has_prev { $_[0]->{page} > 1 }
sub has_next { $_[0]->{page} < $_[0]->pages }
sub prev     { $_[0]->{page} - 1 }
sub next     { $_[0]->{page} + 1 }
sub offset   { ($_[0]->{page} - 1) * $_[0]->{per_page} }

sub page_url {
    my ($self, $p) = @_;
    return "$self->{base_url}page=$p";
}

sub window {
    my ($self, $size) = @_;
    $size //= 5;
    my $half = int($size / 2);
    my $start = $self->{page} - $half;
    my $end   = $self->{page} + $half;
    
    if ($start < 1)            { $end   += 1 - $start; $start = 1 }
    if ($end > $self->pages)   { $start -= $end - $self->pages; $end = $self->pages }
    $start = 1 if $start < 1;
    
    return ($start .. $end);
}

sub render_html {
    my ($self, $size) = @_;
    return "" if $self->pages <= 1;
    
    my @html = ('<nav class="pagination">');
    
    push @html, $self->has_prev
        ? qq{<a href="} . $self->page_url($self->prev) . qq{">← Prev</a>}
        : '<span class="disabled">← Prev</span>';
    
    my @window = $self->window($size // 5);
    push @html, '<a href="' . $self->page_url(1) . '">1</a> ...'
        if $window[0] > 1;
    
    for my $p (@window) {
        push @html, $p == $self->{page}
            ? qq{<strong class="current">$p</strong>}
            : qq{<a href="} . $self->page_url($p) . qq{">$p</a>};
    }
    
    push @html, '... <a href="' . $self->page_url($self->pages) . '">' . $self->pages . '</a>'
        if $window[-1] < $self->pages;
    
    push @html, $self->has_next
        ? qq{<a href="} . $self->page_url($self->next) . qq{">Next →</a>}
        : '<span class="disabled">Next →</span>';
    
    push @html, '</nav>';
    return join(" ", @html);
}
}

package main;

printf "=== Pagination ===\n\n";

# 95 items, 10 per page, currently on page 5
my $pager = Web::Pagination->new(total => 95, page => 5, per_page => 10, base_url => "/items?");

printf "Total: %d  Pages: %d  Current: %d\n", $pager->{total}, $pager->pages, $pager->{page};
printf "Offset: %d  Has prev: %s  Has next: %s\n",
    $pager->offset, $pager->has_prev ? "yes" : "no", $pager->has_next ? "yes" : "no";

my @window = $pager->window(5);
printf "Window: [%s]\n", join(", ", @window);

printf "\nHTML pagination:\n%s\n", $pager->render_html(5);

# Edge cases
for my $page (1, $pager->pages) {
    my $p2 = Web::Pagination->new(total => 95, page => $page, per_page => 10);
    printf "\nPage %d: prev=%s next=%s\n", $page,
        $p2->has_prev ? "yes" : "no", $p2->has_next ? "yes" : "no";
}
```

---

## Step 328: REST-Style CGI Routes

```perl
#!/usr/bin/perl
use strict;
use warnings;
use JSON::PP;

{
package Web::Router;

sub new { bless { routes => [] }, $_[0] }

sub add_route {
    my ($self, $method, $pattern, $handler) = @_;
    # Convert :param to named capture
    my $regex = $pattern;
    $regex =~ s{:(\w+)}{(?<$1>[^/]+)}g;
    $regex = qr{^$regex$};
    push @{$self->{routes}}, {
        method  => uc($method),
        pattern => $regex,
        handler => $handler,
    };
    return $self;
}

sub get    { $_[0]->add_route("GET",    $_[1], $_[2]) }
sub post   { $_[0]->add_route("POST",   $_[1], $_[2]) }
sub put    { $_[0]->add_route("PUT",    $_[1], $_[2]) }
sub delete { $_[0]->add_route("DELETE", $_[1], $_[2]) }

sub dispatch {
    my ($self, $method, $path, %context) = @_;
    $method = uc($method);
    
    for my $route (@{$self->{routes}}) {
        next unless $route->{method} eq $method;
        if ($path =~ $route->{pattern}) {
            my %params = %+;
            return $route->{handler}->({ %context, params => \%params, path => $path });
        }
    }
    
    return { status => 404, body => "Not Found" };
}
}

package main;

my $json  = JSON::PP->new->utf8;
my $router = Web::Router->new;

# In-memory store
my %users = (
    1 => { id => 1, name => "Alice", email => "alice\@example.com" },
    2 => { id => 2, name => "Bob",   email => "bob\@example.com"   },
);
my $next_id = 3;

# Routes
$router->get("/users", sub {
    my @list = values %users;
    { status => 200, body => $json->encode(\@list), content_type => "application/json" }
});

$router->get("/users/:id", sub {
    my $ctx  = shift;
    my $id   = $ctx->{params}{id};
    return { status => 404, body => '{"error":"Not found"}' } unless $users{$id};
    { status => 200, body => $json->encode($users{$id}), content_type => "application/json" }
});

$router->post("/users", sub {
    my $ctx  = shift;
    my $data = eval { $json->decode($ctx->{body}//"{}") } // {};
    
    return { status => 400, body => '{"error":"name required"}' }
        unless $data->{name};
    
    my $id = $next_id++;
    $users{$id} = { id => $id, name => $data->{name}, email => $data->{email}//"" };
    { status => 201, body => $json->encode($users{$id}) }
});

$router->put("/users/:id", sub {
    my $ctx  = shift;
    my $id   = $ctx->{params}{id};
    return { status => 404, body => '{"error":"Not found"}' } unless $users{$id};
    my $data = eval { $json->decode($ctx->{body}//"{}") } // {};
    $users{$id} = { %{$users{$id}}, %$data, id => $id };
    { status => 200, body => $json->encode($users{$id}) }
});

$router->delete("/users/:id", sub {
    my $ctx = shift;
    my $id  = $ctx->{params}{id};
    return { status => 404, body => '{"error":"Not found"}' } unless $users{$id};
    delete $users{$id};
    { status => 200, body => '{"deleted":true}' }
});

# Test routes
printf "=== RESTful Router Test ===\n\n";

my @tests = (
    ["GET",    "/users",   ""],
    ["GET",    "/users/1", ""],
    ["GET",    "/users/99",""],
    ["POST",   "/users",   '{"name":"Carol","email":"carol@example.com"}'],
    ["PUT",    "/users/1", '{"name":"Alice Updated"}'],
    ["DELETE", "/users/2", ""],
    ["GET",    "/users",   ""],
);

for my $test (@tests) {
    my ($method, $path, $body) = @$test;
    my $result = $router->dispatch($method, $path, body => $body);
    printf "%s %-15s => %d: %s\n", $method, $path, $result->{status},
        length($result->{body}) > 60 ? substr($result->{body}, 0, 60) . "..." : $result->{body};
}
```

---

## Step 329: Email with CGI

```perl
#!/usr/bin/perl
use strict;
use warnings;
use MIME::Base64 qw(encode_base64);

{
package Email::Builder;

sub new {
    my ($class, %args) = @_;
    return bless {
        from        => $args{from}        // 'noreply@example.com',
        to          => $args{to}          // [],
        cc          => $args{cc}          // [],
        subject     => $args{subject}     // "(no subject)",
        text_body   => $args{text_body}   // "",
        html_body   => $args{html_body}   // "",
        attachments => [],
    }, $class;
}

sub to       { my ($s,$v)=@_; push @{$s->{to}}, $v; $s }
sub cc       { my ($s,$v)=@_; push @{$s->{cc}}, $v; $s }
sub subject  { $_[0]->{subject}   = $_[1]; $_[0] }
sub text     { $_[0]->{text_body} = $_[1]; $_[0] }
sub html     { $_[0]->{html_body} = $_[1]; $_[0] }

sub attach {
    my ($self, %a) = @_;
    push @{$self->{attachments}}, {
        name         => $a{name}         // "attachment",
        content_type => $a{content_type} // "application/octet-stream",
        content      => $a{content}      // "",
    };
    return $self;
}

sub build {
    my $self  = shift;
    my $bound = "boundary_" . sprintf "%08x%08x", rand(0xFFFFFFFF), rand(0xFFFFFFFF);
    
    my $headers = join "\r\n",
        "From: $self->{from}",
        "To: "   . join(", ", @{$self->{to}}),
        (@{$self->{cc}} ? "Cc: " . join(", ", @{$self->{cc}}) : ()),
        "Subject: $self->{subject}",
        "MIME-Version: 1.0",
        "Content-Type: multipart/mixed; boundary=\"$bound\"",
        "";
    
    my $msg = "$headers\r\n";
    $msg .= "--$bound\r\n";
    $msg .= "Content-Type: multipart/alternative; boundary=\"alt_$bound\"\r\n\r\n";
    
    if ($self->{text_body}) {
        $msg .= "--alt_$bound\r\n";
        $msg .= "Content-Type: text/plain; charset=UTF-8\r\n\r\n";
        $msg .= $self->{text_body} . "\r\n";
    }
    
    if ($self->{html_body}) {
        $msg .= "--alt_$bound\r\n";
        $msg .= "Content-Type: text/html; charset=UTF-8\r\n\r\n";
        $msg .= $self->{html_body} . "\r\n";
    }
    
    $msg .= "--alt_${bound}--\r\n";
    
    for my $att (@{$self->{attachments}}) {
        $msg .= "--$bound\r\n";
        $msg .= "Content-Type: $att->{content_type}\r\n";
        $msg .= "Content-Disposition: attachment; filename=\"$att->{name}\"\r\n";
        $msg .= "Content-Transfer-Encoding: base64\r\n\r\n";
        $msg .= encode_base64($att->{content});
        $msg .= "\r\n";
    }
    
    $msg .= "--${bound}--\r\n";
    return $msg;
}
}

# Email templates
{
package Email::Templates;

sub welcome {
    my ($class, %vars) = @_;
    return Email::Builder->new(
        subject => "Welcome to " . ($vars{app_name}//"App"),
    )->text(<<"TEXT")->html(<<"HTML");
Welcome, $vars{username}!

Your account has been created.
Login: $vars{login_url}

Thanks,
${\($vars{app_name}//"App")} Team
TEXT
<h1>Welcome, $vars{username}!</h1>
<p>Your account has been created.</p>
<p><a href="$vars{login_url}">Click here to login</a></p>
HTML
}

sub password_reset {
    my ($class, %vars) = @_;
    return Email::Builder->new(
        subject => "Password Reset",
    )->text(<<"TEXT");
Hi $vars{username},

Reset your password: $vars{reset_url}

This link expires in 1 hour.
TEXT
}
}

package main;

printf "=== Email Builder ===\n\n";

my $email = Email::Templates->welcome(
    username  => "alice",
    app_name  => "TMS",
    login_url => "https://tms.example.com/login",
)->to("alice\@example.com")
 ->from("noreply\@tms.example.com");

$email->{from} = 'noreply@tms.example.com';

my $msg = $email->build;
my @lines = split /\r\n/, $msg;
printf "Email message: %d lines\n", scalar @lines;

# Print first 20 lines
printf "%s\n", $_ for @lines[0..19];
printf "...\n";
```

---

## Step 330: Capstone — Full CGI Blog Application

```perl
#!/usr/bin/perl
# blog.cgi — Complete CGI blog application
use strict;
use warnings;
use DBI;
use JSON::PP;
use Digest::SHA qw(sha256_hex);

# Setup DB
my $dbh = DBI->connect("dbi:SQLite::memory:", "", "", {
    RaiseError => 1, AutoCommit => 1, sqlite_unicode => 1 });

for my $sql (split /;/, q{
CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT UNIQUE, password TEXT, role TEXT DEFAULT 'author');
CREATE TABLE posts (id INTEGER PRIMARY KEY AUTOINCREMENT, author_id INTEGER, title TEXT, slug TEXT UNIQUE,
    body TEXT, published INTEGER DEFAULT 0, created_at INTEGER DEFAULT (strftime('%s','now')));
CREATE TABLE comments (id INTEGER PRIMARY KEY AUTOINCREMENT, post_id INTEGER, author TEXT, body TEXT,
    approved INTEGER DEFAULT 0, created_at INTEGER DEFAULT (strftime('%s','now')));
}) {
    my $s = $sql; $s =~ s/^\s+|\s+$//g; $dbh->do($s) if $s;
}

# Seed
$dbh->do("INSERT INTO users (username, password, role) VALUES ('alice', ?, 'admin')", undef, sha256_hex("password123"));
$dbh->do("INSERT INTO users (username, password, role) VALUES ('bob', ?, 'author')", undef, sha256_hex("password456"));

my @posts_seed = (
    [1, "Hello World", "hello-world", "This is the first post.", 1],
    [1, "Perl is Great", "perl-is-great", "Perl is a wonderful language.", 1],
    [2, "My First Post", "my-first-post", "Bob's first post here.", 1],
    [2, "Draft Post", "draft-post", "Not published yet.", 0],
);
for my $p (@posts_seed) {
    $dbh->do("INSERT INTO posts (author_id,title,slug,body,published) VALUES (?,?,?,?,?)", undef, @$p);
}

$dbh->do("INSERT INTO comments (post_id, author, body, approved) VALUES (1, 'visitor', 'Great post!', 1)");
$dbh->do("INSERT INTO comments (post_id, author, body, approved) VALUES (1, 'reader', 'Very helpful!', 1)");
$dbh->do("INSERT INTO comments (post_id, author, body, approved) VALUES (2, 'spammer', 'Buy now!', 0)");

# Blog Application
{
package Blog;

sub render_page {
    my ($class, %args) = @_;
    my $title   = $args{title}   // "Blog";
    my $content = $args{content} // "";
    return <<HTML;
<!DOCTYPE html><html><head><title>$title</title></head><body>
<nav><a href="/">Home</a> | <a href="/about">About</a> | <a href="/login">Login</a></nav>
<h1>$title</h1>
$content
</body></html>
HTML
}

sub list_posts {
    my ($class, $dbh, %opts) = @_;
    my $page     = $opts{page}  // 1;
    my $per_page = 5;
    my $offset   = ($page - 1) * $per_page;
    my $tag      = $opts{tag};
    
    my $total = $dbh->selectrow_array("SELECT COUNT(*) FROM posts WHERE published=1") // 0;
    my $posts = $dbh->selectall_arrayref(
        "SELECT p.*, u.username AS author FROM posts p JOIN users u ON p.author_id=u.id
         WHERE p.published=1 ORDER BY p.created_at DESC LIMIT ? OFFSET ?",
        {Slice=>{}}, $per_page, $offset
    );
    
    for my $post (@$posts) {
        $post->{comment_count} = $dbh->selectrow_array(
            "SELECT COUNT(*) FROM comments WHERE post_id=? AND approved=1",
            undef, $post->{id}) // 0;
    }
    
    return {
        posts    => $posts,
        total    => $total,
        page     => $page,
        per_page => $per_page,
        pages    => int(($total + $per_page - 1) / $per_page) || 1,
    };
}

sub get_post {
    my ($class, $dbh, $slug) = @_;
    my $post = $dbh->selectrow_hashref(
        "SELECT p.*, u.username AS author FROM posts p JOIN users u ON p.author_id=u.id
         WHERE p.slug=? AND p.published=1",
        undef, $slug
    );
    return unless $post;
    
    $post->{comments} = $dbh->selectall_arrayref(
        "SELECT * FROM comments WHERE post_id=? AND approved=1 ORDER BY created_at",
        {Slice=>{}}, $post->{id}
    );
    
    return $post;
}

sub post_comment {
    my ($class, $dbh, $post_id, $author, $body) = @_;
    
    # Validate
    return (0, "Author required")   unless $author =~ /\S/;
    return (0, "Comment too short") unless length($body) >= 5;
    return (0, "Comment too long")  if length($body) > 2000;
    
    # Simple spam check
    return (0, "Spam detected") if $body =~ /\b(buy now|click here|free money)\b/i;
    
    $dbh->do("INSERT INTO comments (post_id, author, body, approved) VALUES (?,?,?,0)",
        undef, $post_id, $author, $body);
    
    return (1, "Comment submitted for review");
}
}

# Test the blog
printf "=== CGI Blog Application ===\n\n";

# List posts
my $result = Blog->list_posts($dbh);
printf "Posts: %d/%d (page %d/%d)\n",
    scalar @{$result->{posts}}, $result->{total}, $result->{page}, $result->{pages};

printf "\nPublished posts:\n";
for my $post (@{$result->{posts}}) {
    printf "  [%s] \"%s\" by %s (%d comments)\n",
        $post->{slug}, $post->{title}, $post->{author}, $post->{comment_count};
}

# Get single post
my $post = Blog->get_post($dbh, "hello-world");
printf "\nPost: %s\n", $post->{title};
printf "Comments: %d\n", scalar @{$post->{comments}};
printf "  - %s: %s\n", $_->{author}, $_->{body} for @{$post->{comments}};

# Submit comment
my ($ok, $msg) = Blog->post_comment($dbh, $post->{id}, "Visitor", "Really enjoyed this post!");
printf "\nComment: %s — %s\n", $ok ? "OK" : "FAIL", $msg;

my ($ok2, $msg2) = Blog->post_comment($dbh, $post->{id}, "Spammer", "Buy now click here free money!");
printf "Spam: %s — %s\n", $ok2 ? "OK" : "BLOCKED", $msg2;

# Render HTML page
my $list_html = "";
for my $p (@{$result->{posts}}) {
    $list_html .= "<article><h2><a href=\"/post/$p->{slug}\">$p->{title}</a></h2>";
    $list_html .= "<p>By $p->{author} | $p->{comment_count} comments</p></article>\n";
}

my $page = Blog->render_page(title => "Blog Home", content => $list_html);
printf "\nHTML page lines: %d\n", scalar(split /\n/, $page);
printf "First 200 chars: %.200s...\n", $page;
```

---

## สรุป Part 33 — CGI Web Development

ใน Part นี้คุณได้เรียนรู้:

### CGI Fundamentals
- **CGI Basics** — Request/Response cycle, environment variables
- **HTML Forms** — Field rendering, error display, form builder
- **Cookie Management** — Set/parse/delete, security attributes
- **Session Management** — Create/rotate/destroy, flash messages
- **Template System** — Variables, loops, conditionals, helpers
- **File Uploads** — Validation, magic bytes, secure storage
- **Pagination** — Offset-based, window rendering, HTML nav
- **REST Routes** — Dynamic path params, method routing
- **Email** — MIME multipart, text/html parts, attachments
- **Capstone** — Full CGI blog with posts, comments, auth

**ถัดไป: [Part 34 — Dancer2 Web Framework](part_34.md)**
