---
title: Deploying Ruby on Rails on Ubuntu
date: 2022-07-12 13:54:50 +0500
categories: [Ubuntu, Deployment]
tags: [ruby-on-rails]     # TAG names should always be lowercase
---

# Deploying Ruby on Rails on Ubuntu

I always wanted to learn how the deployment and server provisioning is done. While setting up a homelab using Raspbery Pi 4, which uses an ARM based architecture and many deployment tools do not support that yet so I thought this is perfect oppertuity to finally learn.

This is my first ever home lab setup:

- I use 4, Raspbery Pi 4 with 4Gig of RAM.
- The case I am using is from [UCTRONICS](https://www.amazon.com/UCTRONICS-Upgraded-Enclosure-Raspberry-Removable/dp/B09JNHKL2N).
- For operating system I am using Ubuntu server `22.04`.

While configuring the Raspberry Pi to host my Ruby on Rails sites, these are the steps I had to go through. 


# Secure the remote server with SSH

## Install the copy utility
Make sure `ssh-copy-id` is installed on Mac, it does not come in mac by default.

```bash
brew install ssh-copy-id
```

## Copy command
The following command will prompt for password

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub user@remote-host-ip
```

You should now be able to ssh into remote server without needing to provide your password.

## Disable password based login

Inside the `sshd_config` file set the `PasswordAuthentication` to `no`

```bash
sudo vim /etc/ssh/sshd_config
```

## Restart the SSH service

```bash
sudo systemctl restart ssh
```

# Installing Ruby & Rails dependencies

## Node.js

### Node version manager

```bash
sudo apt install curl 
curl https://raw.githubusercontent.com/creationix/nvm/master/install.sh | bash
```

Run the following command so `nvm` command is available

```bash
source ~/.profile 
```

### Install node version

Install the LTS version

```bash
nvm install --lts
```

## Install Yarn
 
 ```bash
 sudo apt install yarn
 sudo apt install yarn
 ```
 
## Install Redis server
 
 ```bash
 sudo apt install redis-server
 sudo apt install redis-tools

 sudo systemctl status redis-server
 ```
 
 ### Make sure it is up and running
 
 ```bash
 sudo systemctl status redis-server
 ```
 
## Dependencies for compiiling Ruby along with Node.js and Yarn 
 
 ```bash
 sudo apt install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev software-properties-common libffi-dev dirmngr gnupg apt-transport-https ca-certificates
 ```
 
## Install Ruby

### rbenv 

Run each command separately

 ```bash
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
git clone https://github.com/rbenv/rbenv-vars.git ~/.rbenv/plugins/rbenv-vars
exec $SHELL
 ```
 
### Ruby

 ```bash
 ubuntu@ubuntu:~$ rbenv install 3.1.2
 ubuntu@ubuntu:~$ rbenv global 3.1.2
 ubuntu@ubuntu:~$ ruby -v
 ```
 
### Bundler

```bash
gem install bundler
```

## Puma
 
 ```bash
 sudo apt -y install puma
 ```
 
## Nginx
 
 ```bash
 sudo apt -y install nginx
 ```
 
### Nginx site

#### Remove defailt site

 ```bash
 sudo rm /etc/nginx/sites-enabled/default
 ```
 
#### Enable your site
 
 ```bash
 sudo vim /etc/nginx/sites-enabled/app-name
 ```
 
#### Add site configuration

```
upstream puma_your_app_name {
  server unix:///home/ubuntu/freeapi/shared/tmp/sockets/puma.sock;
}

server {
  listen 80 default_server deferred;
  server_name freeapi.com www.freeapi.com;

  # Don't forget to update these, too
  root /home/ubuntu/your_app_directory/current/public;
  access_log /home/ubuntu/your_app_directory/current/log/nginx.access.log;
  error_log /home/ubuntu/your_app_directory/current/log/nginx.error.log info;

  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }

  try_files $uri/index.html $uri @puma_your_app_name;
  location @puma_your_app_name {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;

    proxy_pass http://puma_your_app_name;
  }

  error_page 500 502 503 504 /500.html;
  client_max_body_size 10M;
  keepalive_timeout 10;
}
```

#### Restart Nginx service

```bash
sudo service nginx restart
```

If the service fails to start check the logs to see what the error is. The status command will also provide helpful information `systemctl status nginx.service`. At this stage the error might be related to nginx log files not being available, thats because we did not deploy the app yet. If you want you can comment the `access_log` and `error_log` line in the above confirutaion, restart the nginx service and then check if it is running successfully.

## Install PostgreSQL

```bash
sudo apt install postgresql postgresql-contrib libpq-dev
```

If you see a lock error then simply reboot and run the above command again.

### Create the PostgreSQL user and database

```bash
sudo su - postgres
createuser --pwprompt ubuntu
createdb -O ubuntu your-app-db
exit
```


## Deployment

### Capistrano

Add the following gems to the Gemfile and run the command `bundle install`

```ruby
  gem 'capistrano', require: false
  gem 'capistrano-rails', require: false
  gem 'capistrano-puma', require: false
  gem 'capistrano-rbenv', require: false
  gem 'capistrano-bundler', require: false
```

Generate the deployment files

```bash
cap install STAGES=production
```

This will generate the following files

```bash
Capfile
config/deploy.rb
config/deploy/production.rb
```

Add the following in `Capfile`

```ruby
require "capistrano/rails"
require "capistrano/rbenv"
require "capistrano/bundler"
require "capistrano/rails/assets"
require "capistrano/rails/migrations"

require "capistrano/puma"
install_plugin Capistrano::Puma
install_plugin Capistrano::Puma::Systemd

set :rbenv_type, :user
set :rbenv_ruby, '3.1.2'
```

Add the following inside `config/deploy.rb`

```ruby
set :application, "your-app-name"
set :repo_url, "git@github.com:username/myapp.git"

set :deploy_to, "/home/ubuntu/#{fetch :application}"

append :linked_dirs, 'log', 'tmp/pids', 'tmp/cache', 'tmp/sockets', 'vendor/bundle', '.bundle', 'public/system', 'public/uploads'
```

Add the following inside `config/deploy/production`

```ruby
set :branch, "main"
server "your-ip-server", user: "ubuntu", roles: %w[web app db]

set :ssh_options, {
  keys: %w(/Users/your-user/.ssh/id_ed25519.pub),
  forward_agent: true,
  auth_methods: %w(publickey)
}
```

Make sure the branch you want to deploy is already pushed on GitHub.

Setup env vars

SSH into the remote server 

```bash
mkdir /home/ubuntu/your-app
nano /home/ubuntu/your-app/.rbenv-vars
```

Add the DB and Master key env vars

```
# For Postgres
DATABASE_URL=postgresql://user:PASSWORD@127.0.0.1/myapp

RAILS_MASTER_KEY=xyz123

RACK_ENV=production
RAILS_ENV=production
```

Upload puma config

```bash
cap production puma:config
```
 
 Upload puma service
 
 ```bash
 cap production puma:systemd:config puma:systemd:enable

 ```
 
## Troubleshooting

### Removing the repository from Ubuntu 
 This guide is for Ubuntu `22.04`, if you end up adding the wrong repo which is not compatiable for example `sudo add-apt-repository ppa:chris-lea/redis-server` just remove it by using the `--remove` flag `sudo add-apt-repository --remove ppa:chris-lea/redis-server`
 
### Puma

If you see the following error on deployment

```
NameError: uninitialized constant Capistrano::Puma
install_plugin Capistrano::Puma
                         ^^^^^^
/Users/shairyar/Sites/fakeapi/Capfile:44:in `<top (required)>'
```

Make sure you are using the gem `capistrano3-puma` and not `capistrano-puma`.

### GitHub permission issue

Just make sure the new secure ssh key `id_ed25519.pub` is added to remote server and inside `config/deploy/production` key `forward_agent` is set to `true`

```ruby
set :ssh_options, {
  keys: %w(/Users/shairyar/.ssh/id_ed25519.pub),
  forward_agent: true,
  auth_methods: %w(publickey)
}
```

### Yarn install issue

Running the following command shold fix the problem
`sudo apt-get remove cmdinstall;sudo apt update;sudo apt-get install yarn`

## Contribute

Found a mistake? Or if you have any questions, please feel free to open an issue [here](https://github.com/shairyar/shairyar.github.io/issues/new).

Found something that can be improved? Feel free to [create a PR here](https://github.com/shairyar/shairyar.github.io).

