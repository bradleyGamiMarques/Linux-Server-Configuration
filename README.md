# Linux Configuration Server Project

Table of Contents
===

* Getting Started
* Configuring OAuth
* Creating a user with sudo access
* Enable SSH authentication
* Configure the server to serve our Python application
* We need PostgreSQL
* Install Git and Clone our project
* Configure the App
* Configure the firewall
* Server Details


Getting Started
===
This section outlines the steps that I took to configure my Amazon Lightsail Ubuntu instance.

1. Logged into my AWS account

2. Clicked the create instance button
    * Selected the Linux/Unix blueprints option
    * Selected OS Only
    * Selected Ubuntu 16.04 LTS
    * Selected the $5 month instance plan.
    * Named my instance Catalog-App-Ubuntu-Server
    * Clicked the Create button.

3. Log into the server and update your package list `sudo apt-get update`
4. Upgrade those packages `sudo apt-get upgrade`
5. Configure the timezone to UTC `sudo dpkg-reconfigure tzdata` Select None of the above > UTC


Configuring OAuth
===
This section outlines the steps that I took to configure OAuth for the Catalog App.

1. Navigate to Google Cloud Platform
2. In the Hamburger Menu navigate to API & Services > Credentials
3. Edit the Authorized JavaScript origins to include your server's public IP address in the format `http://{{PublicIPAddress}}` Also include the host name. This can be found by using www.hcidata.info
4. In the Authorized redirect URIs section include the host name/gconnect. Or whatever you called the function that handles the google sign in.

Creating a user with sudo access
===
1. Log into the Amazon Lightsail instance.
2. `sudo adduser grader`
3. Fill out the prompts for creating the user.
4. `sudo touch /etc/sudoers.d/grader`
5. `sudo vim /etc/sudoers.d/grader`
6. Press 'esc' then 'i' for insert, type grader ALL=(ALL:ALL) ALL, save and quit 'esc' ":wq"

Enable SSH authentication
===
On the local machine follow these steps
===
1. Open a terminal window. I use Git Bash
2. Navigate the user's home directory.
3. Create a .ssh directory `mkdir .ssh`
4. run `ssh-keygen`
5. At the prompt type .ssh/{{name of key}} This can be anything
6. Hit enter.
7. Navigate to the .ssh directory and run `cat {{name of key.pub}}. Copy this key, you will need it for later. 

On the Amazon Lightsail instance
===

1. Switch to the account that you want to login using ssh: `sudo su - {{user}}`
2. Create a directory for storing keys: `mkdir .ssh`
3. Set the file permissions for this directory: `chmod 700 .ssh` Only the owner has permission to Read, Write, Execute
4. Create an authorized keys files in the .ssh directory. `touch .ssh/authorized_keys`
5. Set the file permissions for this file: `chmod 600 .ssh/authorized_keys` Only the owner can read or write to the file.
6. Use your favorite text editor and paste the public key for your key pair into the file and save the changes.
    * Example: nano .ssh/authorized_keys

Back on the local machine
===
At this point you should be able to login as the user you created using the following command from your local terminal: ssh {{user}}@{{ip_address}} -p 2200 -i ~/.ssh/{{name of key}}

Configuring the server to serve our Python application
===
1. Install Apache `sudo apt-get install apache2`
2. Install mod_wsgi `sudo apt-get install python-setuptools libapache2-mod-wsgi`
3. Restart Apache `sudo service apache2 restart`

We need PostgreSQL
===

1. Install PostgreSQL using `sudo apt-get install postgresql`
2. Login in as the postgres user `sudo su - postgres`
3. Access the postgreSQL shell `psql`
4. Create a new database to store our catalog 

`postgres=# CREATE DATABASE catalog;`


`postgres=# CREATE USER catalog;`

5. Set the password for the new user we just created.

`postgres=# ALTER ROLE catalog WITH PASSWORD 'password';`

6. Give the user we created permission to access the database.

`postgres=# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;`

7. Quit postgreSQL and exit from the user

`postgres=# \q`

`exit`

Install git and clone my Catalog App Project
===
1. Install Git `sudo apt-get install git`
2. Move into the /var/www directory `cd /var/www`
3. `git clone https://github.com/bradleyGamiMarques/ItemCatalog.git`
4. Directory structure looks like `/var/www/ItemCatalog/catalog`
5. Removed the forum, tournament, README.MD, and Vagrantfile. We don't need them.

Configure the app
===
1. Rename `application.py` to  `__init__.py` using `sudo mv application.py __init__.py`
2. This app originally used SQLite. We need to configure the files that used SQLite to use postgreSQL. Do this by: `sudo nano __init__.py` Edit the line `create_engine('sqlite:///catalog.db)` to read `create_engine('postgresql://catalog:password@localhost/catalog')`

3. Do the same for `database_setup.py`
4. Install pip `sudo apt-get install python-pip`
5. Install the virtualenv `sudo pip install python-virtualenv`
6. Create the virtual environment `sudo virtualenv {{name of virtual environment}}`
7. Start the virtual environment `source {{name of virtual environment}}/bin/activate`
8. Install Flask and other packages (requests, sqlalchemy, oauth2client, httplib2)
9. Create and configure the virtual host `sudo nano /etc/apache2/sites-available/catalog.conf`

```
<VirtualHost *:80>
  ServerName {{PublicIPAddress}}
  ServerAdmin {{Email}}
  ServerAlias {{Host Name}}
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

10. Enable the virtual host you created `sudo a2ensite catalog`
11. Create the wsgi file that you referenced in the virtual host file `sudo nano /var/www/catalog/catalog.wsgi`

```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from catalog import app as application
application.secret_key = 'Add your secret key'
```

12. Restart the apache server `sudo service apache2 restart`

Configure the firewall
===
1. Open `/etc/ssh/sshd_config` using your favorite text editor
2. Change port 22 to 2200
3. Change PermitRootLogin to no
4. Change PasswordAuthentication to no.
5. Configure ufw Uncomplicated Firewall
   ```
   sudo ufw allow 2200/tcp
   sudo ufw allow 80/tcp
   sudo ufw allow 123/udp
   sudo ufw enable
   ```

6. Manage your Amazon Lightsail instance. In the console click on the three dots > Manage > Networking and add the following rules: HTTP TCP 80, Custom UDP 123, Custom TCP 2200


Resources
===
[Disable Root Login](https://mediatemple.net/community/products/dv/204643810/how-do-i-disable-ssh-login-for-the-root-user)

[Markdown Reference](https://en.support.wordpress.com/markdown-quick-reference/)

[Secure PostgreSQL](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)

[Deploy A Flask Application on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)


Server Details
===
This section outlines the server details.
* IP Address: 18.221.125.170
* RAM: 512 MB
* CPU: 1vCPU
* SSD: 20 GB SSD  
