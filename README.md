# 在 VPS 上安装 Discourse 论坛程序的中文指南

## 相关说明

本指南基于以下资源创建：

- Christopher Baus <christopher@baus.net> 的 Discourse 安装指南（英文），遵循 GPL 2.0 协议。
- Lee Dohm <lee@liftedstudios.com> 的 Discourse 安装指南（英文）。

本指南遵循创作共用协议，欢迎转载，请注明出处及原始链接。

关于 Discourse 的详细信息，请访问其[官方网站](http://discourse.org)。

## 安装环境

- 一台安装了 Ubuntu 12.10 x86 操作系统的 VPS
- 内存 1G （Discourse 官方推荐的最小值）




## Log in to Your Server

I will use discoursetest.org when a domain name is required in the installation. You should replace discoursetest.org with your own domain name. If you are using OS X or Linux, start a terminal and ssh to your new server. Windows users should consider installing [Putty](http://putty.org/) to access their new server.

## Create a User Account

While the Amazon EC2 Ubuntu images all have a non-root `ubuntu` admin user, I chose to create a more personal admin user account. For the purposes of this document, I'm going to call the new user `admin`.

Adding the user to the sudo group will allow the user to perform tasks as root using the [sudo](https://help.ubuntu.com/community/RootSudo) command. 

```bash
$ sudo adduser admin
$ sudo adduser admin sudo
```

If you need help configuring SSH access to the new `admin` account and you're using Unix or OS X, you can use [these instructions](SSH.md).

## Log in Using the Admin Account

```bash
$ logout
# Now back at the local terminal prompt
$ ssh admin@discoursetest.org
```

## Use `apt-get` to Install Core System Dependencies

The apt-get command is used to add packages to Ubuntu (and all Debian based Linux distributions). The Amazon EC2 Ubuntu images come with a limited configuration, so you will have to install many of the software dependencies yourself.

To install system packages, you must have root privileges. Since the admin account is part of the sudo group, the admin account can run commands with root privileges by using the sudo command. Just prepend `sudo` to any commands you want to run as root. This includes apt-get commands to install packages.

```bash
# Install required packages
$ sudo apt-get install git-core build-essential postgresql postgresql-contrib libxml2-dev libxslt-dev libpq-dev redis-server nginx postfix
```

During the installation, you will be prompted for Postfix configuration information. [Postfix](https://help.ubuntu.com/community/Postfix) is used to send mail from Discourse. Just keep the default "Internet Site."

At the next prompt just enter your domain name. In my test case this is discoursetest.org.

## Edit Configuration Files

At various points in the installation procedure, you will need to edit configuration files with a text editor. `vi` is installed by default and is the de facto standard editor used by admins (although it appears that `nano` is configured as the default editor for many users in the Ubuntu image), so I use `vi` for any editing commands, but you may want to consider installing the editor of your choice.

## Set the Host Name

EC2's provisioning procedure doesn't assume your instance will require a hostname when it is created. I'd recommend editing /etc/hosts to correctly contain your hostname.

```bash
$ sudo vi /etc/hosts
```
The first line of my /etc/hosts file looks like:

```bash
127.0.0.1  forum.discoursetest.org forum localhost
```

You should replace discoursetest.org with your own domain name. 

## Configure Postgres User Account

Discourse uses the Postgres database to store forum data. The configuration procedure is similar to MySQL, but I am a Postgres newbie, so if you have improvements to this aspect of the installation procedure, please let me know.

Note: this is the easiest way to setup the Postgres server, but it also creates a highly privileged Postgres user account. Future revisions of this document may offer alternatives for creating the Postgres DBs, which would allow Discourse to login to Postgres as a user with lower privileges.

```bash
$ sudo -u postgres createuser admin -s -P
```

It will ask for a password for the account, so pick one and remember it.  You will need the password for this database account when editing the Discourse database configuration file.

## Install and Configure RVM and Ruby

I chose to use RVM to manage installations of Ruby and Gems.  These instructions will employ a multi-user RVM installation so that all user accounts will have access to RVM, Ruby and the gemsets we will create.

First, install RVM and add your admin account to the `rvm` group:

```bash
$ \curl -L https://get.rvm.io | sudo bash -s stable
$ sudo adduser admin rvm
```

After adding yourself to the `rvm` group, you will need to log out and log back in to register the change and activate RVM for your session.

Then install Ruby v1.9.3 and create a gemset for Discourse:

```bash
$ rvm install 1.9.3
$ rvm use --default 1.9.3
$ rvm gemset create discourse
```

*Note:* Some people are using Ruby v2.0 for their installations to good effect, but I have not tested version 2 with these instructions.

## Pull and Configure the Discourse Application

Now we are ready install the actual Discourse application. This will pull a copy of the Discourse app from my own branch. The advantage of using this branch is that it has been tested with these instructions, but it may fall behind the master which is rapidly changing. 

```bash
# I prefer to keep source code in its own subdirectory
$ mkdir source
$ cd source
# Pull the latest version from github.
$ git clone https://github.com/lee-dohm/discourse.git
$ cd discourse
```

Create `.ruby-version` and `.ruby-gemset` for Discourse:

```bash
$ echo "1.9.3" > .ruby-version
$ echo "discourse" > .ruby-gemset
```

Now it is necessary to leave that directory and re-enter it, so that `rvm` will notice the `.ruby-version` and `.ruby-gemset` files that were just created.

```bash
$ cd ~ && cd ~/source/discourse
```

Install the gems necessary for Discourse:

```bash
$ bundle install
```

## Set Discourse Application Settings

Now you have set the Discourse application settings. The configuration files are in a directory called `config`.  There are sample configuration files included, so you need to copy these files and modify them with your own changes.

```
$ cd ~/source/discourse/config
$ cp ./database.yml.sample ./database.yml
$ cp ./redis.yml.sample ./redis.yml
```

Now you need to edit the configuration files and apply your own settings. 

Start by editing the database configuration file which should be now located at `~/source/discourse/config/database.yml`.

```bash
$ vi ~/source/discourse/config/database.yml
```

Edit the file to add your Postgres username and password to each configuration in the file. Also add `localhost` to the production configuration because the production DB will also be run on the localhost in this configuration.

When you are done the file should look similar to:

```yaml
development:
  adapter: postgresql
  database: discourse_development
  username: admin
  password: <your_postgres_password>
  host: localhost
  pool: 5
  timeout: 5000
  host_names:
    - "localhost"

# Warning: The database defined as "test" will be erased and
# re-generated from your development database when you run "rake".
# Do not set this db to the same as development or production.
test:
  adapter: postgresql
  database: discourse_test
  username: admin
  password: <your_postgres_password>
  host: localhost
  pool: 5
  timeout: 5000
  host_names:
    - test.localhost

# using the test db, so jenkins can run this config
# we need it to be in production so it minifies assets
production:
  adapter: postgresql
  database: discourse
  username: admin
  password: <your_postgres_password>
  host: localhost
  pool: 5
  timeout: 5000
  host_names:
    - production.localhost
```

I'm not a fan of entering the DB password as clear text in the database.yml file. If you have a better solution to this, let me know. 

## Deploy the Database and Start the Server

Now you should be ready to deploy the database and start the server.

This will start the development environment on port 3000.

```bash
$ cd ~/source/discourse
# Set Rails configuration
$ export RAILS_ENV=development
$ rake db:create
$ rake db:migrate
$ rake db:seed_fu
$ thin start
```

I tested the configuration by going to http://discoursetest.org:3000/

## Installing the Production Environment

I'm a Unix and Rails newb (which is why I'm doing this the hard way) so I had a few false starts even before getting things up off the ground.  I currently have the following stack:

* `nginx` as load balancer
* `thin` as web server
* `sidekiq` as worker
* `clockwork` as scheduler
* Using init.d for `nginx` and `thin`
* Using Upstart for `sidekiq` and `clockwork`

You may ask why I'm using two different systems for process maintenance and I will simply refer you to the aforementioned newbness.  I pieced all of this together from various guides and so things look a little patchy because of it. But it works!

### Sources and Links

I used the following sources to get my production installation working:

* [Using Foreman for Production Services](http://michaelvanrooijen.com/articles/2011/06/08-managing-and-monitoring-your-ruby-application-with-foreman-and-upstart/)
* [Setting up Thin](http://stackoverflow.com/questions/3230404/rvm-and-thin-root-vs-local-user) -- *also very useful information on creating RVM wrappers since the `www-data` user won't have Ruby in the PATH*

### Set Up the `www-data` Account

```bash
$ sudo mkdir /var/www
$ sudo chgrp www-data /var/www
$ sudo chmod g+w /var/www
```

### Configure `nginx`

```bash
$ cd ~/source/discourse/
$ sudo cp config/nginx.sample.conf /etc/nginx/sites-available/discourse.conf
```

Edit `/etc/nginx/sites-available/discourse.conf` and set `server_name` to the domain you want to use. When done, enable the site.

```bash
$ sudo vi /etc/nginx/sites-available/discourse.conf
$ sudo ln -s /etc/nginx/sites-available/discourse.conf /etc/nginx/sites-enabled/discourse.conf
$ sudo rm /etc/nginx/sites-enabled/default
$ sudo service nginx start
```

### Set a secret session token

```bash
$ rake secret
```

Now copy the output of the `rake secret` command, open `config/initializers/secret_token.rb` in your text editor, and:

* Erase all code in that file
* Paste the token from `rake secret` in this code (replace [TOKEN]):

```ruby
Discourse::Application.config.secret_token = "[TOKEN]"
```

### Create Production Database

```bash
$ export RAILS_ENV=production
$ rake db:create db:migrate db:seed_fu
```

### Deploy Discourse App to `/var/www`

```bash
$ export RAILS_ENV=production
$ rake assets:precompile
$ sudo -u www-data cp -r ~/source/discourse/ /var/www
$ sudo -u www-data mkdir /var/www/discourse/tmp/sockets
```

### Configure `thin`

I set up the `rvm` wrapper for Ruby v1.9.3, but you can configure it for whatever version you decide to use. *Note:* `rvmsudo` executes a command as `root` but with access to the current `rvm` environment.

```bash
$ cd /var/www/discourse
$ rvmsudo thin install
$ rvmsudo thin config -C /etc/thin/discourse.yml -c /var/www/discourse --servers 4 -e production
$ rvm wrapper 1.9.3@discourse bootup thin
```

After generating the configuration, you'll need to edit (using `sudo`) the `/etc/thin/discourse.yml` file to change from `port` to `socket` to make things work with the default `nginx` configuration.  Just replace the line `port: 3000` with:

```yaml
socket: tmp/sockets/thin.sock
```

Then you'll need to edit the `thin` init script:

```bash
$ sudo vi /etc/init.d/thin
```

Change the line starting with `DAEMON=` to:

```text
DAEMON=/usr/local/rvm/bin/bootup_thin
```

Start the service:

```bash
$ sudo service thin start
```

### Use `foreman` to help configure `sidekiq` and `clockwork`

As of this writing, Discourse comes with a sample `Procfile` for `foreman`.  On the other hand, `foreman` is not part of the `Gemfile`, so you will need to install it manually:

```bash
$ gem install foreman
```

Once that is complete then you can use `foreman` to generate the Upstart configuration:

```bash
$ rvmsudo foreman export upstart /etc/init -a discourse -u www-data
```

This will create a number of files in your `/etc/init` directory that all start with the name `discourse`.  Since we are using `init.d` to handle `thin`, we should remove the configuration for `thin` in Upstart:

```bash
$ sudo rm /etc/init/discourse-web*
```

Then we need to create `rvm` wrappers for `sidekiq` and `clockwork` so that the `www-data` user can execute these tools:

```bash
$ rvm wrapper 1.9.3@discourse bootup sidekiq
$ rvm wrapper 1.9.3@discourse bootup clockwork
```

Finally, all we need to do is update the `/etc/init/discourse-clockwork-1.conf` and `/etc/init/discourse-sidekiq-1.conf` files to use the `rvm` wrapper for launching the tools.

#### `discourse-clockwork-1.conf`

```text
start on starting discourse-clockwork
stop on stopping discourse-clockwork
respawn

exec su - www-data -c 'cd /var/www/discourse; export PORT=5200; /usr/local/rvm/bin/bootup_clockwork config/clock.rb >> /var/log/discourse/clockwork-1.log 2>&1'
```

#### `discourse-sidekiq-1.conf`

```text
start on starting discourse-sidekiq
stop on stopping discourse-sidekiq
respawn

exec su - www-data -c 'cd /var/www/discourse; export PORT=5100; /usr/local/rvm/bin/bootup_sidekiq -e production >> /var/log/discourse/sidekiq-1.log 2>&1'
```

#### Start the Services

```bash
$ sudo start discourse
```

### Create Discourse Admin Account

* Logon to the site and create an account using the application UI
* Now make that account the admin:

```bash
# Start the Rails console
$ rails c
```

```ruby
# Take the first (only) user account and mark it as admin
u = User.first
u.admin = true
u.save

# Create a confirmation for their email address, if necessary
token = u.email_tokens.create(email: u.email)
EmailToken.confirm(token.token)
```

## Upgrading Versions in Production

### Stop the Servers

```bash
$ sudo stop discourse
$ sudo service nginx stop
$ sudo service thin stop
```

### Pull Down Latest Code and Update Application

```bash
$ cd ~/source/discourse
$ git pull
$ export RAILS_ENV=production
$ rake db:migrate db:seed_fu assets:precompile
$ sudo -u www-data cp -r ~/source/discourse /var/www
```

### Start the Servers

```bash
$ sudo service thin start
$ sudo service nginx start
$ sudo start discourse
```

## TROUBLESHOOTING

### You get complaints in the logs about gems that haven't been checked out

To solve this:

```bash
$ cd ~/source/discourse
$ bundle pack --all
$ bundle install --path vendor/cache
```

After that follow the update instructions in the previous section.

## TODO

* Fix `bundle exec` issue
* Add [Ruby tuning recommendations](http://meta.discourse.org/t/tuning-ruby-and-rails-for-discourse/4126)
* Convert `thin` to use Upstart for process monitoring
* Convert `nginx` to use Upstart for process monitoring?
* Convert to using Ruby 2.0
* Add script to create admin Discourse account
* Add scripts to automate a lot of this process
