# Udacity Nanodegree - Linux server configuration

The purpose of this project is to take a baseline installation of a Linux server and prepare it to host your web applications. You will secure your server from a number of attack vectors, install and configure a database server, and deploy one of your existing web applications onto it.

### Linux server Instance

1. Start a new Ubuntu Linux server instance on [Amazon Lightsail](https://lightsail.aws.amazon.com), create an AWS account if you do not have one.
2. Click `Create Instance` then choose `OS only` and `Ubuntu 16.04LTS`.
3. Choose the appropriate plan and click `Create`.


### SSH into server

 1.Download default private key from the Account page.
 
 2.Rename the key file as LightsailDefaultKey.rsa and move into `~/.ssh` folder.
 
 3. change the file permission by `chmod 600 ~/.ssh/lightsail_key.rsa`.
 
 4. Connect to the terminal `ssh -i ~/.ssh/LightsailDefaultKey.rsa ubuntu@34.217.93.62`.
 
### Update packages
Run `sudo apt-get update` and `sudo apt-get upgrade` to update all packages.

### Change the SSH port from 22 to 2200

1. Run `sudo nano /etc/ssh/sshd_config` to edit sshd_config file.
2. Change port from `22` to `2200` save and exit.
3. Restart SSH `sudo service ssh restart`.

### Configure Uncomplicated Firewall (UFW) 

1.Deny all incomings: `sudo ufw default deny incoming`.

2.Allow all outgoings: `sudo ufw default allow outgoing`.

3.Allow incoming connections for SSH (port 2200):`sudo ufw allow 2200/tcp`.

4.Allow incoming connections for TCP (port 80)`sudo ufw allow 80/tcp`.

5.Allow incoming connections for UDP (port 123)`sudo ufw allow 123/udp`.

6.Deny incoming request for port 22 : `sudo ufw deny 22`.

6.sudo ufw enable.

7.Check the configuration are correct :`sudo ufw status`.

### Create grader account and provide sudo access

1.`sudo adduser grader` creates a new user `grader`.

2. Run ` sudo touch /etc/sudoers.d/grader` to create a grader file.

3. Edit file grader `sudo nano /etc/sudoers.d/grader` and add line
grader ALL=(ALL:ALL) ALL save and exit.

4. Run `ssh-keygen` on local machine , provide the path `~/.ssh/grader` to save the key.

5. Run `cat ~/.ssh/grader.pub` to copy the contents of the file.

6. On virtual machine  create directory `mkdir .ssh`.

7. create file `touch .ssh/authorized_keys`.

8. Run `nano .ssh/authorized_keys` and paste the contents.

9. Change the directory and file permissions `chmod 700 .ssh ` and
`chmod 644 .ssh/authorized_keys`.

10.`nano /etc/ssh/sshd_config` and set `PasswordAuthentication` to `NO`.

11.Restart SSH - `sudo service ssh restart`.

12.From local machine SSH using grader access `ssh -i ~/.ssh/grader -p 2200 grader@34.217.93.62`.

### Configure the local timezone to UTC
Run `sudo dpkg-reconfigure tzdata ` and choose `US` and `Pacific-New`.

### Install and configure Apache to serve a Python mod_wsgi application

1.SSH as grader, Install Apache2 `sudo apt-get install apache2`.

2.Run `sudo apt-get install libapache2-mod-wsgi python-dev`

3.Enable mod_wsgi with `sudo a2enmod wsgi`

4.Start the web server with `sudo service apache2 start`

### Install git
SSH as grader
Run sudo apt-get install git

### Install and configure PostgreSQL

1.SSH as grader-Run `sudo apt-get install postgresql`.

2. check `/etc/postgresql/9.3/main/pg_hba.conf` to not to allow remote connections.

3. Connect to default postgres `sudo su - postgres`.

4. Run `psql` to connect to PostgreSQL.

5. Create database user with name catalog `CREATE ROLE catalog WITH PASSWORD 'password';`.

6. Allow database user 'catalog' to create database `ALTER USER catalog CREATEDB`.

7. Create database `CREATE DATABASE catalog WITH OWNER catalog;`.

8. conmnect to database 'catalog' `\c catalog`.

9.Restrict access `REVOKE ALL ON SCHEMA public FROM public`;

10.Grant access to user 'catalog' `GRANT ALL ON SCHEMA public TO catalog;`.

11. run `\q` and `exit` to exit postgresql.

### Deploy the Item Catalog project

1. SSH as grader - Create directory `mkdir /var/www/catalog'

2. cd in `cd /var/www/catalog` clone the [item catalog project](https://github.com/priyatrishul/Udacity-Item-Catalog)
`sudo git clone https://github.com/priyatrishul/Udacity-Item-Catalog catalog`.

3.Change owner `sudo chown -R grader:grader catalog/`.

4. create wsgi file `sudo touch /var/www/catalog/catalog.wsgi ` add following contents to the file
``` 
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```
5.Rename 'catalog.py' to '__init__.py'  `mv application.py __init__.py`.

6.Install virtual environment `sudo pip install virtualenv`.

7. change directory `/var/www/catalog/catalog/`.

8. create virtual environment ` sudo virtualenv venv`.

9. Activate the virtual environment `source venv/bin/activate`.

10. Change permissions `sudo chmod -R 777 venv`.

11. Install the required dependencies
```
   $ sudo pip install sqlalchemy
   $ sudo pip install flask
   $ sudo pip install httplib2
   $ sudo pip install requests
   $ sudo pip install oauth2client
   $ sudo apt-get install libpq-dev
   $ sudo pip install psycopg2
```
12.create file `sudo nano /etc/apache2/sites-available/catalog.conf` add following lines
```
<VirtualHost *:80>
    ServerName 34.217.93.62
  ServerAlias http://ec2-34-217-93-62.us-west-2.compute.amazonaws.com/
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
    	Order allow,deny
  	  Allow from all
    </Directory>
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
  	  Order allow,deny
  	  Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
13.edit file `sudo nano /etc/apache2/mods-enabled/wsgi.conf` and add

`#WSGIPythonPath directory|directory-1:directory-2:...
WSGIPythonPath /var/www/catalog/catalog/venv/lib/python2.7/site-packages`

14.Enable virtual environment `sudo a2ensite catalog`.

15.Reload Apache `sudo service apache2 reload`.

16.edit file  __init__.py `suso nano __initi.py`.

Change line `app.run(host='0.0.0.0', port=8000)` to `app.run() `.

17.edit files  __init__.py, db_setup.py, catalogdata.py and change line
`engine = create_engine("sqlite:///catalog.db")` to
`engine = create_engine('postgresql://catalog:password@localhost/catalog')`.

18.edit  __init__.py to provide full path of 'client_secret.json' to `/var/www/catalog/catlog/clients_secret.json`.

19.cd in  `cd /var/www/catalog/catalog` 
    Run `sudo python db_setup.py`
    Run `sudo python catalogdata.py`
    
20.http://ec2-34-217-93-62.us-west-2.compute.amazonaws.com/catalog/ or http://34.217.93.62/catalog/ to access.





