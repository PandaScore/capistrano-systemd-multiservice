# capistrano-systemd-multiservice

[![Gem Version](https://badge.fury.io/rb/capistrano-systemd-multiservice.png)](https://rubygems.org/gems/capistrano-systemd-multiservice) [![Build Status](https://secure.travis-ci.org/groovenauts/capistrano-systemd-multiservice.png)](https://travis-ci.org/groovenauts/capistrano-systemd-multiservice)

This gem adds capistrano tasks to control multiple services with systemd.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'capistrano-systemd-multiservice', require: false
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install capistrano-systemd-multiservice

## Usage

Add these lines to your Capfile:

```ruby
require "capistrano/systemd/multiservice"
install_plugin Capistrano::Systemd::MultiService.new("example1")
install_plugin Capistrano::Systemd::MultiService.new("example2")
```

And put `config/systemd/example1.service.erb` (and `config/systemd/example2.service.erb`, ...) like this
(see [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/systemd.service.html) for details) :

```
[Unit]
Description = <%= fetch(:application) %> application server example1

[Service]
Environment = RAILS_ENV=<%= fetch(:rails_env) %>
Environment = PWD=<%= current_path %>
WorkingDirectory = <%= current_path %>
ExecStart = bundle exec some-application-server start
User = exampleuser
Group = examplegroup

[Install]
WantedBy = multi-user.target
```

And add these lines to config/deploy.rb if you want to reload/restart services on deploy:

```ruby
after 'deploy:publishing', 'systemd:example1:restart'
after 'deploy:publishing', 'systemd:example2:reload-or-restart'
```

And then deploy.

```shell
# Upload and install systemd service unit files before deploy
cap STAGE systemd:example1:setup systemd:example2:setup

# Deploy as usual
cap STAGE deploy
```

## Capistrano Tasks

With `install_plugin Capistrano::Systemd::MultiService.new("example1")`,
following tasks are defined.

- `systemd:example1:setup`
- `systemd:example1:remove`
- `systemd:example1:validate`
- `systemd:example1:daemon-reload`
- `systemd:example1:start`
- `systemd:example1:stop`
- `systemd:example1:reload`
- `systemd:example1:restart`
- `systemd:example1:reload-or-restart`
- `systemd:example1:enable`
- `systemd:example1:disable`

See lib/capistrano/tasks/systemd/multiservice.rake, lib/capistrano/systemd/multiservice.rb for details.

## Configuration Variables

With `install_plugin Capistrano::Systemd::MultiService.new("example1")`,
following Configuration variables are defined.

- `:systemd_example1_role`
- `:systemd_example1_units_src`
- `:systemd_example1_units_dir`
- `:systemd_example1_units_dest`
- `:systemd_example1_instances`
- `:systemd_example1_service`
- `:systemd_example1_instance_services`

See lib/capistrano/systemd/multiservice.rb for details.

## Examples

### Rails application with unicorn and delayed\_job

#### `Capfile`

```ruby
## ...snip...

require 'capistrano/systemd/multiservice'
install_plugin Capistrano::Systemd::MultiService.new('unicorn')
install_plugin Capistrano::Systemd::MultiService.new('delayed_job')

## ...snip...
```

#### `config/deploy.rb`

```ruby
## ...snip...

set :systemd_delayed_job_instances, ->{ 3.times.to_a }

after 'deploy:restart', 'systemd:unicorn:reload-or-restart'
after 'deploy:restart', 'systemd:delayed_job:restart'

after 'deploy:publishing', 'deploy:restart'

## ...snip...
```

#### `config/systemd/unicorn.service.erb`

```
[Unit]
Description = <%= fetch(:application) %> unicorn rack server

[Service]
Environment = PATH=<%= fetch(:rbenv_path) %>/shims:/usr/local/bin:/usr/bin:/bin
Environment = RBENV_VERSION=<%= fetch(:rbenv_ruby) %>
Environment = RBENV_ROOT=<%= fetch(:rbenv_path) %>
Environment = RAILS_ENV=<%= fetch(:rails_env) %>
Environment = PWD=<%= current_path %>

WorkingDirectory = <%= current_path %>

ExecStart = <%= fetch(:rbenv_path) %>/bin/rbenv exec bundle exec unicorn -c <%= current_path %>/config/unicorn.rb
ExecReload = /bin/kill -USR2 $MAINPID

PIDFile = <%= shared_path %>/tmp/pids/unicorn.pid
KillSignal = SIGQUIT
KillMode = process
TimeoutStopSec = 62
Restart = always

User = app-user
Group = app-group

[Install]
WantedBy = multi-user.target
```

#### `config/systemd/delayed_job.service.erb`

```
[Unit]
Description = <%= fetch(:application) %> delayed_job
Requires = <%= fetch(:"#{prefix}_instance_services").join(" ") %>

[Service]
Type = oneshot
RemainAfterExit = yes
ExecStart  = /bin/true
ExecReload = /bin/true

[Install]
WantedBy = multi-user.target
```

#### `config/systemd/delayed_job@.service.erb`

```
[Unit]
Description = <%= fetch(:application) %> delayed_job (instance %i)
PartOf = <%= fetch(:"#{prefix}_service") %>
ReloadPropagatedFrom = <%= fetch(:"#{prefix}_service") %>

[Service]
Type = forking

Environment = PATH=<%= fetch(:rbenv_path) %>/shims:/usr/local/bin:/usr/bin:/bin
Environment = RBENV_VERSION=<%= fetch(:rbenv_ruby) %>
Environment = RBENV_ROOT=<%= fetch(:rbenv_path) %>
Environment = RAILS_ENV=<%= fetch(:rails_env) %>
Environment = PWD=<%= current_path %>

WorkingDirectory = <%= current_path %>

ExecStart  = <%= fetch(:rbenv_path) %>/bin/rbenv exec bundle exec bin/delayed_job -p <%= fetch(:application) %> -i %i start
ExecStop   = <%= fetch(:rbenv_path) %>/bin/rbenv exec bundle exec bin/delayed_job -p <%= fetch(:application) %> -i %i stop
ExecReload = /bin/kill -HUP $MAINPID

PIDFile = <%= shared_path %>/tmp/pids/delayed_job.%i.pid
TimeoutStopSec = 22
Restart = always

User = app-user
Group = app-group

[Install]
WantedBy = multi-user.target
```

#### `config/unicorn.rb`

```ruby
shared_path = "/path/to/shared"

worker_processes   5
listen             "#{shared_path}/tmp/sockets/unicorn.sock"
pid                "#{shared_path}/tmp/pids/unicorn.pid"
stderr_path        "#{shared_path}/log/unicorn_stderr.log"
stdout_path        "#{shared_path}/log/unicorn_stdout.log"
preload_app        true

before_exec do |server|
  ENV["BUNDLE_GEMFILE"] = "/path/to/current/Gemfile"
end

before_fork do |server, worker|
  if defined? ActiveRecord::Base
    ActiveRecord::Base.connection.disconnect!
  end

  old_pid = "#{server.config[:pid]}.oldbin"
  if old_pid != server.pid
    begin
      sig = (worker.nr + 1) >= server.worker_processes ? :QUIT : :TTOU
      Process.kill(sig, File.read(old_pid).to_i)
    rescue Errno::ENOENT, Errno::ESRCH
    end
  end

  sleep 1
end

after_fork do |server, worker|
  if defined?(ActiveRecord::Base)
    ActiveRecord::Base.establish_connection
  end
end
```

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/groovenauts/capistrano-systemd-multiservice.

