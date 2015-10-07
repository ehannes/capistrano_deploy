# Deployment
This readme explains how to deploy using the current setup of Capistrano
(with Phusion Passenger and NGINX as server, MYSQL as database, Git as version
control system and the repository to deploy on GitHub).

## Capistrano
Capistano allows you to automatically deploy a Rails application remotely from
your development machine. It will connect to your live server, fetch the
code from the chosen repository and deploy the application. When reading this,
you should be logged in on the development machine. Capistrano details are
set in the Capfile.

For deployment, Capistrano uses two other files in this setup:
``config/deploy.rb`` and ``config/deploy/production.rb``. ``deploy.rb`` contains
general configurations while ``production.rb`` is specific to the production
stage (Capistrano can handle several stages).

## Live server environment
The live server runs the Rails application via [Phusion
Passenger](https://www.phusionpassenger.com), for deployment details see the
[Phusion Passenger Library](https://www.phusionpassenger.com/library).  The live
server is configured in nginx [integration
mode](https://www.phusionpassenger.com/library/indepth/integration_modes.html),
which means that nginx is the web server and Passenger operates as an nginx
module. This means that you configure Passenger through nginx configuration
files and outputs from Passenger are passed on to nginx logs.

In Capistrano, a deployment directory on the live server is set. In this
directory, Capistrano sets up a folder structure. All releases are located in
``releases/`` and the current release that Passenger is running is
symlinked to ``current/``. Some things are shared across releases.
Those shared files (set in ``config/deploy.rb``) are found in ``shared/``. When
deploying for the first time, ``shared/config/secrets.yml`` and
``shared/config/database.yml`` must be created (they are not included in the
repository for obvious reasons).

The database (MySQL) is located on the same server and the connection details is
located in the ``shared/config/database.yml``.

To setup the database, go to current directory for the application and run
```
rake db:setup RAILS_ENV=production
```
which will create all necessary tables and seed the database (db/seeds.rb).
(Use db:reset to also drop the database first).

Ruby is installed with rbenv which handles multiple Ruby installations.

## Deployment steps

### On the development machine
#### Authentication
The application code in this setup is fetched from a repository on GitHub.  For
authentication, Capistrano is configured to use an SSH Agent to forward SSH keys
to deployment server. This means you should have your development machine
setup so that you can connect to GitHub via SSH with an SSH key. You will also
to have an SSH Agent running on the development machine you are logged in
on.

If the agent is not running then you need to connect to the agent with
```
eval `ssh-agent -s`
```
and verify the connection with
```
echo $SSH_AUTH_SOCK
```

Now add the SSH key with
```
ssh-add
```
and verify that the key is present with
```
ssh-add -l
```

#### Run capistrano
Simply run capistrano via bundle (handles dependencies for Ruby gems)
```
bundle exec cap production deploy
```


### On the live server

#### First time deployment
If this is a first time deployment, there are  several steps to cover on the
live server. Some of them can be found in the [live server
section](#-live-server-environment).

#### Updating the application
When updating the application at the live server the following steps are
necessary.

##### Update the database
If the update includes some changes to the database, Rails migrations takes care
to update the table structure. But the data inside the database could also need
some update. Maybe there are new columns that cannot be null or models that is
no longer valid due to certain attribute values. Be sure to run necessary
database queries to update the database.

##### Restart application
Passenger is already running an older version of the application on the live
server. The code is not affected by Capistrano but the database is. If you do
not restart the application, Passenger will continuing running an old version of
the application with an updated database which means trouble! Therefore, we must
tell Passenger to restart the application. Go to ``current/`` and run
```
passenger-config restart-app $(pwd)
```
Passenger will now run the now application that ``current/`` is linked to.

## Upgrading Ruby version
Sometimes it is necessary to upgrade the Ruby version. Both the development
machine and the live server need be upgraded. Ruby is installed with rbenv
and details about upgrade can be found at [rbenv
repository](https://github.com/sstephenson/rbenv). If the Ruby version is
upgraded, Passenger must also know of this.  The Ruby version is explicitly
stated in the site configuration file in ``/etc/nginx/sites-enabled/``.
