# Part 39: WebSockets & Real-time Perl
## Steps 381-390: Asynchronous & Event-driven Programming

---

## Step 381: AnyEvent Fundamentals

```perl
#!/usr/bin/perl
# anyevent_sim.pl — AnyEvent-style event loop simulation
# Real AnyEvent: cpan AnyEvent
use strict;
use warnings;

{
package AnyEvent::Impl;

# Global event loop state
my @timers;
my @io_watchers;
my $running = 0;
my $current_time = 0;
my @deferred;

sub new_cv {
    return bless { value => undef, signaled => 0, callbacks => [] }, "AnyEvent::CondVar";
}

sub timer {
    my ($class, %args) = @_;
    my $id = @timers + 1;
    push @timers, {
        id       => $id,
        after    => $args{after}    // 0,
        interval => $args{interval} // 0,
        cb       => $args{cb}       // sub {},
        fire_at  => $current_time + ($args{after} // 0),
        active   => 1,
    };
    return bless { id => $id }, "AnyEvent::Timer";
}

sub io {
    my ($class, %args) = @_;
    push @io_watchers, {
        fh   => $args{fh},
        poll => $args{poll} // "r",
        cb   => $args{cb}   // sub {},
    };
}

sub defer {
    my ($class, $cb) = @_;
    push @deferred, $cb;
}

sub one_event {
    $current_time++;
    
    # Run deferred
    my @todo = @deferred; @deferred = ();
    $_->() for @todo;
    
    # Fire timers
    for my $t (@timers) {
        next unless $t && $t->{active};
        next if $current_time < $t->{fire_at};
        
        $t->{cb}->();
        
        if ($t->{interval}) {
            $t->{fire_at} = $current_time + $t->{interval};
        } else {
            $t->{active} = 0;
        }
    }
    
    # Clean up inactive
    @timers = grep { $_ && $_->{active} } @timers;
}

sub run {
    my ($class, $max_steps) = @_;
    $max_steps //= 100;
    $running = 1;
    $class->one_event while $running && $max_steps-- > 0;
    $running = 0;
}

sub stop { $running = 0 }
sub time { $current_time }
}

{
package AnyEvent::CondVar;

sub new { bless { value=>undef, signaled=>0, cbs=>[] }, $_[0] }
sub send { my($self,$v)=@_; $self->{value}=$v; $self->{signaled}=1; $_->($v) for @{$self->{cbs}} }
sub recv { while(!$_[0]->{signaled}){ AnyEvent::Impl->one_event } $_[0]->{value} }
sub cb   { push @{$_[0]->{cbs}}, $_[1] }
sub ready{ $_[0]->{signaled} }
}

{
package AnyEvent::Timer;
sub cancel { my $id=$_[0]->{id}; for(@AnyEvent::Impl::timers){$_->{active}=0 if $_&&$_->{id}==$id} }
}

package main;

printf "=== AnyEvent Event Loop ===\n\n";

my @log;

# One-shot timers
AnyEvent::Impl->timer(after=>3, cb=>sub{ push @log, "t=".AnyEvent::Impl->time.": Timer 3s fired" });
AnyEvent::Impl->timer(after=>5, cb=>sub{ push @log, "t=".AnyEvent::Impl->time.": Timer 5s fired" });
AnyEvent::Impl->timer(after=>8, cb=>sub{ push @log, "t=".AnyEvent::Impl->time.": Timer 8s fired" });

# Recurring
my $count = 0;
AnyEvent::Impl->timer(after=>2, interval=>2, cb=>sub{
    $count++;
    push @log, "t=".AnyEvent::Impl->time.": Interval count=$count";
    if ($count >= 4) { AnyEvent::Impl->stop }
});

# Deferred
AnyEvent::Impl->defer(sub { push @log, "t=0: Deferred task" });

AnyEvent::Impl->run(20);

printf "%s\n", $_ for @log;
printf "\nTotal: %d events logged\n", scalar @log;

# CondVar example
printf "\n--- CondVar ---\n";
my $cv = AnyEvent::CondVar->new;

AnyEvent::Impl->timer(after=>2, cb=>sub {
    $cv->send("Result from async operation");
});

my $result = $cv->recv;
printf "Got: %s\n", $result;
```

---

## Step 382: Non-blocking HTTP Server

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Async::HTTPServer;

my @connections;
my @listeners;

sub new {
    my ($class, %opts) = @_;
    return bless {
        port     => $opts{port}    // 8080,
        host     => $opts{host}    // "0.0.0.0",
        handlers => {},
        middleware => [],
        connections => 0,
        requests  => 0,
    }, $class;
}

sub get    { $_[0]->_route("GET",    $_[1], $_[2]) }
sub post   { $_[0]->_route("POST",   $_[1], $_[2]) }
sub put    { $_[0]->_route("PUT",    $_[1], $_[2]) }
sub delete { $_[0]->_route("DELETE", $_[1], $_[2]) }

sub _route {
    my ($self, $method, $path, $handler) = @_;
    my $regex = $path;
    $regex =~ s{:(\w+)}{(?<$1>[^/]+)}g;
    push @{$self->{routes}}, { method=>$method, path=>$path, regex=>qr{^$regex$}, handler=>$handler };
    return $self;
}

sub use_middleware {
    my ($self, $mw) = @_;
    push @{$self->{middleware}}, $mw;
    return $self;
}

# Simulate handling a connection
sub handle_connection {
    my ($self, $conn_id, %req) = @_;
    $self->{connections}++;
    $self->{requests}++;
    
    my $ctx = {
        conn_id  => $conn_id,
        req      => \%req,
        res      => { status => 200, headers => {}, body => "" },
        done     => 0,
        log      => [],
    };
    
    # Run middleware
    my $next_idx = 0;
    my $run_next;
    $run_next = sub {
        if ($next_idx < @{$self->{middleware}}) {
            my $mw = $self->{middleware}[$next_idx++];
            $mw->($ctx, $run_next);
        } else {
            # Route
            $self->_dispatch($ctx);
        }
    };
    $run_next->();
    
    return $ctx->{res};
}

sub _dispatch {
    my ($self, $ctx) = @_;
    my $method = $ctx->{req}{method};
    my $path   = $ctx->{req}{path};
    
    for my $route (@{$self->{routes}//[]}) {
        next unless uc($route->{method}) eq uc($method);
        next unless $path =~ $route->{regex};
        $ctx->{req}{captures} = {%+};
        $route->{handler}->($ctx);
        return;
    }
    
    $ctx->{res} = { status => 404, body => '{"error":"not found"}' };
}

sub send_json {
    my ($self, $ctx, $data, %opts) = @_;
    require JSON::PP;
    $ctx->{res} = {
        status  => $opts{status} // 200,
        headers => { "Content-Type" => "application/json" },
        body    => JSON::PP->new->utf8->encode($data),
    };
}

sub stats {
    my $self = shift;
    return { connections => $self->{connections}, requests => $self->{requests} };
}
}

package main;

printf "=== Non-blocking HTTP Server ===\n\n";

my $server = Async::HTTPServer->new(port => 8080);

# Middleware
$server->use_middleware(sub {
    my ($ctx, $next) = @_;
    my $start = time();
    push @{$ctx->{log}}, sprintf "[%s] %s %s", time(), $ctx->{req}{method}, $ctx->{req}{path};
    $next->();
    push @{$ctx->{log}}, sprintf "  => %d (%.3fms)", $ctx->{res}{status}, (time()-$start)*1000;
});

$server->use_middleware(sub {
    my ($ctx, $next) = @_;
    $ctx->{res}{headers}{"X-Request-Id"} = sprintf "%08x", rand(0xFFFFFFFF);
    $next->();
});

# Routes
$server->get("/api/status" => sub {
    my $ctx = shift;
    $server->send_json($ctx, { status => "ok", time => time() });
});

$server->get("/api/users/:id" => sub {
    my $ctx = shift;
    my $id  = $ctx->{req}{captures}{id};
    $server->send_json($ctx, { id => $id+0, name => "User $id" });
});

$server->post("/api/users" => sub {
    my $ctx = shift;
    $server->send_json($ctx, { id => 42, created => 1 }, status => 201);
});

# Simulate 10 concurrent connections
printf "Handling requests:\n";
my @requests = (
    { conn_id => 1, method => "GET",  path => "/api/status" },
    { conn_id => 2, method => "GET",  path => "/api/users/1" },
    { conn_id => 3, method => "GET",  path => "/api/users/99" },
    { conn_id => 4, method => "POST", path => "/api/users" },
    { conn_id => 5, method => "GET",  path => "/not/found" },
);

for my $req (@requests) {
    my $res = $server->handle_connection($req->{conn_id}, %$req);
    printf "  [%d] %s %s => %d\n", $req->{conn_id}, $req->{method}, $req->{path}, $res->{status};
}

printf "\nServer stats: connections=%d requests=%d\n",
    $server->stats->{connections}, $server->stats->{requests};
```

---

## Step 383: Pub/Sub Messaging

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package PubSub;

sub new {
    return bless { channels => {}, history => {}, max_history => 100 }, $_[0];
}

sub subscribe {
    my ($self, $channel, $subscriber, $handler) = @_;
    $self->{channels}{$channel}{$subscriber} = $handler;
    return $self;
}

sub unsubscribe {
    my ($self, $channel, $subscriber) = @_;
    delete $self->{channels}{$channel}{$subscriber};
    delete $self->{channels}{$channel} unless %{$self->{channels}{$channel}};
    return $self;
}

sub publish {
    my ($self, $channel, $message, %opts) = @_;
    my $envelope = {
        channel => $channel,
        data    => $message,
        ts      => time() + $self->{_tick}++,
        id      => ++$self->{_msg_id},
    };
    
    # Store history
    push @{$self->{history}{$channel}}, $envelope;
    if (@{$self->{history}{$channel}} > $self->{max_history}) {
        shift @{$self->{history}{$channel}};
    }
    
    # Deliver
    my $subscribers = $self->{channels}{$channel} // {};
    my $delivered = 0;
    for my $sub_id (keys %$subscribers) {
        my $h = $subscribers->{$sub_id};
        next if $opts{exclude} && $sub_id eq $opts{exclude};
        eval { $h->($envelope) };
        $delivered++;
    }
    
    return $delivered;
}

sub broadcast {
    my ($self, $message) = @_;
    my $delivered = 0;
    for my $channel (keys %{$self->{channels}}) {
        $delivered += $self->publish($channel, $message);
    }
    return $delivered;
}

sub history {
    my ($self, $channel, $since_id) = @_;
    my $all = $self->{history}{$channel} // [];
    return $since_id ? [grep { $_->{id} > $since_id } @$all] : $all;
}

sub channels {
    my ($self, $with_count) = @_;
    return $with_count
        ? { map { $_ => scalar keys %{$self->{channels}{$_}} } keys %{$self->{channels}} }
        : [sort keys %{$self->{channels}}];
}

sub subscriber_count { scalar keys %{$_[0]->{channels}{$_[1]}//{}}}
}

package main;

printf "=== Pub/Sub Messaging ===\n\n";

my $ps   = PubSub->new;
my @log;

# Subscribers
$ps->subscribe("news",  "alice", sub { push @log, "Alice[news]: $_[0]{data}" });
$ps->subscribe("news",  "bob",   sub { push @log, "Bob[news]: $_[0]{data}" });
$ps->subscribe("sports","alice", sub { push @log, "Alice[sports]: $_[0]{data}" });
$ps->subscribe("tech",  "carol", sub { push @log, "Carol[tech]: $_[0]{data}" });

# Publish
printf "Publishing messages:\n";
my $d1 = $ps->publish("news",   "Perl 5.40 released!", );
my $d2 = $ps->publish("sports", "Perl won the hackathon!");
my $d3 = $ps->publish("tech",   "New Moose version out");
my $d4 = $ps->publish("news",   "New CPAN modules", exclude => "bob");
my $d5 = $ps->publish("unknown","Nobody here");

printf "  news: %d delivered\n", $d1;
printf "  sports: %d delivered\n", $d2;
printf "  tech: %d delivered\n", $d3;
printf "  news (excl bob): %d delivered\n", $d4;
printf "  unknown: %d delivered\n", $d5;

printf "\nMessages received:\n";
printf "  %s\n", $_ for @log;

# Channels info
printf "\nChannels: %s\n", join(", ", @{$ps->channels});
my $counts = $ps->channels(1);
printf "  %s: %d subscribers\n", $_, $counts->{$_} for sort keys %$counts;

# History
printf "\nNews history: %d messages\n", scalar @{$ps->history("news")};

# Unsubscribe
$ps->unsubscribe("news", "alice");
printf "\nAfter alice unsubs from news: %d subscribers\n", $ps->subscriber_count("news");

# EventBus pattern (type-based routing)
{
package EventBus;

sub new { bless { handlers => {} }, $_[0] }

sub on {
    my ($self, $event_type, $handler) = @_;
    push @{$self->{handlers}{$event_type}}, $handler;
    return $self;
}

sub emit {
    my ($self, $event_type, %data) = @_;
    my $event = { type => $event_type, %data, ts => time() };
    for my $h (@{$self->{handlers}{$event_type}//[]}) {
        $h->($event);
    }
    for my $h (@{$self->{handlers}{"*"}//[]}) {
        $h->($event);
    }
    return $self;
}
}

printf "\n--- EventBus ---\n";
my $bus = EventBus->new;
my @events;

$bus->on("user.created",  sub { push @events, "Created: $_[0]{name}" });
$bus->on("user.deleted",  sub { push @events, "Deleted: $_[0]{id}" });
$bus->on("*",             sub { push @events, "ALL: $_[0]{type}" });

$bus->emit("user.created", name => "Alice", email => 'a@test.com');
$bus->emit("user.created", name => "Bob");
$bus->emit("user.deleted", id => 1);

printf "%s\n", $_ for @events;
```

---

## Step 384: Server-Sent Events (SSE)

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package SSE::Server;

sub new {
    my ($class) = @_;
    return bless {
        clients  => {},
        channels => {},
        next_id  => 1,
        event_id => 1,
    }, $class;
}

sub connect_client {
    my ($self, %opts) = @_;
    my $id = $self->{next_id}++;
    $self->{clients}{$id} = {
        id       => $id,
        channel  => $opts{channel}   // "default",
        last_id  => $opts{last_event_id} // 0,
        messages => [],
        closed   => 0,
    };
    
    # Send any missed events
    if (my $ch = $self->{channels}{$opts{channel}}) {
        my @missed = grep { $_->{id} > ($opts{last_event_id}//0) } @$ch;
        push @{$self->{clients}{$id}{messages}}, @missed;
    }
    
    return $id;
}

sub disconnect_client {
    my ($self, $client_id) = @_;
    $self->{clients}{$client_id}{closed} = 1;
    delete $self->{clients}{$client_id};
}

sub send_event {
    my ($self, %event) = @_;
    my $channel = $event{channel} // "default";
    my $envelope = {
        id      => $self->{event_id}++,
        event   => $event{event}   // "message",
        data    => $event{data}    // "",
        retry   => $event{retry},
        channel => $channel,
    };
    
    # Store in channel history
    push @{$self->{channels}{$channel}}, $envelope;
    
    # Send to connected clients on this channel
    my $sent = 0;
    for my $id (keys %{$self->{clients}}) {
        my $client = $self->{clients}{$id};
        next if $client->{channel} ne $channel;
        next if $client->{closed};
        push @{$client->{messages}}, $envelope;
        $sent++;
    }
    
    return $sent;
}

sub broadcast {
    my ($self, %event) = @_;
    my $total = 0;
    for my $ch (keys %{$self->{channels}}) {
        $total += $self->send_event(%event, channel => $ch);
    }
    return $total;
}

sub format_event {
    my ($class, $event) = @_;
    my $out = "";
    $out .= "id: $event->{id}\n" if $event->{id};
    $out .= "event: $event->{event}\n" if $event->{event} ne "message";
    $out .= "retry: $event->{retry}\n" if $event->{retry};
    # Format data lines
    for my $line (split /\n/, $event->{data}//"") {
        $out .= "data: $line\n";
    }
    $out .= "\n";
    return $out;
}

sub drain_client {
    my ($self, $client_id) = @_;
    my $client = $self->{clients}{$client_id} or return ();
    my @msgs = @{$client->{messages}};
    $client->{messages} = [];
    return @msgs;
}

sub client_count { scalar keys %{$_[0]->{clients}} }
}

package main;

printf "=== Server-Sent Events ===\n\n";

my $sse = SSE::Server->new;

# Connect clients
my $c1 = $sse->connect_client(channel => "updates");
my $c2 = $sse->connect_client(channel => "updates");
my $c3 = $sse->connect_client(channel => "alerts");

printf "Connected: %d clients\n\n", $sse->client_count;

# Send events
$sse->send_event(channel => "updates", event => "message", data => "System starting...");
$sse->send_event(channel => "updates", event => "progress", data => '{"percent":25,"status":"loading"}');
$sse->send_event(channel => "updates", event => "progress", data => '{"percent":50,"status":"processing"}');
$sse->send_event(channel => "alerts",  event => "warning",  data => "Low memory warning");
$sse->send_event(channel => "updates", event => "progress", data => '{"percent":100,"status":"done"}');
$sse->send_event(channel => "updates", event => "complete", data => "All done!");

# Drain client 1
printf "=== Client 1 (updates channel) ===\n";
for my $event ($sse->drain_client($c1)) {
    printf "%s", SSE::Server->format_event($event);
}

printf "=== Client 3 (alerts channel) ===\n";
for my $event ($sse->drain_client($c3)) {
    printf "%s", SSE::Server->format_event($event);
}

# Late connection (should get missed events)
$sse->disconnect_client($c2);
my $late = $sse->connect_client(channel => "updates", last_event_id => 2);
printf "Late client got %d missed messages\n", scalar($sse->drain_client($late));

# Heartbeat
$sse->send_event(channel => "updates", event => "heartbeat", data => "ping");
printf "\nFormated heartbeat:\n%s", SSE::Server->format_event({
    id => 99, event => "heartbeat", data => "ping" });
```

---

## Step 385: Message Queue

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package MessageQueue;

sub new {
    my ($class, %opts) = @_;
    return bless {
        queues      => {},
        dlq         => {},    # dead letter queue
        max_retries => $opts{max_retries} // 3,
        workers     => {},
        stats       => { enqueued=>0, processed=>0, failed=>0, retried=>0 },
    }, $class;
}

sub enqueue {
    my ($self, $queue, $message, %opts) = @_;
    my $envelope = {
        id       => ++$self->{_next_id},
        payload  => $message,
        retries  => 0,
        priority => $opts{priority} // 5,
        delay    => $opts{delay}    // 0,
        created  => time(),
        visible  => time() + ($opts{delay}//0),
    };
    push @{$self->{queues}{$queue}}, $envelope;
    
    # Sort by priority (higher = first), then by enqueue time
    @{$self->{queues}{$queue}} = sort {
        $b->{priority} <=> $a->{priority} || $a->{created} <=> $b->{created}
    } @{$self->{queues}{$queue}};
    
    $self->{stats}{enqueued}++;
    return $envelope->{id};
}

sub dequeue {
    my ($self, $queue, $visibility_timeout) = @_;
    $visibility_timeout //= 30;
    my $now = time();
    
    for my $msg (@{$self->{queues}{$queue}//[]}) {
        next unless $msg->{visible} <= $now;
        $msg->{visible} = $now + $visibility_timeout;  # Hide from other consumers
        $msg->{processing} = 1;
        return $msg;
    }
    return undef;
}

sub ack {
    my ($self, $queue, $msg_id) = @_;
    @{$self->{queues}{$queue}} = grep { $_->{id} != $msg_id } @{$self->{queues}{$queue}};
    $self->{stats}{processed}++;
    return $self;
}

sub nack {
    my ($self, $queue, $msg_id) = @_;
    for my $msg (@{$self->{queues}{$queue}//[]}) {
        next unless $msg->{id} == $msg_id;
        $msg->{retries}++;
        $msg->{processing} = 0;
        
        if ($msg->{retries} >= $self->{max_retries}) {
            # Move to DLQ
            push @{$self->{dlq}{$queue}}, $msg;
            @{$self->{queues}{$queue}} = grep { $_->{id} != $msg_id } @{$self->{queues}{$queue}};
            $self->{stats}{failed}++;
        } else {
            # Retry with backoff
            $msg->{visible} = time() + (2 ** $msg->{retries});
            $self->{stats}{retried}++;
        }
        last;
    }
    return $self;
}

sub peek { scalar @{$_[0]->{queues}{$_[1]}//[]} }
sub dlq_size { scalar @{$_[0]->{dlq}{$_[1]}//[]} }
sub stats { %{$_[0]->{stats}} }

sub consume {
    my ($self, $queue, $handler, %opts) = @_;
    my $max    = $opts{max} // 100;
    my $count  = 0;
    
    while ($count++ < $max) {
        my $msg = $self->dequeue($queue);
        last unless $msg;
        
        my $ok = eval { $handler->($msg->{payload}); 1 };
        if ($ok) {
            $self->ack($queue, $msg->{id});
        } else {
            $self->nack($queue, $msg->{id});
        }
    }
}
}

package main;

printf "=== Message Queue ===\n\n";

my $mq = MessageQueue->new(max_retries => 3);

# Enqueue messages
printf "Enqueuing:\n";
for my $task (
    ["email_queue",   { to => 'alice@test.com', subject => "Welcome" },             priority=>8],
    ["email_queue",   { to => 'bob@test.com',   subject => "Newsletter"},            priority=>3],
    ["email_queue",   { to => 'carol@test.com', subject => "Reset password"},        priority=>10],
    ["report_queue",  { type => "daily", user_id => 1 }],
    ["report_queue",  { type => "weekly", user_id => 2 }],
    ["index_queue",   { doc_id => 42, action => "update" }],
) {
    my ($queue, $msg, %opts) = @$task;
    my $id = $mq->enqueue($queue, $msg, %opts);
    printf "  Queue=%s id=%d payload=%s\n", $queue, $id, $msg->{subject}//$msg->{type}//$msg->{action};
}

printf "\nQueue sizes: email=%d report=%d index=%d\n",
    $mq->peek("email_queue"), $mq->peek("report_queue"), $mq->peek("index_queue");

# Process email queue (with some failures)
printf "\nProcessing email_queue:\n";
my $send_count = 0;
$mq->consume("email_queue", sub {
    my $msg = shift;
    $send_count++;
    printf "  Sending to %s: '%s'\n", $msg->{to}, $msg->{subject};
    die "Simulated failure!\n" if $msg->{subject} eq "Newsletter";  # Force failure
}, max => 10);

my %stats = $mq->stats;
printf "\nStats: processed=%d failed=%d retried=%d\n",
    $stats{processed}, $stats{failed}, $stats{retried};
printf "DLQ: %d messages\n", $mq->dlq_size("email_queue");
```

---

## Step 386: Actor Model

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Actor;

my %actors;
my $next_id = 1;

sub spawn {
    my ($class, %opts) = @_;
    my $id = "actor_" . $next_id++;
    $actors{$id} = {
        id       => $id,
        name     => $opts{name} // $id,
        mailbox  => [],
        state    => $opts{state} // {},
        behavior => $opts{behavior} // sub {},
        parent   => $opts{parent},
        children => [],
    };
    return $id;
}

sub tell {
    my ($class, $actor_id, $msg) = @_;
    push @{$actors{$actor_id}{mailbox}}, $msg;
    return $class;
}

sub process {
    my ($class, $actor_id) = @_;
    my $actor = $actors{$actor_id} or return;
    
    while (my $msg = shift @{$actor->{mailbox}}) {
        eval {
            $actor->{behavior}->($actor->{state}, $msg, {
                self    => $actor_id,
                tell    => sub { Actor->tell(@_) },
                spawn   => sub { Actor->spawn(@_) },
                state   => $actor->{state},
                name    => $actor->{name},
            });
        };
        if ($@) {
            printf "[Actor %s] Error: %s\n", $actor->{name}, $@;
        }
    }
}

sub process_all {
    my $class = shift;
    my $processed = 0;
    my @ids = keys %actors;
    for my $id (@ids) {
        my $before = scalar @{$actors{$id}{mailbox}};
        $class->process($id);
        $processed += $before;
    }
    return $processed;
}

sub mailbox_size { scalar @{$actors{$_[1]}{mailbox}//[]} }
sub state        { $actors{$_[1]}{state} }
sub actor_count  { scalar keys %actors }
sub kill         { delete $actors{$_[1]} }
}

package main;

printf "=== Actor Model ===\n\n";

my @output;

# Counter actor
my $counter = Actor->spawn(
    name  => "counter",
    state => { count => 0 },
    behavior => sub {
        my ($state, $msg, $ctx) = @_;
        if ($msg->{cmd} eq "increment") {
            $state->{count} += $msg->{by} // 1;
            push @output, "Counter: $state->{count}";
        } elsif ($msg->{cmd} eq "reset") {
            $state->{count} = 0;
            push @output, "Counter reset";
        } elsif ($msg->{cmd} eq "get") {
            push @output, "Counter value: $state->{count}";
        }
    }
);

# Logger actor
my $logger = Actor->spawn(
    name  => "logger",
    state => { messages => [] },
    behavior => sub {
        my ($state, $msg, $ctx) = @_;
        if ($msg->{cmd} eq "log") {
            push @{$state->{messages}}, "[" . time() . "] " . $msg->{text};
            push @output, "LOG: $msg->{text}";
        } elsif ($msg->{cmd} eq "count") {
            push @output, "Log count: " . scalar @{$state->{messages}};
        }
    }
);

# Aggregator actor
my $agg = Actor->spawn(
    name  => "aggregator",
    state => { total => 0, count => 0 },
    behavior => sub {
        my ($state, $msg, $ctx) = @_;
        if ($msg->{cmd} eq "add") {
            $state->{total} += $msg->{value};
            $state->{count}++;
            push @output, "Agg: total=$state->{total} count=$state->{count}";
        } elsif ($msg->{cmd} eq "avg") {
            my $avg = $state->{count} ? $state->{total}/$state->{count} : 0;
            push @output, sprintf "Average: %.2f", $avg;
        }
    }
);

# Send messages
printf "Sending messages:\n";
Actor->tell($counter, { cmd => "increment", by => 5 });
Actor->tell($counter, { cmd => "increment" });
Actor->tell($counter, { cmd => "increment", by => 3 });
Actor->tell($counter, { cmd => "get" });

Actor->tell($logger, { cmd => "log", text => "App started" });
Actor->tell($logger, { cmd => "log", text => "User logged in" });
Actor->tell($logger, { cmd => "count" });

for my $v (10, 20, 30, 40) {
    Actor->tell($agg, { cmd => "add", value => $v });
}
Actor->tell($agg, { cmd => "avg" });

# Process all
my $total = Actor->process_all;
printf "Processed %d messages across %d actors\n\n", $total, Actor->actor_count;

printf "Output:\n";
printf "  %s\n", $_ for @output;

printf "\nFinal states:\n";
printf "  Counter: %d\n", Actor->state($counter)->{count};
printf "  Log msgs: %d\n", scalar @{Actor->state($logger)->{messages}};
printf "  Agg total: %d\n", Actor->state($agg)->{total};
```

---

## Step 387: Reactive Streams

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Observable;

sub new {
    my ($class, $producer) = @_;
    return bless { producer => $producer, operators => [] }, $class;
}

# Constructors
sub from_array {
    my ($class, @items) = @_;
    return $class->new(sub {
        my $obs = shift;
        $obs->on_next($_) for @items;
        $obs->on_complete;
    });
}

sub range {
    my ($class, $start, $count) = @_;
    return $class->new(sub {
        my $obs = shift;
        $obs->on_next($_) for $start..($start+$count-1);
        $obs->on_complete;
    });
}

sub interval {
    my ($class, $n) = @_;
    return $class->new(sub {
        my $obs = shift;
        $obs->on_next($_) for 0..$n-1;
        $obs->on_complete;
    });
}

sub of {
    my ($class, @values) = @_;
    return $class->from_array(@values);
}

# Operators
sub map {
    my ($self, $fn) = @_;
    my $source = $self;
    return ref($self)->new(sub {
        my $obs = shift;
        $source->subscribe(
            on_next     => sub { $obs->on_next($fn->($_[0])) },
            on_error    => sub { $obs->on_error($_[0]) },
            on_complete => sub { $obs->on_complete },
        );
    });
}

sub filter {
    my ($self, $pred) = @_;
    my $source = $self;
    return ref($self)->new(sub {
        my $obs = shift;
        $source->subscribe(
            on_next     => sub { $obs->on_next($_[0]) if $pred->($_[0]) },
            on_error    => sub { $obs->on_error($_[0]) },
            on_complete => sub { $obs->on_complete },
        );
    });
}

sub take {
    my ($self, $n) = @_;
    my $source = $self;
    return ref($self)->new(sub {
        my $obs = shift;
        my $count = 0;
        $source->subscribe(
            on_next     => sub { $obs->on_next($_[0]) if ++$count <= $n },
            on_error    => sub { $obs->on_error($_[0]) },
            on_complete => sub { $obs->on_complete },
        );
    });
}

sub skip {
    my ($self, $n) = @_;
    my $source = $self;
    return ref($self)->new(sub {
        my $obs = shift;
        my $count = 0;
        $source->subscribe(
            on_next     => sub { $obs->on_next($_[0]) if ++$count > $n },
            on_error    => sub { $obs->on_error($_[0]) },
            on_complete => sub { $obs->on_complete },
        );
    });
}

sub reduce {
    my ($self, $fn, $init) = @_;
    my $source = $self;
    return ref($self)->new(sub {
        my $obs = shift;
        my $acc = $init;
        $source->subscribe(
            on_next     => sub { $acc = $fn->($acc, $_[0]) },
            on_error    => sub { $obs->on_error($_[0]) },
            on_complete => sub { $obs->on_next($acc); $obs->on_complete },
        );
    });
}

sub flat_map {
    my ($self, $fn) = @_;
    my $source = $self;
    return ref($self)->new(sub {
        my $obs = shift;
        $source->subscribe(
            on_next  => sub {
                my $inner = $fn->($_[0]);
                $inner->subscribe(
                    on_next     => sub { $obs->on_next($_[0]) },
                    on_error    => sub { $obs->on_error($_[0]) },
                    on_complete => sub {},
                );
            },
            on_error    => sub { $obs->on_error($_[0]) },
            on_complete => sub { $obs->on_complete },
        );
    });
}

sub distinct {
    my $source = $_[0];
    return ref($source)->new(sub {
        my $obs = shift;
        my %seen;
        $source->subscribe(
            on_next     => sub { $obs->on_next($_[0]) unless $seen{$_[0]}++ },
            on_error    => sub { $obs->on_error($_[0]) },
            on_complete => sub { $obs->on_complete },
        );
    });
}

sub do_on_next {
    my ($self, $fn) = @_;
    my $source = $self;
    return ref($self)->new(sub {
        my $obs = shift;
        $source->subscribe(
            on_next     => sub { $fn->($_[0]); $obs->on_next($_[0]) },
            on_error    => sub { $obs->on_error($_[0]) },
            on_complete => sub { $obs->on_complete },
        );
    });
}

sub subscribe {
    my ($self, %handlers) = @_;
    my $observer = bless {
        on_next     => $handlers{on_next}     // sub {},
        on_error    => $handlers{on_error}    // sub { die $_[0] },
        on_complete => $handlers{on_complete} // sub {},
    }, "Observer";
    
    eval { $self->{producer}->($observer) };
    $observer->on_error($@) if $@;
}

sub to_array {
    my $self = shift;
    my @result;
    $self->subscribe(on_next => sub { push @result, $_[0] });
    return @result;
}

sub first {
    my @r = $_[0]->take(1)->to_array;
    return $r[0];
}

sub count {
    my @r = $_[0]->reduce(sub { $_[0]+1 }, 0)->to_array;
    return $r[0];
}
}

{
package Observer;
sub on_next     { $_[0]->{on_next}->($_[1])     if $_[0]->{on_next} }
sub on_error    { $_[0]->{on_error}->($_[1])    if $_[0]->{on_error} }
sub on_complete { $_[0]->{on_complete}->()      if $_[0]->{on_complete} }
}

package main;

printf "=== Reactive Streams (Observable) ===\n\n";

# Basic operations
printf "range(1,10).filter(even).map(x2).take(4):\n";
my @result = Observable->range(1, 10)
    ->filter(sub { $_[0] % 2 == 0 })
    ->map(sub { $_[0] * 2 })
    ->take(4)
    ->to_array;
printf "  %s\n\n", join(", ", @result);

# Reduce
my $sum = Observable->range(1, 10)->reduce(sub { $_[0] + $_[1] }, 0)->first;
printf "sum(1..10) = %d\n\n", $sum;

# distinct
printf "distinct:\n";
my @d = Observable->from_array(1,2,2,3,3,3,4)->distinct->to_array;
printf "  %s\n\n", join(", ", @d);

# flat_map
printf "flat_map (multiply each by range):\n";
my @fm = Observable->from_array(1, 2, 3)
    ->flat_map(sub { Observable->range(1,$_[0]) })
    ->to_array;
printf "  %s\n\n", join(", ", @fm);

# Real-world: data pipeline
printf "Data pipeline:\n";
my @data = (
    { name => "Alice",  score => 85, dept => "eng" },
    { name => "Bob",    score => 72, dept => "mkt" },
    { name => "Carol",  score => 91, dept => "eng" },
    { name => "Dave",   score => 65, dept => "mkt" },
    { name => "Eve",    score => 88, dept => "eng" },
);

my @eng_top = Observable->from_array(@data)
    ->filter(sub { $_[0]->{dept} eq "eng" })
    ->filter(sub { $_[0]->{score} >= 80 })
    ->map(sub { { name => $_[0]->{name}, grade => $_[0]->{score} >= 90 ? "A" : "B" } })
    ->to_array;

printf "  Eng dept, score>=80:\n";
printf "    %s: %s\n", $_->{name}, $_->{grade} for @eng_top;
```

---

## Step 388: Connection Multiplexing

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package Multiplexer;

# Simulate multiplexed connections (HTTP/2 style)
sub new {
    my ($class) = @_;
    return bless {
        streams    => {},
        next_stream=> 1,
        handlers   => {},
        queue      => [],
    }, $class;
}

sub open_stream {
    my ($self, %opts) = @_;
    my $id = $self->{next_stream}++;
    $self->{streams}{$id} = {
        id      => $id,
        state   => "open",
        method  => $opts{method} // "GET",
        path    => $opts{path}   // "/",
        headers => $opts{headers} // {},
        data    => [],
        weight  => $opts{weight} // 16,
        priority=> $opts{priority} // 0,
    };
    return $id;
}

sub send_data {
    my ($self, $stream_id, $data, %opts) = @_;
    push @{$self->{queue}}, {
        stream_id => $stream_id,
        type      => "DATA",
        data      => $data,
        end_stream=> $opts{end_stream} // 0,
    };
}

sub send_headers {
    my ($self, $stream_id, %headers) = @_;
    push @{$self->{queue}}, {
        stream_id => $stream_id,
        type      => "HEADERS",
        headers   => \%headers,
    };
}

sub on_stream {
    my ($self, $handler) = @_;
    $self->{handlers}{stream} = $handler;
}

sub process {
    my $self = shift;
    my @frames = @{$self->{queue}};
    $self->{queue} = [];
    
    # Process by priority
    @frames = sort { ($b->{priority}//0) <=> ($a->{priority}//0) } @frames;
    
    my @results;
    for my $frame (@frames) {
        if ($frame->{type} eq "DATA" || $frame->{type} eq "HEADERS") {
            if (my $h = $self->{handlers}{stream}) {
                my $result = $h->($frame);
                push @results, { stream_id => $frame->{stream_id}, result => $result };
            }
        }
    }
    return @results;
}

sub stats {
    my $self = shift;
    return {
        active_streams => scalar(grep { $_->{state} eq "open" } values %{$self->{streams}}),
        total_streams  => scalar keys %{$self->{streams}},
        queued_frames  => scalar @{$self->{queue}},
    };
}
}

package main;

printf "=== Connection Multiplexing ===\n\n";

my $mux = Multiplexer->new;

# Request handler
my %responses = (
    "GET /api/users"    => { status => 200, body => '[{"id":1},{"id":2}]' },
    "GET /api/posts"    => { status => 200, body => '[{"id":1,"title":"Post"}]' },
    "GET /api/status"   => { status => 200, body => '{"ok":true}' },
    "POST /api/events"  => { status => 201, body => '{"created":true}' },
);

$mux->on_stream(sub {
    my $frame = shift;
    return unless $frame->{type} eq "HEADERS";
    my $key = "$frame->{headers}{method} $frame->{headers}{path}";
    return $responses{$key} // { status => 404, body => '{"error":"not found"}' };
});

# Simulate concurrent requests on multiple streams
printf "Opening streams:\n";
for my $req (
    { method => "GET",  path => "/api/users",  priority => 5 },
    { method => "GET",  path => "/api/posts",  priority => 3 },
    { method => "GET",  path => "/api/status", priority => 10 },
    { method => "POST", path => "/api/events", priority => 7 },
    { method => "GET",  path => "/api/missing",priority => 1 },
) {
    my $id = $mux->open_stream(%$req);
    $mux->send_headers($id, method => $req->{method}, path => $req->{path});
    printf "  Stream %d: %s %s (priority=%d)\n", $id, $req->{method}, $req->{path}, $req->{priority}//0;
}

# Update priority on frames
for my $frame (@{$mux->{queue}}) {
    my $stream = $mux->{streams}{$frame->{stream_id}};
    $frame->{priority} = $stream->{priority} if $stream;
}

printf "\nProcessing (priority order):\n";
my @results = $mux->process;
for my $r (@results) {
    printf "  Stream %d: %d %s\n", $r->{stream_id}, $r->{result}{status},
        substr($r->{result}{body},0,40);
}

my $stats = $mux->stats;
printf "\nStats: active_streams=%d total=%d queued=%d\n",
    $stats->{active_streams}, $stats->{total_streams}, $stats->{queued_frames};
```

---

## Step 389: Async Task Queue with Workers

```perl
#!/usr/bin/perl
use strict;
use warnings;

{
package WorkerPool;

sub new {
    my ($class, %opts) = @_;
    return bless {
        workers   => $opts{workers}  // 4,
        queue     => [],
        results   => {},
        stats     => { submitted=>0, completed=>0, failed=>0, worker_idle=>0 },
        _next_id  => 1,
    }, $class;
}

sub submit {
    my ($self, $task, %opts) = @_;
    my $id = $self->{_next_id}++;
    push @{$self->{queue}}, {
        id       => $id,
        task     => $task,
        priority => $opts{priority} // 0,
        retries  => 0,
        max_retry=> $opts{max_retry} // 0,
        timeout  => $opts{timeout}  // 30,
    };
    
    # Priority sort
    @{$self->{queue}} = sort { $b->{priority} <=> $a->{priority} } @{$self->{queue}};
    $self->{stats}{submitted}++;
    return $id;
}

sub run {
    my $self = shift;
    
    while (my $job = shift @{$self->{queue}}) {
        my $worker = $self->_get_worker;
        $self->_execute($worker, $job);
    }
}

sub _execute {
    my ($self, $worker_id, $job) = @_;
    
    my $result = eval { $job->{task}->() };
    my $err = $@;
    
    if ($err) {
        if ($job->{retries} < $job->{max_retry}) {
            $job->{retries}++;
            push @{$self->{queue}}, $job;
            $self->{stats}{submitted}++;
        } else {
            $self->{results}{$job->{id}} = { status => "failed", error => $err };
            $self->{stats}{failed}++;
        }
    } else {
        $self->{results}{$job->{id}} = { status => "done", result => $result };
        $self->{stats}{completed}++;
    }
}

sub _get_worker { ++$_[0]->{_worker_round} % $_[0]->{workers} + 1 }

sub result {
    my ($self, $id) = @_;
    return $self->{results}{$id};
}

sub wait_all {
    my $self = shift;
    $self->run;
    return %{$self->{results}};
}

sub stats { %{$_[0]->{stats}} }
}

package main;

printf "=== Worker Pool ===\n\n";

my $pool = WorkerPool->new(workers => 4);

# Submit tasks
my @task_ids;
my @tasks = (
    ["Compress images",  sub { sleep(0); "compressed 42 images" }],
    ["Send emails",      sub { sleep(0); "sent 15 emails" }],
    ["Generate report",  sub { sleep(0); "report.pdf (2.3MB)" }],
    ["Index search",     sub { sleep(0); "indexed 1000 docs" }],
    ["Cleanup old data", sub { sleep(0); "deleted 500 rows" }],
    ["Failing task",     sub { die "Network error\n" }],
    ["Retry task",       do { my $a=0; sub { die "Temp fail\n" if ++$a<2; "success on retry $a" } }],
);

for my $t (@tasks) {
    my ($name, $task) = @$t;
    my $id = $pool->submit($task, max_retry => 2);
    push @task_ids, [$id, $name];
    printf "  Submitted '%s' (id=%d)\n", $name, $id;
}

printf "\nRunning workers:\n";
$pool->run;

printf "\nResults:\n";
for my $t (@task_ids) {
    my ($id, $name) = @$t;
    my $r = $pool->result($id);
    if ($r) {
        printf "  [%s] %s: %s\n", $r->{status} eq "done" ? "OK" : "FAIL",
            $name, $r->{result} // $r->{error};
    } else {
        printf "  [PENDING] %s\n", $name;
    }
}

my %stats = $pool->stats;
printf "\nStats: submitted=%d completed=%d failed=%d\n",
    $stats{submitted}, $stats{completed}, $stats{failed};
```

---

## Step 390: Capstone — Real-time Chat Application

```perl
#!/usr/bin/perl
# realtime_chat.pl — Complete real-time chat with SSE, pub/sub, workers
use strict;
use warnings;
use JSON::PP;

my $JSON = JSON::PP->new->utf8->canonical;

# Components
my $pubsub = do {
    my %channels;
    my $msg_id = 0;
    {
        publish => sub {
            my ($ch, $data) = @_;
            my $env = { id => ++$msg_id, channel => $ch, %$data, ts => time() };
            push @{$channels{$ch}}, $env;
            return $env;
        },
        subscribe => sub {
            my ($ch, $since) = @_;
            return [grep { !$since || $_->{id} > $since } @{$channels{$ch}//[]}];
        },
        channels => sub { sort keys %channels },
        history  => sub { $channels{$_[0]} // [] },
    };
};

my %users;    # id => user info
my %sessions; # session_token => user_id
my $user_id  = 0;
my $room_id  = 0;
my %rooms;

# Auth
sub register_user {
    my (%info) = @_;
    my $id  = ++$user_id;
    my $tok = sprintf "tok_%08x", rand(0xFFFFFFFF);
    $users{$id} = { id => $id, username => $info{username}, status => "online", rooms => [] };
    $sessions{$tok} = $id;
    return ($id, $tok);
}

sub get_user { $users{$_[0]} }
sub auth     { $sessions{$_[0]} ? $users{$sessions{$_[0]}} : undef }

# Rooms
sub create_room {
    my (%opts) = @_;
    my $id = ++$room_id;
    $rooms{$id} = { id => $id, name => $opts{name}, type => $opts{type}//"public",
                    members => [], created_by => $opts{created_by} };
    return $id;
}

sub join_room {
    my ($user_id, $room_id) = @_;
    my $room = $rooms{$room_id} or return 0;
    my $user = $users{$user_id} or return 0;
    push @{$room->{members}}, $user_id unless grep { $_ == $user_id } @{$room->{members}};
    push @{$user->{rooms}}, $room_id   unless grep { $_ == $room_id } @{$user->{rooms}};
    $pubsub->{publish}->("room:$room_id", { event => "user_joined", user => $user->{username} });
    return 1;
}

sub send_message {
    my ($user_id, $room_id, $text) = @_;
    my $user = $users{$user_id} or die "Unknown user\n";
    my $room = $rooms{$room_id} or die "Unknown room\n";
    die "Not a member\n" unless grep { $_ == $user_id } @{$room->{members}};
    
    my $msg = $pubsub->{publish}->("room:$room_id", {
        event    => "message",
        user_id  => $user_id,
        username => $user->{username},
        text     => $text,
    });
    return $msg;
}

sub get_room_history {
    my ($room_id, $since) = @_;
    return $pubsub->{subscribe}->("room:$room_id", $since);
}

sub online_users {
    return [grep { $_->{status} eq "online" } values %users];
}

sub room_members {
    my $rid = shift;
    return [map { $users{$_} } @{$rooms{$rid}{members}}];
}

# Run simulation
printf "=== Real-time Chat Application ===\n\n";

# Register users
my ($alice_id, $alice_tok) = register_user(username => "alice");
my ($bob_id,   $bob_tok)   = register_user(username => "bob");
my ($carol_id, $carol_tok) = register_user(username => "carol");
my ($dave_id,  $dave_tok)  = register_user(username => "dave");

printf "Registered users: %s\n", join(", ", map { $_->{username} } values %users);

# Create rooms
my $general = create_room(name => "general",  type => "public",  created_by => $alice_id);
my $perl_ch  = create_room(name => "perl",     type => "public",  created_by => $bob_id);
my $dm       = create_room(name => "alice-bob",type => "private", created_by => $alice_id);

printf "\nRooms: %s\n", join(", ", map { "#$_->{name}" } values %rooms);

# Join rooms
join_room($alice_id, $general);
join_room($bob_id,   $general);
join_room($carol_id, $general);
join_room($dave_id,  $general);
join_room($alice_id, $perl_ch);
join_room($bob_id,   $perl_ch);
join_room($carol_id, $perl_ch);
join_room($alice_id, $dm);
join_room($bob_id,   $dm);

printf "\n#general members: %d\n", scalar @{room_members($general)};
printf "#perl members: %d\n", scalar @{room_members($perl_ch)};

# Send messages
printf "\n--- Chat activity ---\n";
my @messages = (
    [$alice_id, $general, "Hey everyone! Welcome to #general"],
    [$bob_id,   $general, "Hi Alice! How's it going?"],
    [$carol_id, $general, "Hello all!"],
    [$alice_id, $perl_ch, "Anyone using Mojolicious?"],
    [$bob_id,   $perl_ch, "Yes! It's great for real-time apps"],
    [$carol_id, $perl_ch, "I prefer Dancer2 actually"],
    [$alice_id, $dm,      "Hey Bob, can we talk?"],
    [$bob_id,   $dm,      "Sure Alice! What's up?"],
    [$dave_id,  $general, "Just joined, this is cool!"],
    [$alice_id, $general, "Great to have you Dave!"],
);

for my $m (@messages) {
    my ($uid, $rid, $text) = @$m;
    my $msg = eval { send_message($uid, $rid, $text) };
    if ($@) { printf "  ERROR: %s\n", $@ }
    else {
        printf "  [#%s] <%s> %s\n",
            $rooms{$rid}{name}, $users{$uid}{username}, $text;
    }
}

# Get history
printf "\n--- Room history ---\n";
printf "#general (%d messages):\n", scalar @{get_room_history($general)};
for my $msg (grep { $_->{event} eq "message" } @{get_room_history($general)}) {
    printf "  <%s> %s\n", $msg->{username}, $msg->{text};
}

printf "\n#perl (%d messages):\n", scalar @{get_room_history($perl_ch)};
for my $msg (grep { $_->{event} eq "message" } @{get_room_history($perl_ch)}) {
    printf "  <%s> %s\n", $msg->{username}, $msg->{text};
}

# SSE events (what a client would receive)
printf "\n--- SSE stream for Alice (polling #general since msg 5) ---\n";
my $since = 5;
for my $event (@{get_room_history($general, $since)}) {
    next unless $event->{event} eq "message";
    printf "data: %s\n\n", $JSON->encode($event);
}

# Stats
printf "--- Stats ---\n";
printf "Online users: %d\n", scalar @{online_users()};
printf "Total rooms: %d\n", scalar keys %rooms;
printf "Total events (all channels): %d\n",
    do { my $t=0; $t += scalar @{$pubsub->{history}->($_)} for $pubsub->{channels}->(); $t };
```

---

## สรุป Part 39 — WebSockets & Real-time Perl

### สิ่งที่เรียนรู้:
- **AnyEvent** — Event loop simulation with timers, intervals, CondVar
- **Async HTTP Server** — Non-blocking connection handling with middleware
- **Pub/Sub** — Channel-based messaging, EventBus type routing
- **Server-Sent Events** — SSE format, history/replay, late connections
- **Message Queue** — Priority queues, DLQ, backoff retry
- **Actor Model** — Mailbox-based message passing, concurrent actors
- **Reactive Streams** — Observable with map/filter/take/reduce/flat_map
- **Multiplexing** — HTTP/2-style stream priority handling
- **Worker Pool** — Concurrent task execution with retry
- **Capstone** — Real-time chat with SSE, pub/sub, rooms, history

**ถัดไป: [Part 40 — Authentication Systems & OAuth2](part_40.md)**
