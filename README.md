# linuxServerConfig

Instructions on how to setup an Apache server to serve my catalog app. This also gives instructions for the configuration of a UFW Firewall, changing SSH port, updating the OAuth files and more.

## Server Details

IP address: `50.112.88.245` 

SSH port: `2200`

URL: `http://ec2-50-112-88-245.us-west-2.compute.amazonaws.com`

## Add user "grader" 

To add a new user named 'grader' type `sudo adduser grader`

<h1>Give grader user sudo access</h1>

Add user to sudo group

`usermod -aG sudo grader`

## Update all currently installed packages

`apt-get update` - to update the package indexes

`apt-get upgrade` - to actually upgrade the installed packages

If you get the `system restart required` message type `reboot` to reboot the machine

## Set-up SSH keys for the grader user

As the root user type:

`mkdir /home/grader/.ssh`
`chown grader:grader /home/grader/.ssh`
`chmod 700 /home/grader/.ssh`
`cp /root/.ssh/authorized_keys /home/grader/.ssh/`
`chown grader:grader /home/grader/.ssh/authorized_keys`
`chmod 644 /home/grader/.ssh/authorized_keys`

You are now able to login as `grader` by typing `ssh -i ~/.ssh/udacity_key.rsa grader@50.112.88.245`

## Fix sudo error
I got the response `sudo: unable to resolve host ip-10-20-62-0` when using sudo while root and grader user. 

To solve this problem I edited the `etc/hosts` file so the first line read 
`127.0.0.1 localhost ip-10-20-62-0`.

Found on Ubuntu Help Forums [here](http://askubuntu.com/questions/59458/error-message-when-i-run-sudo-unable-to-resolve-host-none)

## Disable root login

Use sudo access to open the file `/etc/ssh/sshd_config`:

Change `PermitRootLogin without-password to `PermitRootLogin no`.

Also make sure `PasswordAuthentication no` is not commented out.

Type `sudo service ssh restart` for the changes to be applied.

From now all do all changes as the `grader` user, use sudo when necessary.

## Change timezone to UTC

Change timezone to UTC by typing `sudo timedatectl set-timezone UTC`.

## Change SSH port from 22 to 2200

Use sudo to open the file `/etc/ssh/sshd_config` again and change the `Port 22` so that
it now reads `Port 2200`.

Restart SSH service:

`sudo service ssh restart`

From now on type the following command to login to the server:

`ssh -i ~/.ssh/udacity_key.rsa grader@50.112.88.245 -p 2200`

## Setup the Uncomplicated Firewall

Block all incoming connections on ports:

`sudo ufw default deny incoming`

Allow outgoing connection on all ports

`sudo ufw default allow outgoing`

Allow incoming connection for SSH on port 2200

`sudo ufw allow 2200/tcp`

Allow incoming connections for HTTP port 80:

`sudo ufw allow www`

Allow incoming connection for NTP on port 123:

`sudo ufw allow ntp`

Check to make sure the rules have been added successfully before enabling:

`sudo ufw show added`

Enable the firewall:

`sudo ufw enable`

Check firewall status:

`sudo ufw status`

<h1>Install Apache</h1>

Install Apache:

`sudo apt-get install apache2`

Install mod-wsgi:

`sudo apt-get install libapache2-mod-wsgi`

## Install SQLite

Install sqlite3 by typing:

`sudo apt-get install sqlite3 libsqlite3-dev`

## Install Packages

Type these commands to install appropriate packages:

`sudo apt-get install python-psycopg2 python-flask
`sudo apt-get install python-sqlalchemy python-pip
`sudo pip install oauth2client
`sudo pip install requests
`sudo pip install httplib2`

You could also set up a virtual environment then install these packages. Instructions found [here](http://docs.python-guide.org/en/latest/dev/virtualenvs/)

## Install Git

`sudo apt-get install git`

## Clone Project 3 repository

Navigate to the var/www directory:

`cd /var/www`

Make new 'coffeeshops' directory:

`sudo mkdir coffeeshops`

Change owner of directory:

`sudo chown www-data:www-data coffeeshops`

Clone repository and set user with path:

`sudo -u www-data git clone https://github.com/dunton/coffeeshops coffeeshops`

## Update catalog.wsgi file

This is what the catalog.wsgi file should look like in this instance:

```#!/usr/bin/python
`import sys`
import logging
import os.path
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, 'var/www/coffeeshops/catalog/')
from database_setup import create_database
from database_populator import populate_database
from views import app as application

application.config['DATABASE_URL'] = 'sqlite:////var/www/coffeeshops/catalog/coffeeshopmenu.db'

# Create database and populate it, if not already done so.
if os.path.isfile('var/www/coffeeshops/catalog/coffeeshopmenu.db') is False:
    create_database(application.config['DATABASE_URL'])
    populate_database()
```
    
## Update Facebook Oauth

We need to make sure our Facebook Authorization system works. First, add `app_id` and `app_secret` to the appropiate place in the `fb_client_secrets.json` file. 

Then go to the Facebook Developer's console, under the Coffeeshops App settings go to the Facebook Login product. Scroll down to the "Valid OAuth redirect URIs section and add `http://ec2-50-112-88-245.us-west-2.compute.amazonaws.com/fbconnect`, `http://ec2-50-112-88-245.us-west-2.compute.amazonaws.com/coffeeshops` and `http://50.112.88.245`.

## Configure Apache2 to Serve the App

Navigate to the virtual host configuration file at `/etc/apache2/sites-available`.

Either edit the 000-default.conf file or create your own with these contents:
```
<VirtualHost *:80>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        #ServerName www.example.com

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/coffeeshops

        WSGIDaemonProcess coffeeshops threads=5
        WSGIScriptAlias / /var/www/coffeeshops/catalog/catalog.wsgi

        <Directory coffeeshops>
                WSGIProcessGroup catalog
                WSGIApplicationGroup %{GLOBAL}
                Require all granted
        </Directory>

        # Setup static directory
        Alias /static /var/www/coffeeshops/catalog/static

        # Serve from static directory
        <Directory /var/www/coffeeshops/catalog/static/>
                Require all granted
        </Directory>

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        # For most configuration files from conf-available/, which are
        # enabled or disabled at a global level, it is possible to
        # include a line for only one particular virtual host. For example the
        # following line enables the CGI configuration for this host only
        # after it has been globally disabled with "a2disconf".
        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

If you chose to create your own file instead of editing the 000-default.conf file, you must enable the new file. First type `sudo a2dissite 000-default.conf` to disable the default.conf file. Then type `sudo a2ensite <your-conf-name>.conf` to enable your new .conf file. Then to make these changes go live type `sudo service apache reload`.

The app should now serve at `http://http://ec2-50-112-88-245.us-west-2.compute.amazonaws.com/` as well as `50.112.88.245`.

## Helpful Resources

[https://www.digitalocean.com/community/tutorials/how-to-set-up-apache-virtual-hosts-on-ubuntu-14-04-lts]

[http://www.datasciencebytes.com/bytes/2015/02/24/running-a-flask-app-on-aws-ec2/]

[http://flask.pocoo.org/docs/0.11/deploying/mod_wsgi/]

[http://flask.pocoo.org/docs/0.11/quickstart/]

[http://flask.pocoo.org/docs/0.11/patterns/sqlalchemy/]

[https://docs.python.org/3/library/os.path.html]

[http://alex.nisnevich.com/blog/2014/10/01/setting_up_flask_on_ec2.html]

[https://github.com/SteveWooding/fullstack-nanodegree-linux-server-config]
