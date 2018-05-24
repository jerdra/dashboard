The dashboard webapp provides an overview of the studies supported by the kimel
lab. The app supports manual QC of MRI sessions as well as display of various
QC metrics generated by our nightly pipelines.

## Table of Contents
1. [Technologies](#technologies)
2. [Dashboard Structure](#dashboard-structure)
3. [Database Schema](#database-schema)
4. [Setting Up a Development Environment](#setting-up-a-development-environment)  
   * [Setting Up the Dashboard Virtual Environment](#setting-up-the-dashboard-virtual-environment)  
   * [Configuring Postgres for the Dashboard](#configuring-postgres-for-the-dashboard)  
   * [Setting Up the Database Schema](#setting-up-the-database-schema)  
     * [If you're in TIGRLab](#if-youre-in-tigrlab)  
     * [If you're not](#if-youre-not)  
   * [Setting Up OAuth for the Dashboard](#setting-up-oauth-for-the-dashboard)  
   * [Setting Up the Datman Config Files](#setting-up-the-datman-config-files)  
     * [Copy and Modify](#copy-and-modify)
     * [Build from Scratch](#build-from-scratch)  
   * [Setting Up Your Shell Environment](#setting-up-your-shell-environment)  
     * [Creating an Environment Module](#creating-an-environment-module)
     * [Creating a Script to Source](#creating-a-script-to-source)
   * [Running the Dashboard](#running-the-dashboard)
5. [TIGRLab Dashboard](#tigrlab-dashboard)  
   * [Dashboard Server](#dashboard-server)  
   * [Database Server](#database-server)  
   * [Adding a New Study](#adding-a-new-study)
6. [TIGRLab Ansible Configuration](#tigrlab-ansible-configuration)  
   * [Dashboard Ansible Role](#dashboard-ansible-role)  
     * [dashboard_ini_template.j2](#dashboardinitemplatej2)
     * [dashboard.ini.j2](#dashboardinij2)
   * [Postgres Ansible Role](#postgres-ansible-role)  
     * [Postgresql Configuration](#postgresql-configuration)
     * [Backups](#backups)

## Technologies

The app is supported by a backend [postgreSQL](https://www.postgresql.org/)
database, hosted on the srv-postgres.camhres.ca virtual server. The front end is
programmed in python using the [Flask](http://flask.pocoo.org/) framework, with
[SQLAlchemy](https://www.sqlalchemy.org/). The web server itself is
[NGINX](https://www.nginx.com/) with
[uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/). Authentication is handled
with [OAuth](https://en.wikipedia.org/wiki/OAuth) and we currently support
github and gitlab for login.

## Dashboard Structure
[dashboard/models.py](dashboard/models.py) defines the _Object Relational Mapping (ORM)_ used by
SQLAlchemy to map from the relational database to the python objects used in
the code.

[dashboard/views.py](dashboard/views.py) defines the _entry points_ (i.e. valid URLs for the app).

HTML templates are in [dashboard/templates/](dashboard/templates). Templates with `_snip.html` are
embedded in other pages.

The web-app interacts with the filesystem (checklist.csv, blacklist.csv) and
updates made through the web-app are automatically propogated to the filesystem
before the update is made. If the filesystem cannot be updated the database
update fails. Alterations made directly to the filesystem are propagated
to the database by the nightly [datman](https://github.com/TIGRLab/datman)
scripts.

## Database Schema

This section under construction. - @DESm1th, May 23 2018

## Setting Up a Development Environment
Brace yourself, this is going to be a long one.

1. Fork [the dashboard](https://github.com/TIGRLab/dashboard.git)
2. Clone your new fork
3. Clone [datman](https://github.com/TIGRLab/datman)
4. Ensure that `pidentd`, `postgresql-9.5`, `postgresql-client-9.5` are
installed on your machine
5. [Set Up a Virtual Environment](#setting-up-the-dashboard-virtual-environment)
6. [Configure postgres for the dashboard](#configuring-postgres-for-the-dashboard)
7. [Set up the database schema](#setting-up-the-database-schema)
8. [Set up OAuth](#setting-up-oauth-for-the-dashboard)
9. [Set up your config files for datman](#setting-up-datman-config-files)
10. [Set up your shell environment](#setting-up-your-shell-environment)
11. [Run your new dashboard!](#running-the-dashboard)

### Setting Up the Dashboard Virtual Environment
Currently datman / the dashboard need python 2.7. The examples below use
virtualenv since that's my preference, but you can use any python package
for virtual environments (e.g. conda)

Create a new virtual environment
```
virtualenv --python=python2.7 <your path>/venv
```

Activate your new environment
```
source <your path>/venv/bin/activate
```

Install the required python packages.
```
pip install -r <path to your dashboard clone>/requirements.txt
```

There are a few things that may go wrong with installing these packages, all
centered on the cryptography package.

1. If you get a message about how pip failed to build or failed to install the
cryptography package make sure you have libssl-dev installed.
```
sudo apt install libssl-dev
```
2. Occasionally old versions of the cryptography package are made obsolete due
to discovered security vulnerabilities. If pip refuses to install the version
requested by requirements.txt it should be perfectly fine to just use the newest
version of cryptography instead.

If you find any other problems or cant get the cryptography package working
please let us know by creating an issue
[here](https://github.com/TIGRLab/dashboard/issues)

### Configuring Postgres for the Dashboard
There are three main changes needed to postgres' config files on your machine
to enable the dashboard to access the database. You will need sudo to make these
changes or need to ask an admin with sudo. For Ubuntu and PostgreSQL version 9.5
the config files will be found at `/etc/postgresql/9.5/main/`.

1. Update `postgresql.conf` to listen on your IP and localhost. Add the
following line:
```
listen_addresses = '<YOUR IP HERE>, localhost'
```
2. Update `pg_hba.conf` with the correct 'host' records. If this is a development
setup that only you will access adding the following will be fine:
```
host    dashboard   all   127.0.0.1/32         ident map=default
host    dashboard   all   <YOUR IP HERE>/32    ident map=default
```
If your setup will be used on an internet facing server **DO NOT** use the
ident method shown here. It's really easily fooled by anyone with ill intentions
and only safe to use for your own machine or a private, secure local network.
For more info on postgres authentication methods
[see here.](https://www.postgresql.org/docs/9.5/static/auth-methods.html)

3. Update `pg_ident.conf` to give your user account access to the web_user
role and to a role named after your account. If you're using the ident method
shown in step two then the following should work. `<your account name>` should be
replaced with whatever account name you use to log into your OS.
```
default         <your account name>     web_user
default         /(.*)                   \1
```
The second line will match your account name to a role of the same name. So for
example if my account name is `john_doe` and I add this in place of
`<your account name>` in the example config, I will be able to log into the
dashboard database as either `web_user` (matches first line) or `john_doe`
(matches second line).

Once you've made these changes you may need to restart postgres to get it to
recognize them. On Ubuntu 16.04 you can do this with
```
sudo systemctl restart postgresql
```

In addition, you may need to create your user account yourself with
```
sudo -u postgres createuser -s <your_username>
```

If this is a development set up you may also want to give your user account
Superuser access to the database for convenience. To do this you must connect to
the database as a user that already has superuser permissions (on most new
installations this would be the postgres user). Run the following in your
terminal
```
sudo su postgres
psql dashboard
```
Once connected to the database with psql you can grant superuser to your
account with
```
ALTER USER <your account name here> WITH SUPERUSER;
```

### Setting Up the Database Schema

#### If you're in TIGRLab
The easiest way to set up your test database is to just load in an old backup
of the real database (ask @DESm1th for this). Once you have the backup
you can set up your database by running the following as a user with database
creation privileges
([see this section for configuring these privileges](#configuring-postgres-for-the-dashboard)):
```
createdb dashboard
# If no host is specified psql will connect to postgres on your machine
psql dashboard -f $path_to_your_backup
```

#### If you're not
You'll have to build an empty database from scratch from the SQL files to be
added later. Sorry - @DESm1th, May 24 2018

### Setting Up OAuth for the Dashboard
Instructions for how to do this on **GitHub** are [found here](https://developer.github.com/apps/building-oauth-apps/creating-an-oauth-app/)
and instructions to do this with **GitLab** are
[here.](https://docs.gitlab.com/ee/integration/oauth_provider.html)

The name and description can be anything but the URL you use should be
`http://<your ip>:5000`. The Authorisation callback URL should be
`http://<your computer name or ip>:5000/callback/<provider>`. Replace
`<provider>` with either `github` or `gitlab` depending on which you are
configuring.

The base URL you set for the callback URL is what you should use when you
access your dashboard. If you use your computer name (or another name) in the
callback URL but attempt to access it with your IP in your browser (or vice
versa) authentication may fail if DNS is not properly configured.

So for example, if I use a URL of `http://1.1.1.1:5000` and a callback URL of
`http://borg-cube:5000/callback/github` without DNS configured accessing the
dashboard with `http://borg-cube:5000` should work but `http://1.1.1.1:5000`
may fail when github tries to contact the callback URL. To minimize problems
either set the URL and the base URL of the callback to be the same thing
or make sure that the base URL of your callback is exactly what you intend to
use when you access your app.

The __Client ID__ and __Client Secret__ that are generated by GitHub/GitLab
are needed for the shell environment as
[described here](#setting-up-your-shell-environment).

### Setting Up the Datman Config Files
The dashboard reads almost all of its information from the database, but uses
the config files to locate metadata that might need to be updated (e.g. scans.csv,
blacklist.csv, the study's README) and scans to be viewed in the papaya viewer.

#### Copy and Modify
If you're a TIGRLab member the easiest way to get setup is to copy `tigrlab_config.yaml` and any needed
study config files from `/archive/code/config`.

You should then update the following config entries:
- Update SERVER_LOG_DIR and change LOGSERVER to point to your machine if you
want to make use of datman's logging server
- In SystemSettings delete the existing entries and add a new block that points
to your projects. Here's an example template to use:
```
SystemSettings:
    YOUR_SYSTEM_NAME_HERE:
      hostname: 'yourdomainname'
      DATMAN_PROJECTSDIR: '/path/to/your/datman/archive'
      DATMAN_ASSETSDIR: '/path/to/your/datman/assets'
      CONFIG_DIR: '/path/to/your/config/files/folder'
```

#### Build from Scratch
Currently this page has some advice on [setting up configuration files](https://github.com/TIGRLab/admin/wiki/data-organization-and-config-files).

You may be able to get away with only setting up the 'SystemSettings' portion
of the site config file (tigrlab_config.yml in the wiki) rather than filling in
all sections. However, the code may complain about not having at least one
project settings file configured or missing the 'ExportSettings'.

### Setting Up Your Shell Environment
There are two main options for configuring your shell to run a development
instance of the dashboard.

#### Creating an Environment Module
If you use environment modules (TIGRLab does) you can create your own module
from the template provided in [dashboard.module.template](dashboard.module.template). Just
follow the comments and fill in your own passwords and other secrets. [See
here](https://wiki.scinet.utoronto.ca/wiki/index.php/Installing_your_own_modules)
for advice on setting up a module using environment modules.

Remember that in addition to loading your module you'll have to source your
virtual environment before running the dashboard with
```
source <path to your virtual env>/venv/bin/activate
```

#### Creating a Script to Source
If you dont use environment modules, or are more comfortable just sourcing a
script, you can build your environment set up script from the following template
```
# Provide OAuth details
# You can delete either the github or gitlab entries. You only need one configured
# Github
export OAUTH_SECRET_GITHUB=YOUR-SECRET-HERE
export OAUTH_CLIENT_GITHUB=YOUR-SECRET-HERE
# GitLab
export OAUTH_SECRET_GITLAB=YOUR-SECRET-HERE
export OAUTH_CLIENT_GITLAB=YOUR-SECRET-HERE

# Provide a secret key for Flask
# This can be whatever you want, but you should keep it secret and
# make it something not easily guessed since it's used to encrypt sessions
export FLASK_SECRET_KEY=YOUR-SECRET-HERE

# Provide Postgres info
# You can change the postgres user or database name here, just make sure
# everything is configured correctly in postgres
export POSTGRES_USER=web_user
export POSTGRES_DATABASE=dashboard
export POSTGRES_PASS=YOUR-SECRET-HERE
export POSTGRES_SRVR=YOUR-POSTGRES-SERVERS-IP-HERE

# Provide a redcap token to enable Scan Completed forms to be
# pulled in. This part is optional but you may have to fill in a fake value
# to get the dashboard to start :(
export REDCAP_TOKEN=YOUR-REDCAP-TOKEN-HERE

# Configure datman
# The dashboard requires datman. You can install it whereever you like but
# the site wide configuration file, site name, and the location of the code
# must be provided.
export DM_CONFIG=PATH-TO-YOUR-SITE-CONFIG-HERE
export DM_SYSTEM=YOUR-SYSTEM-NAME-HERE

# If you installed datman + the dashboard inside your environment you do not
# need to modify the python path or add them to your path and can omit this
# section. If you just cloned them somewhere, you do need it.
# Add datman to your paths
export PATH=<path to datman/datman folder>:$PATH
export PATH=<path to datman/bin folder>:$PATH
export PYTHONPATH=<path to datman/datman folder>:$PYTHONPATH
# Add dashboard scripts to your paths
export PATH=<path to dashboard here>:$PATH
export PYTHONPATH=<path to dashboard here>:$PYTHONPATH

# Source your virtual environment for convenience
source <path to your virtualenv>/venv/bin/activate
```

And then just source your script before running the dashboard
```
source <path to your script>
```
### Running the Dashboard
Once you've completed all the other steps to
[set up your development environment,](#setting-up-a-development-environment)
open a shell, load your shell environment by either loading your module and
sourcing your virtual environment or sourcing your setup script, and then
start up your server in one of two ways:

1. Use Flask's built in Werkzeug server with
 ```
 python dashboard/run.py
 ```  
This will give you debugging output when an error occurs and is more than
sufficient for a development instance. It should not be used for a production
server though.  

2. Use a temporary uWSGI instance with
```
bash dashboard/srv_uwsgi.sh
```
This gives you a setup closer to the TIGRLab production server, so you can toy
with uWSGI settings before trying them out on the real server.

## TIGRLab Dashboard
This section describes the TIGRLab's current production setup for our dashboard
and how to make modifications to it if needed. It's only relevant to TIGRLab
members who will be working with the production server :)

### Dashboard Server
**Server:** srv-dashboard.camhres.ca (172.26.216.66)

**NGINX** and **uWSGI** configuration are controlled by ansible. See
[Dashboard Ansible Role](#dashboard-ansible-role) for more info.

The uwsgi app runs as user **clevis** with the **web_user** role to access the
database. These are configured in `/etc/uwsgi/apps-available/dashboard.ini`

The codebase is expected to be located at `/archive/code/dashboard`. This means
that any updates or bug fixes to the dashboard need to be pulled into the
archive and uWSGI needs to be restarted with `systemctl restart uwsgi`
on srv-dashboard before they'll take effect.

**Important:** Some secret information is required for uWSGI. These passwords
should not be committed to github. The relevant passwords are in passpack, and as
mentioned in [Dashboard Ansible Role](#dashboard-ansible-role), separated into a
file that doesnt get committed to git. Please maintain this separation to avoid
any security issues.

### Database Server
**Server:** srv-postgres.camhres.ca (172.26.216.68)  
**Database name:** dashboard

Access requires that `postgresql-client` and `pidentd` packages are both
installed. This should already be the case on all lab workstations. New admin
users may need access to the postgres web_user role. See
[Postgres Ansible Role](#postgres-ansible-role) for more info.

Once authentication is correctly configured the database should be accessible with
```
psql -h srv-postgres -U web_user dashboard
```

In addition, three postgres _roles_ have been defined with access to the
dashboard database.

* **admin:** Manage databases, manage roles on all databases
* **dashboard:** Read, Write, Delete etc. on dashboard database
* **dashboard_read:** Read only on dashboard database

One special _user_ role has been defined. **web_user** is a member of the
**dashboard** role and is used by the webapp front end.

**Clevis** is defined as a superuser to enable backups.

### Adding a New Study
For now, to add a new study to the dashboard you must manually insert some
records into the database.

1. If the PI of the study is not already in the table 'people', they must be added
2. A record must be added to the table 'studies'
3. Any sites unique to this study must be added to the 'sites'
4. For each site in the study make a record in study_sites
5. Add any unique series tags to scantypes
6. For each scantype tag that may appear in this study's data, add a record to study_scantypes

## TIGRLab Ansible Configuration
For our lab, the dashboard's configuration can be found in the role 'dashboard'
and srv-postgres' configuration is in the role 'postgres'. This section is
only relevant for TIGRLab dashboard admins :)

### Dashboard Ansible Role
The most important thing this role does is add the uWSGI and NGINX configuration
for the dashboard to srv-dashboard. The NGINX configuration is just a copy
of `dashboard/templates/nginx_dashboard.conf.j2` while the uWSGI configuration
is generated from `dashboard/templates/dashboard_ini_template.j2` and
`dashboard/templates/dashboard.ini.j2`.

**NOTE**: If you make any configuration changes directly to srv-dashboard
without adding these changes to ansible your changes may be obliterated the
next time ansible is run. Also, it's just generally bad practice and makes it
harder to recreate a working server if anything catastrophic happens.
**Dont do this!**

##### dashboard_ini_template.j2
This file is the main source for uWSGI configuration. It holds all of the
non-sensitive settings for the server. New environment variables and changes
to how uWSGI will run should be added here.

##### dashboard.ini.j2
This file holds sensitive information that gets filled in to the
"dashboard_ini_template.j2" template. It is stored separately in a
directory that we never commit to github and linked into the templates folder.
Any new passwords or secrets should be added here and a line added to
"dashboard_ini_template.j2" where the new secret will be filled in.

### Postgres Ansible Role
The most important tasks this role performs are configuring postgresql for
the dashboard and configuring database backups.

##### PostgreSQL Configuration
As with the dashboard ansible role (and anything else managed by ansible)
configuration changes should be made in ansible and not directly to the server.

The postgres role has templates for `postgresql.conf`, `pg_ident.conf` and
`pg_hba.conf`. Any changes to postgres' configuration should be made to these
templates. To give a new user access to the web_user role their username
must be added to the `pg_admins` list in `postgres/vars/main.yml`. Only admins
might require this role.

##### Backups
Ansible configures a cron job (`/etc/cron.d/dashboard_bkup`) that will dump the
entire database nightly and store the result in `/mnt/backup` on srv-postgres.
We currently keep three weeks worth of backups. TIGRsrv is configured to copy
the backups and store them at `/mnt/backup/dashboard`, so we should have two
copies at all times.
