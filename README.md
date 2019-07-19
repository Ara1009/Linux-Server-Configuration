# Linux Server Configuration 

* SSH port: 2200
* IP address: 18.224.63.158
* Grader password: grader

## Creating a Ubuntu instance on AWS
   [Sign up with AWS]
   "Create a Instance"
   "OS Only"
   "Ubuntu 16.04 LTS"
   Select instance plan

## Accessing the Server
   Download your private key from manage section
   Save file as "LightSailKey.rsa"
   Move file to local /.ssh/
   Set the file permission to owner only : `chmod 600 ~/.ssh/LightSailKey.rsa`
   Log in to the server like: `ssh -i ~/.ssh/LightSailKey.rsa ubuntu@18.224.63.158`

## Updating the current packages
   `sudo apt-get update` to update the packages
   `sudo apt-get upgrade` to install the latest versions

##  Changing the SSH port from 22 to 2200
   `sudo nano /etc/ssh/sshd_config` 
   Change the port number from "22" to "2200" (Should be near the top)
   Save and exit the file
   type `sudo service ssh restart` to restart SSH

## Readying the firewall
   Start by Checking the firewall status by typing: `sudo ufw status`
   Change default firewall to deny all incoming: `sudo ufw default deny incoming`
   Change default firewall to allow all outgoing: `sudo ufw default allow outgoing`
   Allow incoming UDP packets on port 123 to allow NTP: `sudo ufw allow 123/udp`
   Allow incoming TCP on port 2200 to allow SSH: `sudo ufw allow 2200/tcp`
   Allow incoming TCP on port 80 to allow www: `sudo ufw allow www`
   Close port 22 for security: `sudo ufw deny 22`
   Enable the firewall: `sudo ufw enable`
   Check the firewall status: `sudo ufw status`
   Be sure too add "TCP port:2200" and "UDP port:123" to the firewall on AWS under networking and Delete "port:22"
   ssh in using the new port 2200: `ssh -i ~/.ssh/LightSailKey.rsa ubuntu@18.224.63.158 -p 2200`

## Creating a new user called "grader" and giving it sudo access
   Create a new user called "grader" by typing:`sudo adduser grader`
   `sudo nano /etc/sudoers`
   Create a file named grader under: `sudo touch /etc/sudoers.d/grader`
   Edit it to give "grade" sudo access: `sudo nano /etc/sudoers.d/grader`, add code `grader ALL=(ALL:ALL) ALL`. 

## Logging in with the SSH key pairs
   Create an SSH key pair for "grader" by typing `ssh-keygen` on your local machine. when it prompts for a location save it in `~/.ssh` 
   to move your public key the server go to your local machine, open the public key by typing `cat ~/.ssh/[name of file].pub`
   On your server type `mkdir .ssh` to create a .ssh folder
   create a "authorized_keys" file by typing `touch .ssh/authorized_keys`
   then type `nano .ssh/authorized_keys` to edit the file and copy the public key to this file
   type `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys` to restrict permissions
   Restart SSH by typing: `sudo service ssh restart`
   log in as grader by typing: `ssh -i ~/.ssh/graderkey -p 2200 grader@18.224.63.158`
   for security reason to disable password type `sudo nano /etc/ssh/sshd_config`
   Change the line `PasswordAuthentication yes` to a "no"
   Restart SSH by typing: `sudo service ssh restart`

## Changing the local timezone to UTC
   Type `sudo dpkg-reconfigure tzdata`
   choose "none of the above" and then choose "UTC"

## Disable Root Login Via SSH
   Open the /etc/ssh/sshd_config file
   Change 'PermitRootLogin yes' to 'PermitRootLogin no'
   Add line 'AllowUsers grader'

## Installing and configuring Apache
   Install "apache2" by typing: `sudo apt-get install apache2`
   Go to http://18.224.63.158/, apache will show a default page if it is working

## Installing and configuring Python mod_wsgi
   Install "mod_wsgi" package by typing: `sudo apt-get install libapache2-mod-wsgi python-dev`
   Then enable it: `sudo a2enmod wsgi`
   Restart the "Apache2" service by typing: `sudo service apache2 restart`

## Installing PostgreSQL
   type `sudo apt-get install postgresql` to install PostgreSQL

## Creating a PostgreSQL user called "catalog"
   Access the PostgreSQL default user "postgres" by typing: `sudo su - postgres`
   Connect to PostgreSQL by typing: `psql`
   Create a user names "catalog" with LOGIN role: `# CREATE ROLE catalog WITH PASSWORD 'randompass';`
   Allow the user to create database tables: `# ALTER USER catalog CREATEDB;`
   Create a database: `# CREATE DATABASE catalog WITH OWNER catalog;`
   Connect to the database **catalog**: `# \c catalog`
   Revoke all the rights: `# REVOKE ALL ON SCHEMA public FROM public;`
   Grant access to the "catalog" user: `# GRANT ALL ON SCHEMA public TO catalog;`
   Exit psql by typing: `\q`

## Creating a Linux user names "catalog" and a database
   Create a new user by typing: `sudo adduser catalog`
   Give the "catalog" user sudo access through: `sudo visudo`
   Copy the folowing into it: `catalog ALL=(ALL:ALL) ALL` under line `root ALL=(ALL:ALL) ALL`
   Log in as the "catalog" user: `sudo su - catalog`
   Create a database called "catalog": `createdb catalog`
   Exit by `^d`

## Installing git and cloning the catalog project via Github
   type `sudo apt-get install git`
   Clone the project: `sudo git clone "github link"`
   Change the the owner of the folder: `sudo chown -R ubuntu:ubuntu FlaskApp/`
   CD to `/var/www/FlaskAppFolder/FlaskApp`
   Change `catalog.py` to `__init__.py` by typing: `mv catalog.py __init__.py`
   Change line `app.run(host="0.0.0.0", port=5000, debug=False)` to `app.run()` in the `__init__.py` file

## Readying the server for the Flask project
   Install pip by typing: `sudo apt-get install python-pip`
   Installing the packages:
   Install packages from requirements.txt file

## Setting up and enabling the virtual host
   Create a file by typing: `sudo touch /etc/apache2/sites-available/FlaskApp.conf`
   Add this to the file:
`
<VirtualHost *:80>
                ServerName 18.224.63.158
                ServerAdmin grader@18.224.63.158
                WSGIDaemonProcess application  python-path=/home/grader/.local/lib/python2.7/site-packages
                WSGIProcessGroup application
                WSGIScriptAlias / /var/www/FlaskAppFolder/catalog.wsgi
                <Directory /var/www/FlaskAppFolder/FlaskApp/>
                        Order allow,deny
                        Allow from all
                        Options -Indexes
                </Directory>
                Alias /static /var/www/FlaskAppFolder/FlaskApp/static
                <Directory /var/www/FlaskAppFolder/FlaskApp/static/>
                        Order allow,deny
                        Allow from all
                        Options -Indexes
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
   </VirtualHost>
`
   type `$ sudo a2ensite FlaskApp` to start the virtual host (It might already be enabled)
   Restart the "apache2" service: `sudo service apache2 reload`

## Configuring the .wsgi file
   Create this file by typing: `sudo touch /var/www/FlaskAppFolder/catalog.wsgi`
   Add the folloing to the file:

`import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/FlaskAppFolder/FlaskApp")
from __init__ import app as application
application.secret_key = 'super secret key'`

   Restart the "apache2" service: `sudo service apache2 reload`

## Editing the database paths
   Replace lines in `__init__.py`, `lotsofcategories.py`, and `database_setup.py` files with `engine = create_engine('postgresql://catalog:catalog@localhost/catalog')`

## Disabling the default Apache2 page
   type: `a2dissite 000-default.conf`
   Restart the "apache2" service: `sudo service apache2 reload`

## Setting up the database
   While in /var/www/Item-Catalog/catalog type: `sudo python database_setup.py` to create the database
   and ` sudo python lotsofcategories.py` to populate the database
   Restart the "apache2" service: `sudo service apache2 reload`
   go to  http://18.224.63.158/ and you should see the website up and running