---
layout: post
title:  "Deploy your Rails app to DigitalOcean with Capistrano"
date:   2015-06-13
comments: on
---

I recently deployed the project I've been working on to DigitalOcean. I'd only ever used Heroku before, and there was definitely a learning curve. The simplicity of Heroku is fantastic, but it definitely comes at a cost (the cost being money). My specific reason for setting up this project on DigitalOcean was that I needed to use an SSL certificate, and the Heroku add-ons that allow you to do this are pretty pricey.

Getting set up on DigitalOcean ultimately wasn't a difficult process, but it took a lot of Googling, and Stack Overflowing, and piecing together bits of various how-to guides that only got me about 90% of the way there in series of process for all of which the last 10% was as time-consuming as the first 90%. So, usual computer stuff.

At first I assumed that this was a pretty rudimentary skill, and everyone I worked with probably already knew what an Nginx config file looked like, and how to set up Capistrano, but after asking around, I learned that lots of people had never done this before either. This blog post is adapted from an engineering talk I gave [at work](https://www.shopify.com/careers) on how to set up a Rails app on DigitalOcean, deploy with Capistrano, and get that lil' green lock next to your website address. :green_heart::lock:

I'll try to cover all of the pitfalls I hit along the way - all of the problems I had are easy to address up front, once you've determined what the problem even is, and what the correct way to fix it is (usually after investigating at least one red herring, and trying a handful of potential solutions).

# Deploying a Rails app to DigitalOcean with Capistrano (and setting up SSL!)

DigitalOcean projects or instances are called 'Droplets' - you can opt for an empty Ubuntu box, but they also have options for droplets with a pretty wide selection of base images. We're going to be using their [Rails droplet](https://www.digitalocean.com/features/one-click-apps/ruby-on-rails/) - this gives us an Ubuntu box with Nginx, Unicorn, Ruby, and Rails already installed. My older droplet (as in a couple of months old) came with Mysql2, but my more recent droplet had Postgres out of the box, so it looks like the base image changes every now and then, but most of this guide should be applicable either way.

## Enable swap :computer::point_right::floppy_disk:

I chose the $5/month droplet, which doesn't have the *most* compute behind it. I pretty quickly started running out of memory - creating a swap file will address this.

The following commands will create a 4gb swap file, adjust some permissions, and enable the swap file to be used as swap space:

```
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo swapon -s
```

Next, open up `/etc/fstab` and add the following line to make our changes persistent:

```
/swapfile   none    swap    sw    0   0
```

We can then verify that everything is working, and check out our memory usage:

```
root@moon-church:~# sudo swapon -s
Filename        Type    Size  Used  Priority
/swapfile                               file    4194300 163872  -1

root@moon-church:~# free -m
             total       used       free     shared    buffers     cached
Mem:           490        448         41          2         54         80
-/+ buffers/cache:        313        176
Swap:         4095        160       3935

```

You can also open up `sudo nano /etc/sysctl.conf` and add a couple of extra lines here:

```
vm.swappinss=10
vm.vfs_cache_pressure=50
```

The `swappiness` option is pretty much exactly what it sounds like - this will adjust the tendency to swap to disk (the number is out of 100). We'll keep this low. For `vfs_cache_pressure`, this specifies how much caching to do on inode and dentry information. [This is system stuff, like metadata about files](http://unix.stackexchange.com/questions/4402/what-is-a-superblock-inode-dentry-and-a-file), and is pretty well-suited to caching. For this, a higher number means faster removal from the cache.

You can read more about what we just did [here](https://www.digitalocean.com/community/tutorials/how-to-add-swap-on-ubuntu-14-04).

## Configuring Nginx and Unicorn :icecream::horse:

(<small>:icecream::horse: indicates a unicorn</small>)

There's three main files we'll be concerned with here:

```
/etc/nginx/sites-enabled/rails
/etc/unicorn.conf
/etc/default/unicorn
```

These files are all pretty small, and can show you a lot about how Nginx and Unicorn work together to serve your website.

We don't actually have to do much config for either Nginx or Unicorn. What we will do is change the directory that they look for our Rails app in, so that it plays well with Capistrano.

When we set up Capistrano, it will give us a directory structure on our server that will look something like this:

```
/home/rails
  - /home/rails/current
  - /home/rails/releases
    - /home/rails/releases/20150624185638
    - /home/rails/releases/20150624185811
    - / home/rails/releases/20150625041325
```

Our app will be served out of `current`, where `current` is symlinked to the most recent release. We can also tell Capistrano to keep around a couple previous releases in case we need to revert to an old version. You can read more about Capistrano folder structure [here](http://capistranorb.com/documentation/getting-started/structure/).

Open up your Nginx configuration - in one of my droplets, it's located at `/etc/nginx/sites-enabled/default`, but my other droplet uses `/etc/nginx/sites-enabled/rails`. If you're starting with a fresh droplet, there should only be one file in the `sites-enabled` folder, so that will be the one you're looking for.

This file shows us a bit about what Nginx does - we can see that for now, it has just one server block (`server { .... }`) that listens on port 80, points to our root directory, and specifies some things like where to look for an index page. We can also see that Nginx proxies requests to `127.0.0.1:8080`.

For now, we'll just update the root folder. By default, it will be something along the lines of `/home/rails` - modify it to add `/current` to whatever your existing folder structure is, to reflect the Capistrano folder structure we saw up above:

```
root /home/rails/current;
```

We'll come back to this file later to add our SSL certificate.

Open up `/etc/unicorn.conf`. The first thing that we'll see is that it's listening on `127.0.0.1:8080`, which is where we saw that Nginx is proxying requests to. Neat. The fog is clearing. :sun_with_face:

Next we can see that this is the place to specify how many worker processes Unicorn runs, and also where the logs end up.

As we did in the Nginx config, update the `working_directory`:

```
working_directory "/home/rails/current"
```

Next, move on to `/etc/default/unicorn`:

Here, we'll update the `APP_ROOT`. Now everything is all looking for our app in the same location.

```
APP_ROOT=/home/rails/current
```

Moving on!

## Create a deploy user :cop::envelope::mailbox:

We don't want to deploy as root, so we'll make a deploy user on our server:

```
groupadd deploy
adduser deploy --ingroup deploy
sudo adduser deploy sudo
sudo chown deploy:deploy /home/rails
su deploy
```

I also had to `chown deploy:deploy /usr/local/rvm/gems/` to get some things to work, but I'm unsure if this was a ramification of accidentally running `bundle install` as root. Don't run `bundle install` as root :no_good:. It might cause problems down the line. If in doubt, `chown` it out (<small>not an actual turn of phrase</small>)! But seriously, I'm not sure if that's bad practice. It's hard to find out if the things you're doing on your server are horribly bad practice, or the actual correct way to do things. But, this is the path we chose for ourselves.

## Install some gems :sparkles::gem::sparkles:

Time to add Capistrano to your app. In your `Gemfile`:

```
gem 'capistrano', '~> 3.3.0'
gem 'capistrano-bundler', '~> 1.1.2'
gem 'capistrano-rails'
gem ‘capistrano-rvm'

group :production do
  gem 'pg'
end
```

And then in your local terminal:

```
cap install
```

True fact: There's a [capistrano-rvm](https://github.com/capistrano/rvm/) (maintained by Capistrano) and also an [rvm-capistrano](https://github.com/rvm/rvm-capistrano) (maintained by RVM). That was not at all confusing the first time I did this, because I am an infallible computer genius.

Here's what my `Capfile` looks like:

```
# Load DSL and set up stages
require 'capistrano/setup'

# Include default deployment tasks
require 'capistrano/rvm'
require 'capistrano/deploy'
require 'capistrano/bundler'
require 'capistrano/rails/assets'
require 'capistrano/rails/migrations'

# Load custom tasks from `lib/capistrano/tasks' if you have any defined
Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
```

Be sure to uncomment `require 'capistrano/rvm'` in this file. Next, open up your `config/deploy/production.rb` and fill in the details for your server:

```
# Define roles, user and IP address of deployment server
# role :name, %{[user]@[IP adde.]}
role :app, %w{deploy@101.010.101.010}
role :web, %w{deploy@101.010.101.010}
role :db,  %w{deploy@101.010.101.010}

# Define server(s)
server '101.010.101.010', user: 'deploy', roles: %w{web}

# SSH Options
# See the example commented out section in the file
# for more options.
set :ssh_options, {
    forward_agent: true,
    auth_methods: %w(password),
    password: 'alphafoxtrotbravobravogamma';
    user: 'deploy',
}
```

Note that we're using our deploy user here, rather than root. You should check out Capistrano's guide on [authenticatoin and authorization](http://capistranorb.com/documentation/getting-started/authentication-and-authorisation/) and set up SSH keys for this, but for now we can put the password here, and add the file to our `.gitignore`.

If you're having SSH key problems pulling down your code from GitHub to your server, check that you've set '`forward_agent: true` - this will forward your local SSH session, so that you don't have to set up your GitHub SSH keys on the server as well as your local machine.

Finally, my `deploy.rb` looks like this:

```
lock '3.3.5'

set :stage, :production

set :application, 'moonchurch'
set :repo_url, 'https://github.com/funionnn/moonchurch.git'
set :deploy_to, '/home/rails'
set :scm, :git
set :branch, 'master'

set :group, 'deploy'
set :use_sudo, false
set :rails_env, 'production'
set :deploy_via, :copy

set :linked_files, %w{config/database.yml config/secrets.yml}

set :format, :pretty
set :log_level, :debug
set :keep_releases, 5

namespace :deploy do
  after :finishing, 'deploy:cleanup'
  after 'deploy:publishing', 'deploy:restart'
end

desc "Symlink shared config files"
task :symlink_config_files do
    run "#{ try_sudo } ln -s #{ deploy_to }/shared/config/database.yml #{ current_path }/config/database.yml"
    run "#{ try_sudo } ln -s #{ deploy_to }/shared/config/secrets.yml #{ current_path }/config/secrets.yml"
end
```

You can see here how I've handled my `database.yml` and `secrets.yml` - I copied them to `/home/rails/shared/config` on the server, and then symlink to them in the deploy process.

That's it!  You're ready!

```
princess-bubblegum:moonchurch lydia$ bundle exec cap production deploy
```
:tada::sparkles::tada::sparkles::tada::sparkles::tada::sparkles::tada::sparkles:

You should see a bunch of output in your terminal, and your app should successfully deploy to your droplet. Almost done.

## Set up your SSL cert :lock::muscle::no_good::computer::skull::fire::lock:

First off, you're going to need an SSL cert. I :money_with_wings: paid for mine :money_with_wings: like a chump. However, you can get free (non-wildcard) certificates from [StartSSL](https://www.startssl.com/), and [Let's Encrypt](https://letsencrypt.org/) is mere months away.

Next, upload your cert to your server (mine live in `/etc/nginx/cert`), and open up your Nginx configuration (`/etc/nginx/sites-enabled/default`) one last time:

```
server {
  listen 80;
  server_name yourname.cool;
  return 301 https://$server_name$request_uri;
}

server {
  listen 443 ssl;

  ssl on;
  ssl_certificate /etc/nginx/cert/STAR_yourname_cool.pem;
  ssl_certificate_key /etc/nginx/cert/STAR_yourname_cool.key;

  root /home/rails/current;
  server_name you.cool;
  index index.htm index.html;

  location / {
    try_files $uri/index.html $uri.html $uri @app;
  }

  location ~* ^.+\.(jpg|jpeg|gif|png|ico|zip|tgz|gz|rar|bz2|doc|xls|exe|p$
    try_files $uri @app;
  }

   location @app {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://127.0.0.1:8080;
  }
}
```

This was a little bit tricky - it's easy to end up in redirect loops, but it should only take a couple extra lines. I split the config into two server blocks - the first listens on port 80, just like before, but now this server block just does a 301 redirect to HTTPS. The second server block listens on port 443, and I added the following lines:

```
ssl on;
ssl_certificate /etc/nginx/cert/STAR_you_cool.pem;
ssl_certificate_key /etc/nginx/cert/STAR_you_cool.key;
```
Just turn on SSL (Easy! :information_desk_person:) and tell Nginx where to find your certificate, and then under `location @app`:
```
proxy_set_header X-Forwarded-Proto https;
```

That's it! Now you're done for real! Almost!

## Polish it up :radio::musical_note::nail_care:

You'll probably want to turn on some log rotation and so on, especially if you're only paying for the $5 droplet, and *especially* if you're already down 4gb of hard drive space for the swapfile we made earlier. Read about managing log files [here](https://www.digitalocean.com/community/tutorials/how-to-manage-log-files-with-logrotate-on-ubuntu-12-10).

You'll also probably want some kind of backups - read about DigitalOcean's backups [here](https://www.digitalocean.com/community/tutorials/understanding-digitalocean-droplet-backups).


