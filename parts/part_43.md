# Part 43: Email & MIME Processing
## Steps 421-430: Email Composition, MIME, Templates, SMTP

---

## Step 421: MIME Email Builder

```perl
#!/usr/bin/perl
use strict;
use warnings;
use MIME::Base64 qw(encode_base64 decode_base64);

{
package MIME::Builder;

sub new {
    my ($class) = @_;
    return bless {
        parts    => [],
        headers  => {},
        boundary => _gen_boundary(),
    }, $class;
}

sub from    { $_[0]->{headers}{From}    = $_[1]; $_[0] }
sub to      { $_[0]->{headers}{To}      = ref $_[1] ? join(", ", @{$_[1]}) : $_[1]; $_[0] }
sub cc      { $_[0]->{headers}{Cc}      = ref $_[1] ? join(", ", @{$_[1]}) : $_[1]; $_[0] }
sub bcc     { $_[0]->{headers}{Bcc}     = ref $_[1] ? join(", ", @{$_[1]}) : $_[1]; $_[0] }
sub subject { $_[0]->{headers}{"Subject"} = $_[1]; $_[0] }
sub reply_to{ $_[0]->{headers}{"Reply-To"} = $_[1]; $_[0] }

sub text_body {
    my ($self, $text) = @_;
    push @{$self->{parts}}, { type => "text/plain", content => $text, encoding => "7bit" };
    return $self;
}

sub html_body {
    my ($self, $html) = @_;
    push @{$self->{parts}}, { type => "text/html", content => $html, encoding => "quoted-printable" };
    return $self;
}

sub attach {
    my ($self, %opts) = @_;
    push @{$self->{parts}}, {
        type        => $opts{type}     // "application/octet-stream",
        name        => $opts{name}     // "attachment",
        content     => $opts{content},
        encoding    => "base64",
        disposition => "attachment",
    };
    return $self;
}

sub inline_image {
    my ($self, %opts) = @_;
    push @{$self->{parts}}, {
        type        => $opts{type}    // "image/png",
        name        => $opts{name}    // "image",
        cid         => $opts{cid}     // "img_" . rand(999),
        content     => $opts{content},
        encoding    => "base64",
        disposition => "inline",
    };
    return $self;
}

sub build {
    my $self = shift;
    my $msg  = "";
    
    my $now  = _format_date(time());
    $self->{headers}{"Date"}         //= $now;
    $self->{headers}{"MIME-Version"} //= "1.0";
    $self->{headers}{"Message-Id"}   //= "<" . _gen_boundary() . "\@perl.mail>";
    
    my @text_parts = grep { $_->{type} =~ /^text\// } @{$self->{parts}};
    my @att_parts  = grep { ($_->{disposition}//"") eq "attachment" } @{$self->{parts}};
    my @inline_parts= grep { ($_->{disposition}//"") eq "inline" } @{$self->{parts}};
    
    # Determine content type
    my $content_type;
    if (@att_parts || @inline_parts) {
        $content_type = "multipart/mixed; boundary=\"$self->{boundary}\"";
    } elsif (@text_parts > 1) {
        $content_type = "multipart/alternative; boundary=\"$self->{boundary}\"";
    } else {
        $content_type = $text_parts[0]{type};
        $self->{headers}{"Content-Transfer-Encoding"} = $text_parts[0]{encoding};
    }
    
    $self->{headers}{"Content-Type"} = $content_type;
    
    # Headers
    for my $h (qw(Date From To Cc Subject Reply-To Message-Id MIME-Version Content-Type Content-Transfer-Encoding)) {
        $msg .= "$h: $self->{headers}{$h}\r\n" if $self->{headers}{$h};
    }
    $msg .= "\r\n";
    
    if ($content_type =~ /^multipart\//) {
        for my $part (@{$self->{parts}}) {
            $msg .= "--$self->{boundary}\r\n";
            $msg .= "Content-Type: $part->{type}";
            $msg .= "; name=\"$part->{name}\"" if $part->{name};
            $msg .= "\r\n";
            $msg .= "Content-Transfer-Encoding: $part->{encoding}\r\n" if $part->{encoding};
            if ($part->{disposition}) {
                $msg .= "Content-Disposition: $part->{disposition}";
                $msg .= "; filename=\"$part->{name}\"" if $part->{name} && $part->{disposition} eq "attachment";
                $msg .= "\r\n";
            }
            $msg .= "Content-Id: <$part->{cid}>\r\n" if $part->{cid};
            $msg .= "\r\n";
            
            if ($part->{encoding} eq "base64") {
                $msg .= encode_base64($part->{content}//"", "\r\n");
            } else {
                $msg .= ($part->{content}//"") . "\r\n";
            }
        }
        $msg .= "--$self->{boundary}--\r\n";
    } else {
        $msg .= $text_parts[0]{content} // "";
    }
    
    return $msg;
}

sub _gen_boundary { sprintf "%08x_%08x", int(rand(0xFFFFFFFF)), time() }
sub _format_date  {
    my @months = qw(Jan Feb Mar Apr May Jun Jul Aug Sep Oct Nov Dec);
    my @days   = qw(Sun Mon Tue Wed Thu Fri Sat);
    my ($s,$m,$h,$d,$mo,$y,$wd) = localtime($_[0]);
    sprintf "%s, %02d %s %04d %02d:%02d:%02d +0000",
        $days[$wd], $d, $months[$mo], $y+1900, $h, $m, $s;
}
}

package main;

printf "=== MIME Email Builder ===\n\n";

# Simple text email
my $text_email = MIME::Builder->new
    ->from("sender\@test.com")
    ->to("alice\@test.com")
    ->subject("Hello from Perl!")
    ->text_body("Hello Alice,\n\nThis is a plain text email.\n\nBest,\nBot")
    ->build;

printf "=== Simple Text Email ===\n%s\n\n", $text_email;

# HTML + text (multipart/alternative)
my $html_email = MIME::Builder->new
    ->from("app\@test.com")
    ->to(["alice\@test.com", "bob\@test.com"])
    ->subject("Your Weekly Report")
    ->text_body("Weekly Report\n==============\nItems: 42\nTotal: \$1,234.56")
    ->html_body("<h1>Weekly Report</h1><p>Items: <b>42</b></p><p>Total: <b>\$1,234.56</b></p>")
    ->build;

printf "=== HTML+Text Email (first 400 chars) ===\n%s\n\n", substr($html_email,0,400) . "...";

# Email with attachment
my $attach_email = MIME::Builder->new
    ->from("reports\@test.com")
    ->to("manager\@test.com")
    ->subject("Q4 Report Attached")
    ->html_body("<p>Please find the Q4 report attached.</p>")
    ->attach(type=>"application/pdf", name=>"q4_report.pdf", content=>"PDF_DATA_HERE")
    ->build;

printf "=== Email with Attachment ===\n%s\n", substr($attach_email,0,500) . "\n...";
```

---

## Step 422: Email Template Engine

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Email::Template;

sub new {
    my ($class, %opts) = @_;
    return bless {
        templates   => {},
        layout      => $opts{layout},
        partials    => {},
    }, $class;
}

sub add {
    my ($self, $name, %parts) = @_;
    $self->{templates}{$name} = {
        subject => $parts{subject} // "",
        text    => $parts{text}    // "",
        html    => $parts{html}    // "",
    };
}

sub add_partial {
    my ($self, $name, $content) = @_;
    $self->{partials}{$name} = $content;
}

sub render {
    my ($self, $name, %vars) = @_;
    my $tmpl = $self->{templates}{$name}
        or die "Template '$name' not found\n";
    
    return {
        subject => $self->_interpolate($tmpl->{subject}, %vars),
        text    => $self->_interpolate($tmpl->{text},    %vars),
        html    => $self->_interpolate($tmpl->{html},    %vars),
    };
}

sub _interpolate {
    my ($self, $template, %vars) = @_;
    
    # Replace {{partial:name}}
    $template =~ s/\{\{partial:(\w+)\}\}/
        $self->{partials}{$1} // ""/ge;
    
    # Replace {{if:var}}...{{/if}} blocks
    $template =~ s/\{\{if:(\w+)\}\}(.*?)\{\{\/if\}\}/
        $vars{$1} ? $2 : ""/gse;
    
    # Replace {{unless:var}}...{{/unless}} blocks
    $template =~ s/\{\{unless:(\w+)\}\}(.*?)\{\{\/unless\}\}/
        !$vars{$1} ? $2 : ""/gse;
    
    # Replace {{each:var}}...{{/each}} (simple array rendering)
    $template =~ s/\{\{each:(\w+)\}\}(.*?)\{\{\/each\}\}/
        do {
            my ($key, $tmpl_block) = ($1, $2);
            my @items = ref $vars{$key} eq "ARRAY" ? @{$vars{$key}} : ($vars{$key}//"");
            join "", map { my $item=$_; my $block=$tmpl_block; $block=~s/\{\{item\}\}/$item/g; $block } @items;
        }
    /gse;
    
    # Replace {{var}} with values
    $template =~ s/\{\{(\w+)\}\}/$vars{$1} \/\/ ""/ge;
    
    # Escape for HTML ({{html:var}})
    $template =~ s/\{\{html:(\w+)\}\}/do { my $v=$vars{$1}//""; $v=~s\/[&<>"']\/{"&"=>"&amp;","<"=>"&lt;",">"=>"&gt;",'"'=>"&quot;","'"=>"&#39;"}->{$&}\/ge; $v }/ge;
    
    return $template;
}
}

package main;

printf "=== Email Template Engine ===\n\n";

my $et = Email::Template->new;

# Add partials
$et->add_partial("footer_text", "\n\n---\nThis email was sent by Perl App.\nUnsubscribe: {{unsubscribe_url}}");
$et->add_partial("footer_html", '<p style="color:#666;font-size:12px">Sent by Perl App. <a href="{{unsubscribe_url}}">Unsubscribe</a></p>');

# Welcome email
$et->add("welcome",
    subject => "Welcome to {{app_name}}, {{first_name}}!",
    text    => "Hi {{first_name}},\n\nWelcome to {{app_name}}!\n\nYour account is ready.\nUsername: {{username}}\n\nGet started: {{login_url}}{{partial:footer_text}}",
    html    => '<h1>Welcome, {{html:first_name}}!</h1><p>Your account on <b>{{app_name}}</b> is ready.</p><p><a href="{{login_url}}">Log in now</a></p>{{partial:footer_html}}',
);

# Password reset
$et->add("password_reset",
    subject => "Reset your {{app_name}} password",
    text    => "Hi {{first_name}},\n\nClick to reset your password:\n{{reset_url}}\n\nThis link expires in {{expires_in}} minutes.\n\n{{unless:requested}}If you did not request this, ignore this email.{{/unless}}",
    html    => '<p>Hi {{html:first_name}},</p><p><a href="{{reset_url}}">Reset Password</a></p><p>Expires in {{expires_in}} minutes.</p>',
);

# Order confirmation
$et->add("order_confirmation",
    subject => "Order #{{order_id}} Confirmed",
    text    => "Order #{{order_id}}\nDate: {{order_date}}\n\nItems:\n{{each:items}}  - {{item}}\n{{/each}}\nTotal: {{currency}}{{total}}\n",
    html    => '<h2>Order #{{order_id}}</h2>{{each:items}}<li>{{item}}</li>{{/each}}<p>Total: {{currency}}{{total}}</p>',
);

# Render welcome
my $w = $et->render("welcome",
    first_name    => "Alice",
    app_name      => "PerlApp",
    username      => "alice42",
    login_url     => "https://app.example.com/login",
    unsubscribe_url => "https://app.example.com/unsub",
);
printf "Welcome email:\n  Subject: %s\n  Text:\n%s\n\n",
    $w->{subject}, join("    ", map { "    $_\n" } split /\n/, $w->{text});

# Render password reset
my $pr = $et->render("password_reset",
    first_name  => "Bob",
    app_name    => "PerlApp",
    reset_url   => "https://app.example.com/reset?token=abc123",
    expires_in  => 30,
    requested   => 0,
    unsubscribe_url => "https://app.example.com/unsub",
);
printf "Password reset:\n  Subject: %s\n  Text (first 200): %s...\n\n",
    $pr->{subject}, substr($pr->{text},0,200);

# Render order
my $ord = $et->render("order_confirmation",
    order_id   => "ORD-2024-0042",
    order_date => "2024-01-08",
    items      => ["Perl Cookbook (x1)", "Learning Perl (x2)", "Moose T-shirt (x1)"],
    total      => "89.97",
    currency   => "\$",
);
printf "Order confirmation:\n  Subject: %s\n  Text:\n%s\n",
    $ord->{subject}, $ord->{text};
```

---

## Step 423: Email Validation & Sanitization

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Email::Validator;

# RFC 5321 / RFC 5322 simplified validation
my @DISPOSABLE_DOMAINS = qw(
    mailinator.com guerrillamail.com tempmail.com throwaway.email
    10minutemail.com yopmail.com trashmail.com sharklasers.com
);

my @COMMON_TYPO_DOMAINS = (
    ["gmial.com",   "gmail.com"],
    ["gamil.com",   "gmail.com"],
    ["hotmial.com", "hotmail.com"],
    ["yahooo.com",  "yahoo.com"],
    ["outloook.com","outlook.com"],
);

sub validate {
    my ($class, $email) = @_;
    my @errors;
    
    # Basic format
    $email = lc($email // "");
    $email =~ s/^\s+|\s+$//g;
    
    push @errors, "empty" unless $email;
    return (0, \@errors) unless $email;
    
    push @errors, "too_long"    if length($email) > 254;
    push @errors, "missing_at"  unless $email =~ /\@/;
    return (0, \@errors) if @errors;
    
    my ($local, $domain) = split /\@/, $email, 2;
    
    push @errors, "empty_local"       unless $local;
    push @errors, "local_too_long"    if length($local) > 64;
    push @errors, "invalid_local"     unless $local =~ /^[a-zA-Z0-9._+\-]+$/;
    push @errors, "double_dot_local"  if $local =~ /\.\./;
    push @errors, "dot_at_start"      if $local =~ /^\./;
    push @errors, "dot_at_end"        if $local =~ /\.$/;
    
    push @errors, "empty_domain"      unless $domain;
    push @errors, "domain_too_long"   if length($domain // "") > 253;
    push @errors, "invalid_domain"    unless ($domain//"") =~ /^[a-zA-Z0-9._\-]+\.[a-zA-Z]{2,}$/;
    push @errors, "ip_literal"        if ($domain//"") =~ /^\[/;
    
    return (@errors ? 0 : 1, \@errors);
}

sub is_disposable {
    my ($class, $email) = @_;
    my (undef, $domain) = split /\@/, lc($email // ""), 2;
    return grep { $_ eq ($domain//"") } @DISPOSABLE_DOMAINS;
}

sub suggest_correction {
    my ($class, $email) = @_;
    my ($local, $domain) = split /\@/, lc($email//""), 2;
    for my $pair (@COMMON_TYPO_DOMAINS) {
        return "$local\@$pair->[1]" if ($domain//"") eq $pair->[0];
    }
    return undef;
}

sub normalize {
    my ($class, $email) = @_;
    $email = lc($email // "");
    $email =~ s/^\s+|\s+$//g;
    
    my ($local, $domain) = split /\@/, $email, 2;
    
    # Gmail: remove dots and +tags from local
    if (($domain//"") eq "gmail.com") {
        $local =~ s/\.//g;
        $local =~ s/\+.*$//;
    }
    # Outlook/Hotmail: remove +tags
    elsif (($domain//"") =~ /outlook\.com|hotmail\.com|live\.com/) {
        $local =~ s/\+.*$//;
    }
    
    return "$local\@$domain";
}
}

package main;

printf "=== Email Validation & Sanitization ===\n\n";

my @test_emails = (
    'alice@example.com',
    'alice+tag@gmail.com',
    'invalid-email',
    '@missing.local',
    'a' x 65 . '@domain.com',
    'user@.domain.com',
    'user..double@domain.com',
    '.starts@domain.com',
    'user@domain',
    'user@mailinator.com',
    'user@gmial.com',
    'valid+tag@subdomain.example.org',
    'user@domain.c',
    '',
    'UPPERCASE@DOMAIN.COM',
);

printf "%-40s %-5s  %s\n", "Email", "Valid", "Issues / Suggestion";
printf "%s\n", "-" x 80;
for my $email (@test_emails) {
    my ($ok, $errors) = Email::Validator->validate($email);
    my $disposable    = Email::Validator->is_disposable($email);
    my $suggestion    = Email::Validator->suggest_correction($email);
    
    my $note = "";
    $note = "errors: " . join(", ", @$errors) if !$ok;
    $note .= " [DISPOSABLE]" if $disposable;
    $note .= " => did you mean: $suggestion?" if $suggestion;
    
    printf "%-40s %-5s  %s\n",
        substr($email || "(empty)", 0, 40),
        $ok ? "YES" : "NO",
        $note;
}

# Normalization
printf "\n--- Email Normalization ---\n";
my @to_normalize = (
    'Alice.Smith+newsletter@Gmail.Com',
    'Bob+spam@Hotmail.com',
    'carol@yahoo.com',
);
for my $e (@to_normalize) {
    printf "  %-40s => %s\n", $e, Email::Validator->normalize($e);
}
```

---

## Step 424: Email Parser

```perl
#!/usr/bin/perl
use strict;
use warnings;
use MIME::Base64 qw(decode_base64);

{
package Email::Parser;

sub parse {
    my ($class, $raw) = @_;
    my @lines = split /\r?\n/, $raw;
    
    my (%headers, @body_lines);
    my $in_body = 0;
    my $prev_header = "";
    
    for my $line (@lines) {
        if (!$in_body) {
            if ($line eq "") {
                $in_body = 1;
                next;
            }
            if ($line =~ /^\s+/ && $prev_header) {
                # Header continuation
                $headers{lc $prev_header} .= " " . $line;
                $headers{lc $prev_header} =~ s/^\s+//;
                next;
            }
            if ($line =~ /^([\w\-]+):\s*(.*)$/) {
                $prev_header = $1;
                $headers{lc $1} = $2;
            }
        } else {
            push @body_lines, $line;
        }
    }
    
    my $body = join("\n", @body_lines);
    my $ct   = $headers{"content-type"} // "text/plain";
    my ($mime_type, %params) = _parse_content_type($ct);
    
    my $email = {
        headers   => \%headers,
        mime_type => $mime_type,
        params    => \%params,
    };
    
    if ($mime_type =~ /^multipart\//) {
        $email->{parts} = $class->_parse_multipart($body, $params{boundary});
    } else {
        my $enc = $headers{"content-transfer-encoding"} // "7bit";
        $email->{body} = $class->_decode_body($body, $enc);
    }
    
    return $email;
}

sub _parse_multipart {
    my ($class, $body, $boundary) = @_;
    return [] unless $boundary;
    
    my @raw_parts = split /--\Q$boundary\E(?:--)?/, $body;
    my @parts;
    
    for my $raw (@raw_parts) {
        $raw =~ s/^\r?\n//;
        next unless $raw =~ /\S/;
        next if $raw =~ /^--/;
        
        my $part = $class->parse($raw);
        push @parts, $part if $part;
    }
    return \@parts;
}

sub _decode_body {
    my ($class, $body, $encoding) = @_;
    $encoding = lc($encoding // "7bit");
    if ($encoding eq "base64") {
        return decode_base64($body);
    } elsif ($encoding eq "quoted-printable") {
        $body =~ s/=\r?\n//g;
        $body =~ s/=([0-9A-Fa-f]{2})/chr hex $1/ge;
        return $body;
    }
    return $body;
}

sub _parse_content_type {
    my $ct = shift;
    my ($type, @params) = split /\s*;\s*/, $ct;
    $type =~ s/^\s+|\s+$//g;
    my %p;
    for my $param (@params) {
        if ($param =~ /^(\w+)="?(.*?)"?$/) {
            $p{lc $1} = $2;
        }
    }
    return (lc $type, %p);
}

sub subject { $_[0]{headers}{"subject"} }
sub from    { $_[0]{headers}{"from"} }
sub to      { $_[0]{headers}{"to"} }
sub date    { $_[0]{headers}{"date"} }
sub body    { $_[0]{body} }
}

package main;

printf "=== Email Parser ===\n\n";

# Sample raw email
my $raw_simple = <<'END';
From: sender@example.com
To: alice@example.com
Subject: Hello World
Date: Mon, 08 Jan 2024 10:00:00 +0000
Content-Type: text/plain
Content-Transfer-Encoding: 7bit
MIME-Version: 1.0

Hello Alice,

This is a simple test email.

Best,
Bob
END

my $parsed = Email::Parser->parse($raw_simple);
printf "Simple email:\n";
printf "  From:    %s\n", $parsed->{headers}{from};
printf "  To:      %s\n", $parsed->{headers}{to};
printf "  Subject: %s\n", $parsed->{headers}{subject};
printf "  Type:    %s\n", $parsed->{mime_type};
printf "  Body:    %s\n\n", substr($parsed->{body}//"-",0,60);

# Multipart email
my $boundary = "ABC123";
my $raw_multi = <<"END";
From: system\@app.com
To: user\@example.com
Subject: Multipart Test
MIME-Version: 1.0
Content-Type: multipart/alternative; boundary="$boundary"

--$boundary
Content-Type: text/plain
Content-Transfer-Encoding: 7bit

This is the plain text version.

--$boundary
Content-Type: text/html
Content-Transfer-Encoding: quoted-printable

<h1>HTML Version</h1><p>This is the =
HTML part.</p>

--$boundary--
END

my $mp = Email::Parser->parse($raw_multi);
printf "Multipart email:\n";
printf "  Parts: %d\n", scalar @{$mp->{parts}//[]};
for my $i (0..$#{$mp->{parts}//[]}) {
    my $part = $mp->{parts}[$i];
    printf "  Part %d: type=%s body=%s\n",
        $i+1, $part->{mime_type}, substr($part->{body}//"-",0,40);
}

# QP decoding
my $qp_body = "Subject: Test =\r\nThis is a long line\r\nPrice: =C2=A3100 and =E2=82=AC50";
(my $decoded = $qp_body) =~ s/=\r?\n//g;
$decoded =~ s/=([0-9A-Fa-f]{2})/chr hex $1/ge;
printf "\nQP decoded: %s\n", $decoded;
```

---

## Step 425: SMTP Client Simulation

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package SMTP::Client;

sub new {
    my ($class, %opts) = @_;
    return bless {
        host     => $opts{host}     // "localhost",
        port     => $opts{port}     // 25,
        tls      => $opts{tls}      // 0,
        username => $opts{username},
        password => $opts{password},
        timeout  => $opts{timeout}  // 30,
        debug    => $opts{debug}    // 0,
        _log     => [],
    }, $class;
}

sub connect {
    my $self = shift;
    $self->_log("CONNECT $self->{host}:$self->{port}");
    $self->_log("<- 220 $self->{host} ESMTP Postfix (PerlApp)");
    return $self;
}

sub ehlo {
    my ($self, $domain) = @_;
    $domain //= "localhost";
    $self->_log("-> EHLO $domain");
    $self->_log("<- 250-$self->{host}");
    $self->_log("<- 250-PIPELINING");
    $self->_log("<- 250-SIZE 10240000");
    $self->_log("<- 250-STARTTLS") if $self->{tls};
    $self->_log("<- 250-AUTH PLAIN LOGIN");
    $self->_log("<- 250 8BITMIME");
    return $self;
}

sub auth {
    my ($self, $user, $pass) = @_;
    $self->_log("-> AUTH LOGIN");
    $self->_log("<- 334 VXNlcm5hbWU6");  # "Username:"
    $self->_log("-> " . ("*" x length($user)));
    $self->_log("<- 334 UGFzc3dvcmQ6");  # "Password:"
    $self->_log("-> " . ("*" x length($pass)));
    $self->_log("<- 235 2.7.0 Authentication successful");
    return $self;
}

sub mail_from {
    my ($self, $from) = @_;
    $self->_log("-> MAIL FROM:<$from>");
    $self->_log("<- 250 2.1.0 Ok");
    return $self;
}

sub rcpt_to {
    my ($self, @recipients) = @_;
    for my $to (@recipients) {
        $self->_log("-> RCPT TO:<$to>");
        if ($to =~ /invalid/i || $to !~ /\@/) {
            $self->_log("<- 550 5.1.1 $to User unknown");
        } else {
            $self->_log("<- 250 2.1.5 Ok");
        }
    }
    return $self;
}

sub data {
    my ($self, $message) = @_;
    $self->_log("-> DATA");
    $self->_log("<- 354 End data with <CR><LF>.<CR><LF>");
    $self->_log("-> [message data: " . length($message) . " bytes]");
    $self->_log("-> .");
    $self->_log("<- 250 2.0.0 Ok: queued as " . sprintf "%08X", int rand 0xFFFFFFFF);
    return $self;
}

sub quit {
    my $self = shift;
    $self->_log("-> QUIT");
    $self->_log("<- 221 2.0.0 Bye");
    return $self;
}

sub send_email {
    my ($self, %opts) = @_;
    
    $self->connect;
    $self->ehlo("myapp.com");
    $self->auth($self->{username}, $self->{password})
        if $self->{username};
    $self->mail_from($opts{from});
    $self->rcpt_to(ref($opts{to}) ? @{$opts{to}} : $opts{to});
    $self->data($opts{message});
    $self->quit;
    
    return 1;
}

sub log { $_[0]->{_log} }
sub _log { push @{$_[0]->{_log}}, $_[1]; print "$_[1]\n" if $_[0]->{debug} }
}

{
package SMTP::Pool;
# Connection pool for SMTP

sub new {
    my ($class, %opts) = @_;
    return bless {
        config      => \%opts,
        connections => [],
        in_use      => {},
        max_size    => $opts{max_size} // 5,
        sent        => 0,
    }, $class;
}

sub acquire {
    my $self = shift;
    my ($conn) = grep { !$self->{in_use}{$_} } @{$self->{connections}};
    
    unless ($conn) {
        if (@{$self->{connections}} < $self->{max_size}) {
            $conn = SMTP::Client->new(%{$self->{config}});
            $conn->connect->ehlo;
            push @{$self->{connections}}, $conn;
        } else {
            die "Connection pool exhausted\n";
        }
    }
    
    $self->{in_use}{$conn} = 1;
    return $conn;
}

sub release {
    my ($self, $conn) = @_;
    delete $self->{in_use}{$conn};
}

sub send {
    my ($self, %opts) = @_;
    my $conn = $self->acquire;
    eval {
        $conn->mail_from($opts{from});
        $conn->rcpt_to(ref $opts{to} ? @{$opts{to}} : $opts{to});
        $conn->data($opts{message});
        $self->{sent}++;
    };
    $self->release($conn);
    die $@ if $@;
}

sub stats { { connections => scalar @{$_[0]->{connections}}, sent => $_[0]->{sent} } }
}

package main;

printf "=== SMTP Client ===\n\n";

# Simulate sending
my $smtp = SMTP::Client->new(
    host     => "smtp.gmail.com",
    port     => 587,
    tls      => 1,
    username => "myapp\@gmail.com",
    password => "app_password",
    debug    => 1,
);

printf "--- SMTP Session ---\n";
$smtp->send_email(
    from    => "myapp\@gmail.com",
    to      => ["alice\@example.com", "bob\@example.com", "invalid_no_at"],
    message => "From: myapp\@gmail.com\r\nTo: alice\@example.com\r\nSubject: Test\r\n\r\nHello!",
);

printf "\nLog entries: %d\n", scalar @{$smtp->log};

# Pool simulation
printf "\n--- SMTP Pool (3 connections, 6 sends) ---\n";
my $pool = SMTP::Pool->new(host=>"smtp.example.com", port=>25, max_size=>3);
for my $i (1..6) {
    eval { $pool->send(from=>"app\@test.com", to=>"user${i}\@test.com", message=>"msg$i") };
    printf "  Send $i: %s\n", $@ ? "ERROR: $@" : "OK";
}
my $ps = $pool->stats;
printf "Pool stats: connections=%d sent=%d\n", $ps->{connections}, $ps->{sent};
```

---

## Step 426: Email Queue & Delivery

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Email::Queue;

sub new {
    my ($class, %opts) = @_;
    return bless {
        queue      => [],
        sent       => [],
        failed     => [],
        max_retry  => $opts{max_retry}  // 3,
        retry_wait => $opts{retry_wait} // 5,
        transport  => $opts{transport}  // sub { 1 },
        throttle   => $opts{throttle}   // 10,  # per minute
        _sent_ts   => [],
    }, $class;
}

sub enqueue {
    my ($self, %email) = @_;
    push @{$self->{queue}}, {
        id       => sprintf("em_%08x", int rand 0xFFFFFFFF),
        %email,
        attempts => 0,
        status   => "pending",
        created  => time(),
        next_try => time(),
    };
}

sub flush {
    my $self = shift;
    my @to_send = grep { $_->{status} eq "pending" && $_->{next_try} <= time() }
                  @{$self->{queue}};
    
    for my $email (@to_send) {
        # Rate limiting
        $self->{_sent_ts} = [grep { $_ > time()-60 } @{$self->{_sent_ts}}];
        if (@{$self->{_sent_ts}} >= $self->{throttle}) {
            printf "  THROTTLED: %d/%d per minute\n",
                scalar @{$self->{_sent_ts}}, $self->{throttle};
            last;
        }
        
        $email->{attempts}++;
        my $ok = eval { $self->{transport}->($email) };
        
        if ($ok && !$@) {
            $email->{status} = "sent";
            $email->{sent_at} = time();
            push @{$self->{sent}}, $email;
            push @{$self->{_sent_ts}}, time();
            printf "  SENT: [%s] to=%s\n", $email->{id}, $email->{to};
        } else {
            my $err = $@ || "transport returned false";
            if ($email->{attempts} >= $self->{max_retry}) {
                $email->{status}   = "failed";
                $email->{last_err} = $err;
                push @{$self->{failed}}, $email;
                printf "  FAILED: [%s] err=%s\n", $email->{id}, substr($err,0,40);
            } else {
                $email->{next_try} = time() + $self->{retry_wait} * $email->{attempts};
                printf "  RETRY: [%s] attempt=%d next_try_in=%ds\n",
                    $email->{id}, $email->{attempts}, $self->{retry_wait}*$email->{attempts};
            }
        }
    }
}

sub stats {
    return {
        pending => scalar(grep { $_->{status} eq "pending" } @{$_[0]->{queue}}),
        sent    => scalar @{$_[0]->{sent}},
        failed  => scalar @{$_[0]->{failed}},
        total   => scalar @{$_[0]->{queue}},
    };
}
}

package main;

printf "=== Email Queue & Delivery ===\n\n";

my $call_count = 0;
my $eq = Email::Queue->new(
    max_retry => 3,
    retry_wait=> 1,
    throttle  => 5,  # 5 per minute for demo
    transport => sub {
        my $email = shift;
        $call_count++;
        # Simulate 30% failure on first attempt
        die "SMTP error: connection timeout\n"
            if $email->{attempts} < 2 && $email->{to} =~ /^(bob|dave)/;
        return 1;
    },
);

# Enqueue emails
for my $to (qw(alice bob carol dave eve frank george henry iris jack)) {
    $eq->enqueue(
        to      => "${to}\@example.com",
        from    => "app\@example.com",
        subject => "Newsletter - January 2024",
        body    => "Hello $to, here is your newsletter...",
    );
}

printf "Queued: %d emails\n\n", scalar @{$eq->{queue}};

# First flush
printf "First flush:\n";
$eq->flush;

# Second flush (retries)
printf "\nSecond flush (retries):\n";
$eq->flush;

# Stats
my $stats = $eq->stats;
printf "\nStats: pending=%d sent=%d failed=%d total=%d\n",
    $stats->{pending}, $stats->{sent}, $stats->{failed}, $stats->{total};
printf "Transport calls: %d\n", $call_count;
```

---

## Step 427: Newsletter & Bulk Email

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Newsletter;

sub new {
    my ($class, %opts) = @_;
    return bless {
        name        => $opts{name}      // "Newsletter",
        from        => $opts{from}      // "news\@example.com",
        subscribers => [],
        campaigns   => {},
        unsubscribed=> {},
        stats       => {},
    }, $class;
}

sub subscribe {
    my ($self, %user) = @_;
    return 0 if $self->{unsubscribed}{$user{email}};
    return 0 if grep { $_->{email} eq $user{email} } @{$self->{subscribers}};
    push @{$self->{subscribers}}, {
        email    => $user{email},
        name     => $user{name} // "Subscriber",
        tags     => $user{tags} // [],
        joined   => time(),
        opens    => 0,
        clicks   => 0,
    };
    return 1;
}

sub unsubscribe {
    my ($self, $email) = @_;
    $self->{unsubscribed}{$email} = 1;
    @{$self->{subscribers}} = grep { $_->{email} ne $email } @{$self->{subscribers}};
}

sub segment {
    my ($self, $filter) = @_;
    return [grep { $filter->($_) } @{$self->{subscribers}}];
}

sub create_campaign {
    my ($self, %opts) = @_;
    my $id = "camp_" . sprintf "%04d", scalar keys %{$self->{campaigns}};
    $self->{campaigns}{$id} = {
        id       => $id,
        subject  => $opts{subject},
        html     => $opts{html},
        text     => $opts{text},
        tags     => $opts{tags},  # Target specific tags
        created  => time(),
        status   => "draft",
    };
    return $id;
}

sub send_campaign {
    my ($self, $campaign_id, %opts) = @_;
    my $camp = $self->{campaigns}{$campaign_id}
        or die "Campaign not found\n";
    
    # Get target list
    my @targets;
    if ($camp->{tags}) {
        @targets = @{$self->segment(sub {
            my $sub = shift;
            grep { my $tag=$_; grep { $_ eq $tag } @{$sub->{tags}} } @{$camp->{tags}}
        })};
    } else {
        @targets = @{$self->{subscribers}};
    }
    
    my ($sent, $skipped) = (0, 0);
    for my $sub (@targets) {
        # Personalize
        (my $personalized = $camp->{html}//"") =~ s/\{\{name\}\}/$sub->{name}/g;
        
        # Add tracking pixel and unsubscribe link
        my $unsub_url = "https://app.example.com/unsub?email=$sub->{email}&cid=$campaign_id";
        $personalized .= "<img src='https://track.example.com/open/$campaign_id/$sub->{email}' width='1' height='1'>";
        $personalized .= "<br><a href='$unsub_url'>Unsubscribe</a>";
        
        $sent++;
    }
    
    $camp->{status} = "sent";
    $camp->{sent_at} = time();
    $camp->{recipients} = $sent;
    
    $self->{stats}{$campaign_id} = {
        sent    => $sent,
        opens   => 0,
        clicks  => 0,
        unsubs  => 0,
        bounces => 0,
    };
    
    return ($sent, $skipped);
}

sub track_open   { $_[0]->{stats}{$_[1]}{opens}++  if $_[0]->{stats}{$_[1]} }
sub track_click  { $_[0]->{stats}{$_[1]}{clicks}++ if $_[0]->{stats}{$_[1]} }
sub track_unsub  { $_[0]->{stats}{$_[1]}{unsubs}++; $_[0]->unsubscribe($_[2]) }
sub track_bounce { $_[0]->{stats}{$_[1]}{bounces}++ }

sub campaign_stats {
    my ($self, $campaign_id) = @_;
    my $s    = $self->{stats}{$campaign_id} or return {};
    my $sent = $s->{sent} || 1;
    return {
        %$s,
        open_rate  => sprintf("%.1f%%", 100 * $s->{opens}  / $sent),
        click_rate => sprintf("%.1f%%", 100 * $s->{clicks} / $sent),
        unsub_rate => sprintf("%.1f%%", 100 * $s->{unsubs} / $sent),
    };
}

sub subscriber_count { scalar @{$_[0]->{subscribers}} }
}

package main;

printf "=== Newsletter & Bulk Email ===\n\n";

my $nl = Newsletter->new(name => "Perl Weekly", from => "news\@perlweekly.com");

# Subscribe
my @users = (
    { email => "alice\@test.com",  name => "Alice",  tags => ["premium","tech"] },
    { email => "bob\@test.com",    name => "Bob",    tags => ["free","tech"]    },
    { email => "carol\@test.com",  name => "Carol",  tags => ["premium","biz"]  },
    { email => "dave\@test.com",   name => "Dave",   tags => ["free","tech"]    },
    { email => "eve\@test.com",    name => "Eve",    tags => ["premium","tech"] },
);

$nl->subscribe(%$_) for @users;
printf "Subscribers: %d\n", $nl->subscriber_count;

# Segment
my $premium = $nl->segment(sub { grep { $_ eq "premium" } @{$_[0]{tags}} });
printf "Premium segment: %d\n", scalar @$premium;

# Create and send campaigns
my $all_camp = $nl->create_campaign(
    subject => "Perl Weekly #42 — New Features",
    html    => "<h1>Hello {{name}}!</h1><p>This week in Perl: Moose 2.5 released...</p>",
    text    => "Hello {{name}}! This week in Perl...",
);

my $premium_camp = $nl->create_campaign(
    subject => "Premium: Exclusive Perl Tutorial",
    html    => "<h1>{{name}}, your exclusive content is here!</h1>",
    tags    => ["premium"],
);

printf "\nSending campaigns:\n";
my ($sent1, $skip1) = $nl->send_campaign($all_camp);
printf "  All-subscribers: sent=%d\n", $sent1;

my ($sent2, $skip2) = $nl->send_campaign($premium_camp);
printf "  Premium-only: sent=%d\n\n", $sent2;

# Track engagement
$nl->track_open($all_camp) for 1..3;
$nl->track_click($all_camp) for 1..2;
$nl->track_unsub($all_camp, "bob\@test.com");
$nl->track_bounce($all_camp);

my $stats = $nl->campaign_stats($all_camp);
printf "Campaign stats (all_camp):\n";
printf "  sent=%d opens=%d clicks=%d unsubs=%d bounces=%d\n",
    $stats->{sent}, $stats->{opens}, $stats->{clicks}, $stats->{unsubs}, $stats->{bounces};
printf "  open_rate=%s click_rate=%s unsub_rate=%s\n",
    $stats->{open_rate}, $stats->{click_rate}, $stats->{unsub_rate};

printf "\nSubscribers after unsub: %d\n", $nl->subscriber_count;
```

---

## Step 428-430: Email Best Practices, SPF/DKIM Simulation, Capstone

```perl
#!/usr/bin/perl
# email_advanced.pl — SPF simulation, DKIM signing, and delivery capstone
use strict;
use warnings;
use Digest::SHA qw(hmac_sha256 sha256_hex);
use MIME::Base64 qw(encode_base64url);

# --- SPF Record Checker (simulation) ---
{
package SPF::Checker;

my %spf_records = (
    "example.com"    => "v=spf1 ip4:192.168.1.0/24 include:gmail.com ~all",
    "myapp.com"      => "v=spf1 ip4:10.0.0.0/8 ip4:203.0.113.5 -all",
    "nospf.com"      => "",
);

sub check {
    my ($class, %opts) = @_;
    my $domain = $opts{from_domain} // "";
    my $ip     = $opts{sender_ip}   // "";
    
    my $record = $spf_records{$domain} or return { result => "none", reason => "no_spf_record" };
    
    # Parse mechanisms
    my @mechanisms = split /\s+/, $record;
    shift @mechanisms;  # Remove "v=spf1"
    
    for my $mech (@mechanisms) {
        if ($mech =~ /^ip4:(.+)$/) {
            my $cidr = $1;
            return { result => "pass", mechanism => $mech }
                if _ip_in_range($ip, $cidr);
        } elsif ($mech =~ /^([~+-?])all$/) {
            my %qualifier = ( "~" => "softfail", "-" => "fail", "+" => "pass", "?" => "neutral" );
            return { result => $qualifier{$1} // "neutral", mechanism => $mech };
        }
    }
    return { result => "neutral", reason => "no_matching_mechanism" };
}

sub _ip_in_range {
    my ($ip, $cidr) = @_;
    return ($ip eq $cidr) unless $cidr =~ m{/};
    my ($network, $bits) = split /\//, $cidr;
    # Simplified check for demo
    my $net_prefix = join ".", (split /\./, $network)[0..int($bits/8)-1];
    my $ip_prefix  = join ".", (split /\./, $ip)[0..int($bits/8)-1];
    return $net_prefix eq $ip_prefix;
}
}

# --- DKIM Signer ---
{
package DKIM::Signer;

sub new {
    my ($class, %opts) = @_;
    return bless {
        domain     => $opts{domain}    // die("domain required"),
        selector   => $opts{selector}  // "default",
        private_key=> $opts{private_key}// "PRIVATE_KEY_BYTES",
        headers    => $opts{headers}   // [qw(from to subject date)],
    }, $class;
}

sub sign {
    my ($self, $email_headers, $body) = @_;
    
    # Canonicalize body (relaxed)
    my $canon_body = $body;
    $canon_body =~ s/\s+$//gm;
    $canon_body .= "\r\n" unless $canon_body =~ /\r\n$/;
    
    # Body hash
    my $body_hash = encode_base64url(Digest::SHA::sha256($canon_body), "");
    
    # Header canonicalization
    my @signed_headers;
    for my $h (@{$self->{headers}}) {
        my $val = $email_headers->{lc $h} // next;
        push @signed_headers, lc($h) . ":" . $val;
    }
    
    # DKIM signature header (partial)
    my $dkim_base = sprintf
        "v=1; a=rsa-sha256; c=relaxed/relaxed; d=%s; s=%s; h=%s; bh=%s; b=",
        $self->{domain}, $self->{selector},
        join(":", @{$self->{headers}}),
        $body_hash;
    
    # Sign (simulated with HMAC-SHA256)
    my $data_to_sign = join("\r\n", @signed_headers) . "\r\n" . "dkim-signature:" . $dkim_base;
    my $signature    = encode_base64url(hmac_sha256($data_to_sign, $self->{private_key}), "");
    
    return "DKIM-Signature: " . $dkim_base . $signature;
}

sub verify_simulation {
    my ($class, $dkim_header) = @_;
    return $dkim_header =~ /^DKIM-Signature:.*b=[A-Za-z0-9\-_]+$/ ? 1 : 0;
}
}

package main;

printf "=== Email Security: SPF & DKIM ===\n\n";

# SPF checks
printf "SPF Checks:\n";
my @spf_tests = (
    { from_domain => "example.com", sender_ip => "192.168.1.50" },
    { from_domain => "example.com", sender_ip => "10.20.30.40"  },
    { from_domain => "myapp.com",   sender_ip => "203.0.113.5"  },
    { from_domain => "myapp.com",   sender_ip => "172.16.1.1"   },
    { from_domain => "nospf.com",   sender_ip => "1.2.3.4"      },
);

for my $test (@spf_tests) {
    my $result = SPF::Checker->check(%$test);
    printf "  %-15s from %-15s => %s\n",
        $test->{from_domain}, $test->{sender_ip}, $result->{result};
}

# DKIM signing
printf "\nDKIM Signing:\n";
my $signer = DKIM::Signer->new(
    domain     => "myapp.com",
    selector   => "mail2024",
    private_key=> "my-rsa-private-key-bytes",
);

my %headers = (
    from    => "sender\@myapp.com",
    to      => "alice\@example.com",
    subject => "Hello World",
    date    => "Mon, 08 Jan 2024 10:00:00 +0000",
);

my $dkim_sig = $signer->sign(\%headers, "Hello Alice!\r\nThis is a test.\r\n");
printf "  Signature: %s\n\n", substr($dkim_sig, 0, 80) . "...";

my $valid = DKIM::Signer->verify_simulation($dkim_sig);
printf "  Signature valid: %s\n\n", $valid ? "YES" : "NO";

# Full email delivery capstone
printf "=== Full Email Delivery Capstone ===\n\n";

# Build email
printf "Building multipart email:\n";
my $from = "noreply\@myapp.com";
my $to   = "customer\@gmail.com";
my $subj = "Your Order Has Shipped!";

my $text = "Dear Customer,\n\nYour order #ORD-1234 has shipped!\nTracking: TRK-9876543\n\nThank you!";
my $html = "<h2>Your Order Shipped!</h2><p>Order <b>#ORD-1234</b> is on its way!</p><p>Tracking: <b>TRK-9876543</b></p>";

# Check SPF
my $spf = SPF::Checker->check(from_domain => "myapp.com", sender_ip => "203.0.113.5");
printf "  SPF check: %s\n", $spf->{result};

# Sign with DKIM
my $sig = $signer->sign({ from=>$from, to=>$to, subject=>$subj, date=>"Mon, 08 Jan 2024 12:00:00 +0000" }, $text);
printf "  DKIM signed: %s\n", length($sig) > 0 ? "YES" : "NO";

# Send (simulation)
printf "  Delivered to MTA: SIMULATED\n";
printf "  Message ID: <msg_%08x\@myapp.com>\n", int rand 0xFFFFFFFF;
printf "\nDelivery complete: from=%s to=%s subject='%s'\n", $from, $to, $subj;
```

---

## สรุป Part 43 — Email & MIME Processing

### สิ่งที่เรียนรู้:
- **MIME Builder** — text/plain, text/html, multipart/alternative, attachments
- **Email Templates** — {{var}}, {{if:}}, {{each:}}, partials system
- **Email Validation** — RFC 5321, typo detection, normalization
- **Email Parser** — Headers, multipart parsing, QP/Base64 decoding
- **SMTP Client** — EHLO, AUTH, MAIL FROM, RCPT TO, DATA, QUIT
- **Email Queue** — Retry, throttling, delivery tracking
- **Newsletter** — Segmentation, campaigns, open/click/unsub tracking
- **SPF Simulation** — ip4 mechanism, qualifier checking
- **DKIM Signing** — Canonical headers, body hash, signature
- **Capstone** — Complete email delivery pipeline

**ถัดไป: [Part 44 — File Processing & I/O](part_44.md)**
