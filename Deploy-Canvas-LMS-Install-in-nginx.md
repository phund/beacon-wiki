##Installing Ruby-Brightbox
first remove all older version of ruby if installed mistakenly

    sudo apt-get remove ruby1.9.3
    sudo apt-get purge ruby1.9.3
    sudo apt-get remove ruby1.9.1
    sudo apt-get purge ruby1.9.1

for installing ruby brightbox create a repository using following command

    sudo apt-add-repository ppa:brightbox/ruby-ng-experimental
    sudo apt-get update
    sudo apt-get install ruby1.9.3


then make sure that installed ruby version is brightbox by the command

    ruby -v
    ruby 1.9.3p550 (2014-10-27) [x86_64-linux] Brightbox

## Installing Postgres
Follow [Postgresql install ](https://github.com/m-narayan/beacon/wiki/Install-Postgresql) install to make sure that the database is ready

## Create user and databases for canvas

```
psql -U postgres
create user canvas password 'canvas';
CREATE DATABASE canvas_production ENCODING 'UTF8' OWNER canvas;
CREATE DATABASE canvas_queue_production ENCODING 'UTF8' OWNER canvas;
GRANT ALL PRIVILEGES ON DATABASE canvas_production to canvas;
GRANT ALL PRIVILEGES ON DATABASE canvas_queue_production to canvas;
\q
```

## Install Ruby and Nginx with Passenger  

Follow [Nginx-with-Passenger install ](https://github.com/m-narayan/beacon/wiki/Install-Nginx-with-Passenger) install to configure the server

## Code installation

We need to put the Canvas code in the location where it will run from. 

    sysadmin@appserver:~$ sudo mkdir -p /var/canvas
    sysadmin@appserver:~$ sudo chown -R sysadmin /var/canvas
    sysadmin@appserver:~$ cd canvas
    sysadmin@appserver:~/canvas$ ls
    app     db   Gemfile  log     Rakefile  spec  tmp
    config  doc  lib      public  script    test  vendor
    sysadmin@appserver:~/canvas$ cp -av * /var/canvas
    sysadmin@appserver:~/canvas$ cd /var/canvas
    sysadmin@appserver:/var/canvas$ ls
    app     db   Gemfile  log     Rakefile  spec  tmp
    config  doc  lib      public  script    test  vendor
    sysadmin@appserver:/var/canvas$


## Bundler and Canvas dependencies

    sysadmin@appserver:/var/canvas$ sudo gem install bundler
    sysadmin@appserver:/var/canvas$ bundle install --path vendor/bundle --without=sqlite

## Canvas default configuration

    sysadmin@appserver:/var/canvas$ for config in amazon_s3 database \
      delayed_jobs domain file_store outgoing_mail security external_migration
    do cp config/$config.yml.example config/$config.yml; done

## Database configuration

  
        sysadmin@appserver:/var/canvas$ vi config/database.yml
  

Update this section to reflect your Postgres server's location and authentication credentials. 

## Outgoing mail configuration

   
    sysadmin@appserver:/var/canvas$ nano config/outgoing_mail.yml
    

Find the **production** section and configure it to match your SMTP provider's settings. Note that the *domain* and *outgoing_address* fields are not for SMTP, but are for Canvas. *domain* is required, and is the domain name that outgoing emails are expected to come from. *outgoing_address* is optional, and if provided, will show up as the address in the *From* field of emails Canvas sends.

## URL configuration

In many notification emails, and other events that aren't triggered by a web request, Canvas needs to know the URL that it is visible from. For now, these are all constructed based off a domain name. Please edit the **production** section of *config/domain.yml* to be the appropriate domain name for your Canvas installation. For the *domain* field, this will be the part between `http://` and the next `/`. Instructure uses *canvas.instructure.com*.

   
    sysadmin@appserver:/var/canvas$ nano config/domain.yml
    

## Database population

   
    sysadmin@appserver:/var/canvas$ RAILS_ENV=production bundle exec rake db:initial_setup
   

## File Generation

   
    sysadmin@appserver:/var/canvas$ bundle exec rake canvas:compile_assets
   

## Canvas ownership

### Making sure Canvas can't write to more things than it should

   
    sysadmin@appserver:~$ cd /var/canvas
    sysadmin@appserver:/var/canvas$ sudo adduser --disabled-password --gecos canvas canvasuser
    sysadmin@appserver:/var/canvas$ sudo mkdir -p log tmp/pids public/assets public/stylesheets/compiled
    sysadmin@appserver:/var/canvas$ sudo touch Gemfile.lock
    sysadmin@appserver:/var/canvas$ sudo chown -R canvasuser config/environment.rb log tmp public/assets \
                                      public/stylesheets/compiled Gemfile.lock config.ru

  

### Making sure other users can't read private Canvas files

There are a number of files in your configuration directory (`/var/canvas/config`) that contain passwords, encryption keys, and other private data that would compromise the security of your Canvas installation if it became public. These are the *.yml* files inside the *config* directory, and we want to make them readable only by the *canvasuser* user.

```
sysadmin@appserver:/var/canvas$ sudo chown canvasuser config/*.yml
sysadmin@appserver:/var/canvas$ sudo chmod 400 config/*.yml
```

# configure nginx and passenger for canvas
```
sysadmin@appserver:/var/canvas$ cd /opt/nginx/sites-available
sysadmin@appserver:/opt/nginx/sites-available$ sudo vi canvas 
```

add the following line to the file to redirect plain http request url to secure https url

```
server {
    listen      80;
    server_name lms.arrivu.corecloud.com;
    # rewrite     ^   https://$server_name$request_uri? permanent;
    return 301 https://lms.arrivu.corecloud.com$request_uri;
}
```

add the following lines to create the https site configurations

```
server  {
               listen 443;
               server_name lms.arrivu.corecloud.com;
               root /var/canvas/public;
                           charset utf-8;
                           include mime.types;
                           default_type application/octet-stream;
               access_log /var/log/nginx/canvas.access.log;
               error_log /var/log/nginx/canvas.error.log;
               passenger_enabled on;
               rails_env production;
               ssl on;
               ssl_certificate /opt/nginx/ssl/canvas_cert_nginx.crt;
               ssl_certificate_key /opt/nginx/ssl/canvas_cert_nginx.key;
               ssl_session_timeout  5m;
        }

```

# Configure ssl for nginx

create the server private key, you'll be asked for a passphrase: enter the passphrase as 'arrivu'

```
sysadmin@appserver:/opt/nginx/sites-available$ cd /opt/nginx/ssl
sysadmin@appserver:/opt/nginx/ssl$ sudo openssl genrsa -des3 -out canvas_cert_nginx.key 2048
```

Create the Certificate Signing Request (CSR):

```
sysadmin@appserver:/opt/nginx/ssl$ sudo openssl req -new -key canvas_cert_nginx.key -out canvas_cert_nginx.csr
```

This will as lot of questions and enter the server name as ** lms.arrivu.corecloud.com **

Remove the necessity of entering a passphrase for starting up nginx with SSL using the above private key:

```
sysadmin@appserver:/opt/nginx/ssl$ sudo cp canvas_cert_nginx.key canvas_cert_nginx.key.org
sysadmin@appserver:/opt/nginx/ssl$ sudo openssl rsa -in canvas_cert_nginx.key.org -out canvas_cert_nginx.key
```

Finally sign the certificate using the above private key and CSR:

```
sysadmin@appserver:/opt/nginx/ssl$ sudo openssl x509 -req -days 365 -in canvas_cert_nginx.csr -signkey canvas_cert_nginx.key -out canvas_cert_nginx.crt
```
Update Nginx configuration by including the newly signed certificate and private key:

```
sysadmin@appserver:/opt/nginx/ssl$ cd /opt/nginx/sites-available
sysadmin@appserver:/opt/nginx/sites-available$ sudo vi canvas
```
make sure the following lines are in the /opt/nginx/sites-available/canvas file ssl server configuration

```
ssl_certificate /opt/nginx/ssl/canvas_cert_nginx.crt;
ssl_certificate_key /opt/nginx/ssl/canvas_cert_nginx.key;
```
 
# Enable the canvas site in Nginx

```
sysadmin@appserver:/opt/nginx$ sudo ln -s /opt/nginx/sites-available/canvas /opt/nginx/sites-enables/canvas
```

# reload the nginx server configuration

```
sysadmin@appserver:/opt/nginx$ sudo service nginx reload 
```

## Cache configuration

### Redis config

```
sysadmin@appserver:/var/canvas$ sudo apt-add-repository ppa:chris-lea/redis-server
sysadmin@appserver:/var/canvas$ sudo apt-get install redis-server
sysadmin@appserver:/var/canvas$ cd /var/canvas/
sysadmin@appserver:/var/canvas$ cp config/cache_store.yml.example config/cache_store.yml
sysadmin@appserver:/var/canvas$ nano config/cache_store.yml
```

The file starts with all caching methods commented out. Uncomment the `cache_store: redis_store` line of the config file. 

```yaml
# if this file doesn't exist, memcache will be used if there are any
# servers configured in config/memcache.yml
production:
  cache_store: redis_store
  # if no servers are specified, we'll look in config/redis.yml
  # servers:
  # - localhost
  # database: 0
```
  database: 0

Then specify your redis instance information in `redis.yml`, by coping and editing [redis.yml.example](https://github.com/instructure/canvas-lms/blob/stable/config/redis.yml.example):

    sysadmin@appserver:/var/canvas$ cd /var/canvas/
    sysadmin@appserver:/var/canvas$ cp config/redis.yml.example config/redis.yml
    sysadmin@appserver:/var/canvas$ nano config/redis.yml

```yaml
production:
  servers:
  - localhost
```

In our example, redis is running on the same server as Canvas. That's not ideal in a production setup, since Rails and redis are both memory-hungry. Just change 'localhost' to the address of your redis instance server.

Canvas has the option of using a different redis instance for cache and for other data. The simplest option is to use the same redis instance for both. If you would like to split them up, keep the redis.yml config for data redis, but add another separate server list to cache_store.yml to specify which instance to use for caching.

QTIMigrationTool
================

The QTIMigrationTool needs to be installed for copying content from one Canvas course to another to succeed. Instructions are at https://github.com/instructure/QTIMigrationTool/wiki. When Canvas is installed activate the plugin in Site Admin -> Plugins -> QTI Converter.

Automated jobs
========

Canvas has some automated jobs that need to run at occasional intervals, such as email reports, statistics gathering, and a few other things. Your Canvas installation will not function properly without support for automated jobs, so we'll need to set that up as well.

Canvas comes with a daemon process that will monitor and manage any automated jobs that need to happen. If your application root is */var/canvas*, this daemon process manager can be found at */var/canvas/script/canvas_init*. 

**You'll need to run these job daemons on at least one server.** Canvas supports running the background jobs on multiple servers for capacity/redundancy, as well.

Because Canvas has so many jobs to run, it is advisable to dedicate one of your app servers to be just a job server. You can do this by simply skipping the Apache steps on one of your app servers, and then only on that server follow these automated jobs setup instructions.

Installation
----

If you're on Debian/Ubuntu, you can install this daemon process very easily, first by making a symlink from */var/canvas/script/canvas_init* to */etc/init.d/canvas_init*, and then by configuring this script to run at valid runlevels (we'll be making an *upstart* script soon):

```
sysadmin@appserver:/var/canvas$ sudo ln -s /var/canvas/script/canvas_init /etc/init.d/canvas_init
sysadmin@appserver:/var/canvas$ sudo update-rc.d canvas_init defaults
sysadmin@appserver:/var/canvas$ sudo /etc/init.d/canvas_init start
```

Ready, set, go!
========

Restart nginx (`sudo /etc/init.d/nginx restart`), and point your browser to your new Canvas installation! Log in with the administrator credentials you set up during database configuration, and you should be ready to use Canvas.