# Discourse 安装完全中文指南

#### 相关说明

[Discourse](http://discourse.org) 意为“谈话”，是由 Stack Overflow 的联合创始人 Jeff Atwood 推出的下一代开源论坛程序，关于它的介绍可以看看 [OSChina](http://www.oschina.net/p/discourse) 和 [36kr](http://www.36kr.com/p/201256.html) 的报道。目前，网络上还没有一份详细、全面的中文 Discourse 安装指南，ofGEEK 特此整理编写本文，希望能够对有需要的人有所帮助。

#### 本指南在以下环境安装成功

- 一台安装 Ubuntu 12.10 x86 操作系统的 VPS 服务器
- 内存 1G （Discourse 官方推荐的最小值）

## 安装并测试 Discourse

**说明：为了操作的安全，不推荐直接使用 root 账户安装 Discourse，建议在 VPS 上增加一个用户，本文设为 admin，你可以任意设定喜欢的名称。本文将 VPS 的主机名设为 ofgeek.com，读者请根据实际情况自行修改。**

### 登录 VPS

用 root 登录 VPS，创建新用户 admin，并将其加入 sudo 组，以获取管理员权限。（关于登录 VPS 的方法，本文不做详细介绍，如有需要请自行查阅。Mac OS X 或 Linux 用户可以直接打开终端使用 ssh 进行连接。Windows 用户推荐使用 [Xshell](http://www.netsarang.com/products/xsh_overview.html)，非商业用途可免费使用）。


```bash
$ sudo adduser admin
$ sudo adduser admin sudo
```

退出 root 账户，用 admin 账户重新登录 VPS。

```bash
$ logout
$ ssh admin@ofgeek.com
```

### 安装 Discourse 所需的系统包文件

```bash
$ sudo apt-get install git-core build-essential postgresql postgresql-contrib libxml2-dev libxslt-dev libpq-dev redis-server nginx postfix
```

在安装过程中，会弹出 Postfix 的配置界面，它是 Linux 系统中管理邮件程序，一般情况下请选择"Internet Site"即可，然后在下一步输入你自己的域名，在本文中则是 ofgeek.com。

###设定主机名称

这里需要用到文本编辑器，你可以根据自己的喜好使用 vi 或者 emacs，Ubuntu 12.10 系统里默认的是 nano。

```bash
$ sudo nano /etc/hosts
```

在第一行后面加入自己的域名，本文修改成：

```bash
127.0.0.1  localhost ofgeek.com
```

确认无误后，按 Ctrl+X 保存，按 y 确认后退出。

### 配置数据库

Discourse 使用 Postgres 数据库。为数据库增加用户 admin，并设置好密码。请将密码保存好，后面的设置中还需用到。

```bash
$ sudo -u postgres createuser admin -s -P
```

### 安装并配置 RVM 和 Ruby

本文使用 RVM 来安装和管理 Ruby 及相关的 Gem。

首先，安装 RVM，并将 admin 用户加入 rvm 组。

```bash
$ \curl -L https://get.rvm.io | sudo bash -s stable
$ sudo adduser admin rvm
```

然后注销一次，再重新登录，使得变更生效，并激活 RVM。

```bash
$ logout
$ ssh@ofgeek.com
```

再安装 Ruby v1.9.3 并为 Discourse 创建一个 Gem 集。

```bash
$ rvm install 1.9.3
$ rvm use --default 1.9.3
$ rvm gemset create discourse
```

### 获取 Discourse 源代码并搭建安装环境

在 admin 的主目录里创建一个 source 目录，使用 git 将 Discourse 的源代码放在其中，然后进入该目录。

```bash
$ mkdir source
$ cd source
$ git clone https://github.com/discourse/discourse.git
$ cd discourse
```

为 Discourse 创建 .ruby-version 和 .ruby-gemset：

```bash
$ echo "1.9.3" > .ruby-version
$ echo "discourse" > .ruby-gemset
```

退到 admin 主目录，然后再次进入 Discourse 目录，以便 rvm 对 .ruby-version 和 .ruby-gemset 的管理生效。

```bash
$ cd ~ && cd ~/source/discourse
```

安装 Discourse 所需的 Gem：

```bash
$ bundle install
```

### 修改 Discourse 的相关配置文件

Discourse 所有的配置文件都放在 config 目录中，其中带有 sample 字样的都是范例文件，按照本文的安装过程仅需修改 database.yml 和 redis.yml 即可。

```
$ cd ~/source/discourse/config
$ cp ./database.yml.sample ./database.yml
$ cp ./redis.yml.sample ./redis.yml
$ nano ./database.yml
```

本文修改后的 database.yml 如下，请结合你的实际情况加以修改。

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
    - "ofgeek.com"
```

### 部署开发数据库并启动服务

为了确保 Discourse 的安装环境正确，在正式部署生产环境之前，可以先启动开发模式（development）进行检测。

```bash
$ cd ~/source/discourse
# 进入开发模式
$ export RAILS_ENV=development
# 生成数据库
$ rake db:create
# 导入初始化数据
$ psql discourse_development < pg_dumps/production-image.sql
$ rake db:migrate
$ rake db:seed_fu
# 启动 Thin 服务器
$ thin start
```

此时，在浏览器地址栏中输入 http://ofgeek.com:3000/ 并回车，如果一切正确，那么你应该可以看到 Discourse 的开发模式界面了。确认无误后，可以按 Ctrl+C 停止开发模式，并进入下一步。

## 部署生产（production）环境

生产环境即正式运行的环境，在生产环境中，本文使用了以下服务：

* nginx 作为负载均衡
* thin 作为服务器
* sidekiq 负责任务排序
* clockwork 负责任务调度
* init.d 负责管理 nginx 和 thin
* Upstart 负责管理 sidekiq 和 clockwork

### 生产环境目录

Discourse 默认的生产环境目录为 /var/www，如果系统中没有，则新建一个，将其加入 www-data 组，并设置相应权限。

```bash
$ sudo mkdir /var/www
$ sudo chgrp www-data /var/www
$ sudo chmod g+w /var/www
```

### 修改 nginx 配置文件

nginx 的配置文件范例也在 config 目录中，仅需将其中的 server_name 改为你自己的域名即可。

```bash
$ cd ~/source/discourse/
& nano config/nginx.sample.conf
# 改完后将其复制 nginx 配置文件目录中，并改名为 discourse.conf
$ sudo cp config/nginx.sample.conf /etc/nginx/sites-available/discourse.conf
# 建立一个连接
$ sudo ln -s /etc/nginx/sites-available/discourse.conf /etc/nginx/sites-enabled/discourse.conf
# 删除其它所有 nginx 的默认配置文件
$ sudo rm /etc/nginx/sites-enabled/default
```

### 生成密钥会话令牌

```bash
$ rake secret
```

将生成的密钥记下来，打开 config/initializers/secret_token.rb 文件，执行以下步骤：

* 清空该文件中的所有已有内容
* 将下面这行代码拷贝到该文件中，用刚才生成的密钥代替 [TOKEN] 部分

```ruby
Discourse::Application.config.secret_token = "[TOKEN]"
```

### 建立生产环境数据库

```bash
$ export RAILS_ENV=production
$ rake db:create db:migrate db:seed_fu
# 导入初始化数据
$ psql discourse < pg_dumps/production-image.sql
```

### 将 Discourse 部署到 /var/www

```bash
# 编译源文件
$ rake assets:precompile
$ sudo -u www-data cp -r ~/source/discourse/ /var/www
$ sudo -u www-data mkdir /var/www/discourse/tmp/sockets
```

### 配置 thin 服务器

```bash
$ cd /var/www/discourse
$ rvmsudo thin install
$ rvmsudo thin config -C /etc/thin/discourse.yml -c /var/www/discourse --servers 4 -e production
$ rvm wrapper 1.9.3@discourse bootup thin
```

配置文件生成后，需要修改 /etc/thin/discourse.yml 文件，将其中的：

```
port: 3000
```

修改为：

```yaml
socket: tmp/sockets/thin.sock
```

还需修改 Thin 的初始化脚本：

```bash
$ sudo vi /etc/init.d/thin
```

将 DAEMON= 那一行改为：

```text
DAEMON=/usr/local/rvm/bin/bootup_thin
```



### 使用 foreman 来配置 sidekiq 和 clockwork

安装 foreman 并生成 Upstart 配置文件：

```bash
$ gem install foreman
$ rvmsudo foreman export upstart /etc/init -a discourse -u www-data
```

本文使用 init.d 来管理 Thin 服务器，所以不需要 Upstart 生成的 Thin 相关配置文件，将其删除：

```bash
$ sudo rm /etc/init/discourse-web*
```

为 sidekiq 和 clockwork 生成 rvm 封装：

```bash
$ rvm wrapper 1.9.3@discourse bootup sidekiq
$ rvm wrapper 1.9.3@discourse bootup clockwork
```

修改 /etc/init/discourse-clockwork-1.conf 和 /etc/init/discourse-sidekiq-1.conf 文件，便于使用 rvm 来管理：

#### discourse-clockwork-1.conf 修改后如下：

```text
start on starting discourse-clockwork
stop on stopping discourse-clockwork
respawn

exec su - www-data -c 'cd /var/www/discourse; export PORT=5200; /usr/local/rvm/bin/bootup_clockwork config/clock.rb >> /var/log/discourse/clockwork-1.log 2>&1'
```

#### discourse-sidekiq-1.conf 修改后如下：

```text
start on starting discourse-sidekiq
stop on stopping discourse-sidekiq
respawn

exec su - www-data -c 'cd /var/www/discourse; export PORT=5100; /usr/local/rvm/bin/bootup_sidekiq -e production >> /var/log/discourse/sidekiq-1.log 2>&1'
```

### 启动所有服务

```bash
$ cd /var/www/discourse
$ sudo service thin start
$ sudo service nginx start
$ sudo start discourse
$ sudo clockworkd -c config/clock.rb start
$ bundle exec sidekiq -d -L ~/source/discourse/log/sidekiq.log
```

到这里，生产环境就部署完成了。此时用浏览器打开 http://www.ofgeek.com ，就能看到正式的 Discourse 界面。论坛管理员的初始用户名和密码分别是：

```text
username: forumadmin
password: password
```

登录进去之后，改为更加安全的密码。

### 启用中文支持

目前，Discourse 已经可以支持中文界面，使用管理员账户登录后，点击右上角的管理员名称进入设置界面，然后再点击右上角的小扳手 Admin 进入系统管理界面，在第二项 Settings 中，把 default_locale 从默认的 en 更改为 zh_CN，然后返回论坛主界面，按 Ctrl+F5 刷新浏览器缓存，中文界面就出来了。

**现在，开始享受你的 Discourse 之旅吧！**

## 升级 Discourse

由于 Discourse 目前仍处于 beta 测试阶段，文件升级更新非常频繁，建议经常升级以保持最佳状态。

### 停止相关服务

```bash
$ sudo stop discourse
$ sudo service nginx stop
$ sudo service thin stop
```

### 更新、编译并部署源文件

```bash
$ cd ~/source/discourse
$ git pull
$ bundle install
$ export RAILS_ENV=production
$ rake db:migrate db:seed_fu assets:precompile
$ sudo rm -r -f /var/www/discourse/.git
$ sudo -u www-data cp -r ~/source/discourse /var/www
```

### 重新启动相关服务

```bash
$ sudo service thin start
$ sudo service nginx start
$ sudo start discourse
```

## 致谢

本指南基于以下资源创建，特此表示感谢：

- Discourse 官方[开发者指南](https://github.com/discourse/discourse/blob/master/docs/DEVELOPER-ADVANCED.md)。
- Christopher Baus <christopher@baus.net> 的 Discourse [安装指南（英文）](https://github.com/baus/install-discourse)。
- Lee Dohm <lee@liftedstudios.com> 的 Discourse [安装指南（英文）](https://github.com/lee-dohm/install-discourse)。

本指南遵循创作共用协议，欢迎转载，请注明出处及原始链接。

&copy; ofgeek.com 最初发布于 2013.5.21 。
