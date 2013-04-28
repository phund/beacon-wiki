## Installing Postgres
Follow [Postgresql ](https://github.com/m-narayan/beacon/wiki/Postgresql-Config) install to make sure that the database is ready

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

Follow [Ruby AppSrv ](https://github.com/m-narayan/beacon/wiki/Ruby-AppSrv-config) install to configure the server

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

## Nginx configuration

```
sysadmin@appserver:/var/canvas$ cd /opt/nginx
sysadmin@appserver:/opt/nginx$ sudo mkdir ssl
sysadmin@appserver:/opt/nginx$ sudo mkdir sites-available
sysadmin@appserver:/opt/nginx$ sudo mkdir sites-enabled

```
### passenger.conf file 

```
sysadmin@appserver:/opt/nginx$ cd /opt/nginx/conf
sysadmin@appserver:/opt/nginx/conf$ sudo vi passenger.conf

```


## Cache configuration