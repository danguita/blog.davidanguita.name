---
layout: post
title: "Setting up a cheap Redmine server using Unicorn and Apache"
date: 2013-03-03 17:04
comments: true
categories:
  - Redmine
  - Rails
  - Unicorn
  - Apache
  - DevOps
---

[Redmine](http://redmine.org) is an excellent web-based project management tool.
It's open source, free and is built using Ruby on Rails.

We use Redmine as our main tool for planning, issue tracking and
documenting processes. It's an everyday tool so we need to have an always ready
Redmine instance within our private domain.

<!-- more -->

### Get a small VPS server

We use an Amazon EC2 t1.micro instance running Ubuntu Precise 64-bit as
development server.

This *machine* is mainly managing a Git server with dozens of repos (and
backing them up as a S3 filesystem) and a Redmine instance.

As described on [Amazon EC2 docs](http://aws.amazon.com/ec2/instance-types/),
this instance type has pretty basic resources, but are enough for running
some lightweight processes with low I/O load.

#### Add your service user

We're so bad choosing funny names so our service username is `service`.

### Redmine

Log in as your service user and clone the latest Redmine tree:

    $ mkdir -p ~/apps/redmine
    $ git clone git://github.com/redmine/redmine.git ~/apps/redmine

Install and set up your database engine server (MySQL, PostgreSQL…) and edit the
`~/apps/redmine/config/database.yml` file to match these settings. Then you'll
be ready to set up the production environment:

    $ cd ~/apps/redmine
    $ bundle install --without development test
    $ RAILS_ENV=production bundle exec rake db:migrate

### Unicorn

#### "unicorn" gem

Add the `unicorn` gem as an application dependency by adding a
`Gemfile.local` file (unobtrusive way to add dependencies)
with this content:

``` ruby ~/apps/redmine/Gemfile.local
source 'https://rubygems.org'

gem 'unicorn'
```

After that you should run `bundle install` again:

    $ cd ~/apps/redmine
    $ bundle install --without development test

#### Filesystem

We do store all shared resources such as config, logs or process pids in a
folder named `shared`:

    $ mkdir -p ~/shared/{config/redmine,log/redmine,pid/redmine}

The previous command will create the following structure under
`/home/service`:

    ├── shared
    │   ├── config
    │   │   └── redmine
    │   ├── log
    │   │   └── redmine
    │   └── pid
    │       └── redmine

#### Config

    $ curl -o ~/shared/config/redmine/unicorn.rb https://raw.github.com/defunkt/unicorn/master/examples/unicorn.conf.rb

After getting the Unicorn config example, you will need to tweak a few
lines:

* *worker_processes*: Number of Unicorn workers you need.
* *working directory*: `working_directory "/home/service/apps/redmine"`
* *pid*: `pid "/home/service/shared/pid/redmine/unicorn.pid"`
* *log*:
    `stderr_path "/home/service/shared/log/redmine/unicorn.stderr.log"`
    `stdout_path "/home/service/shared/log/redmine/unicorn.stdout.log"`

#### "Automate everything"

We put all service-related scripts in the `~/script` directory. 
These commands might help you on launching and stopping Unicorn
processes:

``` bash ~/script/launch-redmine-unicorn
cd /home/service/apps/redmine && unicorn_rails -c /home/service/shared/config/redmine/unicorn.rb -p 5000 -E production -D
```

Command explanation:

* `-c` sets the config file (*/home/service/shared/config/redmine/unicorn.rb*)
* `-p` sets the port of the master Unicorn process (*5000*)
* `-E` sets the environment (*production*)
* `-D` daemonizes the command

``` bash ~/script/stop-redmine-unicorn
kill -QUIT $(cat /home/service/shared/pid/redmine/unicorn.pid)
```

Because [Unicorn is Unix](http://tomayko.com/writings/unicorn-is-unix) you can
send a `QUIT` signal to the master process to stop all its workers.

### Apache

#### Modules

You will need to enable these Apache2 modules:

    sudo a2enmod rewrite
    sudo a2enmod proxy
    sudo a2enmod proxy_balancer
    sudo a2enmod proxy_http

#### Proxy Unicorn processes 

We have to set up a new VirtualHost as a proxy balancer of the master
Unicorn process running at specified local port (*127.0.0.1:5000*):

{% gist 5077319 %}

Note that Apache logging was disabled for reducing I/O load. Feel free
to set it up if your resources are not being affected by I/O operations.

### Applying all

After enabling the recently created VirtualHost and restarting the
`apache2` service, a server proxy will be listening for processes on
the specified host and port.

Now run the Unicorn process as shown above and Redmine should be served
by the new VirtualHost.

Now it's time to do some tests in order to adjust the number of Unicorn workers
you need to have running on your server.
