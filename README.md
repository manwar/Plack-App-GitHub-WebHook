# NAME

Plack::App::GitHub::WebHook - GitHub WebHook receiver as Plack application

# STATUS

[![Build Status](https://travis-ci.org/nichtich/Plack-App-GitHub-WebHook.png)](https://travis-ci.org/nichtich/Plack-App-GitHub-WebHook)
[![Coverage Status](https://coveralls.io/repos/nichtich/Plack-App-GitHub-WebHook/badge.png?branch=master)](https://coveralls.io/r/nichtich/Plack-App-GitHub-WebHook?branch=master)
[![Kwalitee Score](http://cpants.cpanauthors.org/dist/Plack-App-GitHub-WebHook.png)](http://cpants.cpanauthors.org/dist/Plack-App-GitHub-WebHook)

# SYNOPSIS

    use Plack::App::GitHub::WebHook;

    # Basic Usage
    Plack::App::GitHub::WebHook->new(
        hook => sub {
            my $payload = shift;
            ...
        },
        events => ['pull'],  # optional
        secret => $secret,   # optional
        access => 'github',  # default
    )->to_app;

    # Multiple hooks
    use IPC::Run3;
    Plack::App::GitHub::WebHook->new(
        hook => [
            sub { $_[0]->{repository}{name} eq 'foo' },
            sub {
                my ($payload, $event, $delivery, $logger) = @_;
                run3 \@cmd, undef, $logger->{info}, $logger->{error}; 
            },
            sub { ...  }, # some more action
        ]
    )->to_app;

# DESCRIPTION

This [PSGI](https://metacpan.org/pod/PSGI) application receives HTTP POST requests with body parameter
`payload` set to a JSON object. The default use case is to receive 
[GitHub WebHooks](http://developer.github.com/webhooks/), for instance
[PushEvents](http://developer.github.com/v3/activity/events/types/#pushevent).

The response of a HTTP request to this application is one of:

- HTTP 403 Forbidden

    If access was not granted (for instance because it did not origin from GitHub).

- HTTP 405 Method Not Allowed

    If the request was no HTTP POST.

- HTTP 400 Bad Request

    If the payload was no well-formed JSON or the `X-GitHub-Event` header did not
    match configured events.

- HTTP 200 OK

    Otherwise, if the hook was called and returned a true value.

- HTTP 202 Accepted

    Otherwise, if the hook was called and returned a false value.

- HTTP 500 Internal Server Error

    If a hook died with an exception, the error is returned as content body. Use
    configuration parameter `safe` to disable HTTP 500 errors. 

This module requires at least Perl 5.10.

# CONFIGURATION

- hook

    A hook can be any of a code reference, an object instance with method `code`,
    a class name, or a class name mapped to parameters. You can also pass a list of
    hooks as array reference. Class names are prepended by [GitHub::WebHook](https://metacpan.org/pod/GitHub::WebHook)
    unless prepended by `+`.

        hook => sub {
            my ($payload, $event, $delivery, $logger) = @_;
            ...
        }

        hook => 'Foo'
        hook => '+GitHub::WebHook::Foo'
        hook => GitHub::WebHook::Foo->new

        hook => { Bar => [ doz => 'baz' ] }
        hook => GitHub::WebHook::Bar->new( doz => 'baz' )
        

    Each hook gets passed the encoded payload, the type of webhook
    [event](https://developer.github.com/webhooks/#events), a unique delivery ID,
    and a [logger object](#logging).  If the hook returns a true value, the next
    the hook is called or HTTP status code 200 is returned.  If a hook returns a
    false value (or if no hook was given), HTTP status code 202 is returned
    immediately.  Information can be passed from one hook to the next by modifying
    the payload. 

- events

    A list of [event types](http://developer.github.com/v3/activity/events/types/)
    expected to be send with the `X-GitHub-Event` header (e.g. `['pull']`).

- logger

    Object or function reference to hande [logging events](#logging).  An object
    must implement method `log` that is called with named arguments:

        $logger->log( level => $level, message => $message );

    For instance [Log::Dispatch](https://metacpan.org/pod/Log::Dispatch) can be used as logger this way.
    A function reference is called with hash reference arguments:

        $logger->({ level => $level, message => $message });

    By default [PSGI::Extensions](https://metacpan.org/pod/psgix.logger) is used as logger (if set).

- secret

    Secret token set at GitHub Webhook setting to validate payload.  See
    [https://developer.github.com/webhooks/securing/](https://developer.github.com/webhooks/securing/) for details. Requires
    [Plack::Middleware::HubSignature](https://metacpan.org/pod/Plack::Middleware::HubSignature).

- access

    Access restrictions, as passed to [Plack::Middleware::Access](https://metacpan.org/pod/Plack::Middleware::Access). A recent list
    of official GitHub WebHook IPs is vailable at [https://api.github.com/meta](https://api.github.com/meta).
    The default value

        access => 'github'

    is a shortcut for these official IP ranges

        access => [
            allow => "204.232.175.64/27",
            allow => "192.30.252.0/22",
            deny  => 'all'
        ]

    and

        access => [
            allow => 'github',
            ...
        ]

    is a shortcut for

        access => [
            allow => "204.232.175.64/27",
            allow => "192.30.252.0/22",
            ...
        ]

    To disable access control via IP ranges use any of

        access => 'all'
        access => []

- safe

    Wrap all hooks in `eval { ... }` blocks to catch exceptions.  Error
    messages are send to the PSGI error stream `psgi.errors`.  A dying hook in
    safe mode is equivalent to a hook that returns a false value, so it will result
    in a HTTP 202 response.

    If you want errors to result in a HTTP 500 response, don't use this option but
    wrap the application in an eval block such as this:

        sub {
            eval { $app->(@_) } || do {
                my $msg = $@ || 'Server Error';
                [ 500, [ 'Content-Length' => length $msg ], [ $msg ] ];
            };
        };

# LOGGING

Each hook is passed a logger object to facilitate logging to
[PSGI::Extensions](https://metacpan.org/pod/psgix.logger). The logger provides logging methods for each
log level and a general log method:

    sub sample_hook {
        my ($payload, $event, $delivery, $log) = @_;

        $log->debug('message');  $log->{debug}->('message');
        $log->info('message');   $log->{info}->('message');
        $log->warn('message');   $log->{warn}->('message');
        $log->error('message');  $log->{error}->('message');
        $log->fatal('message');  $log->{fatal}->('message');

        $log->log( warn => 'message' );

        run3 \@system_command, undef,
            $log->{info},   # STDOUT to log level info
            $log->{error};  # STDERR to log level error
    }

Trailing newlines on log messages are trimmed.

# EXAMPLES

## Synchronize with a GitHub repository

The following application automatically pulls the master branch of a GitHub
repository into a local working directory.

    use Plack::App::GitHub::WebHook;
    use IPC::Run3;

    my $branch = "master";
    my $work_tree = "/some/path";

    Plack::App::GitHub::WebHook->new(
        events => ['push','ping'],
        hook => [
            sub { 
                my ($payload, $event, $delivery, $log) = @_;
                $log->info("$event $delivery");
                $event eq 'ping' or $payload->{ref} eq "refs/heads/$branch";
            },
            sub {
                my ($payload, $event, $delivery, $log) = @_;
                my $origin = $payload->{repository}->{clone_url} 
                           or die "missing clone_url\n";
                my $cmd;
                if ( -d "$work_tree/.git") {
                    chdir $work_tree;
                    $cmd = ['git','pull',$origin,$branch];
                } else {
                    $cmd = ['git','clone',$origin,'-b',$branch,$work_tree];
                }
                $log->info(join ' ', '$', @$cmd);
                run3 $cmd, undef, $log->{debug}, $log->{warn};
                1;
            },
            # sub { ...optional action after each pull... } 
        ],
    )->to_app;

See [GitHub::WebHook::Clone](https://metacpan.org/pod/GitHub::WebHook::Clone) for before copy and pasting this code.

# DEPLOYMENT

Many deployment methods exist. An easy option might be to use Apache webserver
with mod\_cgi and [Plack::Handler::CGI](https://metacpan.org/pod/Plack::Handler::CGI). First install Apache, Plack and
Plack::App::GitHub::WebHook:

    sudo apt-get install apache2
    sudo apt-get install cpanminus libplack-perl
    sudo cpanm Plack::App::GitHub::WebHook

Then add this section to `/etc/apache2/sites-enabled/default` (or another host
configuration) and restart Apache.

    <Directory /var/www/webhooks>
       Options +ExecCGI -Indexes +SymLinksIfOwnerMatch
       AddHandler cgi-script .cgi
    </Directory>

You can now put webhook applications in directory `/var/www/webhooks` as long
as they are executable, have file extension `.cgi` and shebang line
`#!/usr/bin/env plackup`. You might further want to run webhooks scripts as
another user instead of `www-data` by using Apache module SuExec.

# SEE ALSO

- GitHub WebHooks are documented at [http://developer.github.com/webhooks/](http://developer.github.com/webhooks/).
- See [GitHub::WebHook](https://metacpan.org/pod/GitHub::WebHook) for a collection of handlers for typical tasks.
- [WWW::GitHub::PostReceiveHook](https://metacpan.org/pod/WWW::GitHub::PostReceiveHook) uses [Web::Simple](https://metacpan.org/pod/Web::Simple) to receive GitHub web
hooks. A listener as exemplified by the module can also be created like this:

        use Plack::App::GitHub::WebHook;
        use Plack::Builder;
        build {
            mount '/myProject' => 
                Plack::App::GitHub::WebHook->new(
                    hook => sub { my $payload = shift; }
                );
            mount '/myOtherProject' => 
                Plack::App::GitHub::WebHook->new(
                    hook => sub { run3 \@cmd ... }
                );
        };

- [Net::GitHub](https://metacpan.org/pod/Net::GitHub) and [Pithub](https://metacpan.org/pod/Pithub) provide access to GitHub APIs.
- [Github::Hooks::Receiver](https://metacpan.org/pod/Github::Hooks::Receiver) and [App::GitHubWebhooks2Ikachan](https://metacpan.org/pod/App::GitHubWebhooks2Ikachan) are alternative
application that receive GitHub WebHooks.

# COPYRIGHT AND LICENSE

Copyright Jakob Voss, 2014-

This library is free software; you can redistribute it and/or modify it under
the same terms as Perl itself.
