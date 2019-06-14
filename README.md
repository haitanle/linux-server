# Linux Configuration and Deployment

## Access
IP Address: 18.220.19.249 port 2200
ssh grader@18.220.19.249 -p 2200
(ssh private key is included in submission's comment)

URL: http://18.220.19.249/main

## Start Configuration

### Add user - grader 
* As ubuntu, add user `grader` with command: `sudo adduser grader` (password: grader)

### Allow sudo commands to user grader
 * Access the `/etc/sudoers.d` with `sudo ls /etc/sudoers.d`
 * Create the file `grader`, by copying an existing file and renamed it.
 ```
 sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader
 ```
 * Use `sudo nano /etc/sudoers.d/grader` and change contents of the file to the following:
```
grader ALL=(ALL) NOPASSWD:ALL
```

## Set-up SSH keys for user grader
### Generating Key Pairs:
 * On our local machine, using `ssh-keygen` in the terminal we generate a keygen named `udacity` with no Passphrase.
 * On our VM use `mkdir /home/grader/.ssh` then generate `touch /home/grader/.ssh/authorized_keys`.
 * Back on our local machine, view the contents of our `.pub` file with `cat ~/.ssh/udacity.pub` and copy it.
 * On the VM use `sudo nano /home/grader/.ssh/authorized_keys` to edit the file and paste the contents.


 * Force Key Based Authentication by editing the file `sudo nano /etc/ssh/sshd_config`.

Successfully login as the `grader` user using the command:
```
ssh -i ~/.ssh/udacity grader@18.220.19.249 -p 2200
```

### File Permissions
> To ensure other users don't gain access to our account
 * `sudo chmod 700 .ssh`
 * `sudo chmod 644 .ssh/authorized_keys`

# Packages
## Add Package
  * Install package finger `sudo apt-get install finger`

> Check info about user grader using finger with - `finger grader`

## Update all currently installed packages
 * Show list of packages to be updated - `sudo apt-get update`
 * Upgrade the installed packages - `sudo apt-get upgrade`


## Change SSH Port
* On the networking tab of our ubuntu instance on the Amazon lightsail page.
  * Add a custom TCP protocol on port 2200.
  * Add the HTTP (port 80) and NTP (port 123).
* On our VM, change port from 22 to 2200 by editing `sudo nano /etc/ssh/sshd_config`.
* Restart the SSH service
 * Log in via port 2200
 ```
ssh -i ~/.ssh/udacity@18.220.19.249 -p 2200
```

# Uncomplicated Firewall - UFW
### *Configuring our Firewall*
* Check status on our firewall - `sudo ufw status`
* Block all connections coming in - `sudo ufw default deny incoming`
* Allow all connections outgoing from our server - `sudo ufw default allow outgoing`
* Allow all SSH connections - `sudo ufw allow ssh`
* According to the project rubric, allow the following - 
 ```
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/udp
 ```
* Since we aren't using port 22 for anything we can deny it with `sudo ufw deny 22` (we can also edit our Networking tab on our aws lightsail console).
* Enable our Firewall (*only once we have configured our firewall correctly*) - `sudo ufw enable`
* Check out status to see implementation of firewall using - `sudo ufw status`

# Install Software and Configure our vm to serve our python application

### Installing Apache
> Apache is our web server, its job is to process HTTP requests.
  * Install with `sudo apt-get install apache2`
  * Test by opening our public IP on our browser `http://18.220.19.249`, it should display the Default Apache2 Ubuntu **It works** page.

_We can `cd` to the `var/www/html` directory, to find the html file that is currently being served onto the site_ 

### Installing mod_wsgi
> This is a free tool for serving Python applications from Apache.
> We will also be installing a helper package called python-setuptools.
  * Install with `sudo apt-get install python-setuptools libapache2-mod-wsgi`.

* Restart our Apache server for **mod_wsgi** to load: `sudo service apache2 restart`.

### Install git
> git is our version control software.
  * Install with `sudo apt-get install git`

### Setup our flask project
> Move into the www directory , create a folder to store a clone of our catalog repo from github.
  * Move using `cd` to `/var/www/html` and create the directory `sudo mkdir CatalogApp`.
  * In the `CatalogApp` directory clone our project repo with
    ```
    sudo git clone git@github.com:haitanle/catalog-app.git CatalogApp
    ```

### Setting up our project to run on our Ubuntu server
* Create a catalog.wsgi file - `sudo nano flaskapp.wsgi` in the `/var/www/html/CatalogApp/catalog/` directory,
with the following contents:

 ```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/html/CatalogApp/catalog")

from __init__ import app as application
application.secret_key = 'super_secret_key'
 ```

* Rename project.py to __init__.py `mv application.py __init__.py`

### Installing a virtual environment, flask and other project dependencies
> Setting up a virtual environment will keep the application and its dependencies isolated from the main system. 
> Changes to it will not affect the cloud server's system configurations.

* Use pip to install virtualenv.
  ```
  sudo apt-get install python-pip
  sudo pip install virtualenv
  ```
*  Install - `sudo apt install python-psycopg2`

* `cd` into our `/var/www/html/CatalogApp/catalog` folder.
* Create an instance of the virtual environment and activate it
```
sudo virtualenv venv
source venv/bin/activate
```

* Install flask and other dependencies
```
sudo pip install Flask
sudo pip install bleach 
sudo pip install httplib2
sudo pip install requests
sudo pip install oauth2client 
sudo pip install sqlalchemy
```
* Leave the virtual env with `deactivate`.


### Configure / Enable a new virtual host
* Create and edit with the following - `sudo nano /etc/apache2/sites-available/FlaskApp.conf`
```
  <VirtualHost *:80>
      ServerName 18.220.19.249
      ServerAdmin admin@18.220.19.249
      WSGIScriptAlias / /var/www/html/CatalogApp/catalog/flaskapp.wsgi
      <Directory /var/www/html/CatalogApp/catalog/>
          Order allow,deny
          Allow from all
      </Directory>
      Alias /static /var/www/html/CatalogApp/catalog/static
      <Directory /var/www/html/CatalogApp/catalog/static/>
          Order allow,deny
          Allow from all
      </Directory>
      ErrorLog ${APACHE_LOG_DIR}/error.log
      LogLevel warn
      CustomLog ${APACHE_LOG_DIR}/access.log combined
  </VirtualHost>
```
* Restart the apache service with `service apache2 reload` or `service apache2 restart`
* Enable our new virtual host with `sudo a2ensite FlaskApp`

### Install and Configure our PostgreSQL database
* Install - `sudo apt-get install postgresql postgresql-contrib`
* Log in as user - `postgres` (a user created during installation) and create the following tables.
  * Log in with `sudo su - postgres`
  * Run `psql` in terminal
* Create a user named `catalog` with password `catalog`
```sql
CREATE USER catalog WITH PASSWORD 'catalog';
```
* Give this user the ability to create databases
```sql
ALTER USER catalog CREATEDB;
```
* Then create a database managed by that user
```sql
CREATE DATABASE catalog OWNER catalog;
```
* Connect to our newly created database - `\c catalog`
* once connected to the database, lock down the permissions to only let `catalog` create tables:
```sql
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
```
* Log out of the `psql` terminal with `\q`, and then use `exit` to logout/ switch back to our `grader` user.

## Edits made to our repository on our server to config our server

> Changes made to our project repo from the cloned localhost version.

* Refactor the following files - `__init__.py`,`database_setup.py`,`data.py` to contain our new database connection.
```python
engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
```
* Include the full file path on any code using the `.open` method.
```python
app_id = json.loads(open('/var/www/html/CatalogApp/catalog/client_secrets.json', 'r').read())[
        'web']['app_id']
```

============================================

# Tips / Notes
### User - ubuntu Password
> Default ubuntu user is created upon instance.
 * To change password - `sudo passwrd <username>`
## Moving folders, renaming and removing files
 * To move folders in linux use `mv source target` and `mv filename1.txt filename2.txt`.
 * To remove `sudo rm -f -r filename`

## SSH Reboot and Restart
* Use `sudo service ssh restart` or `/etc/init.d/ssh restart` to restart ssh after changes.
* Use `sudo reboot` to disconnect and restart VM.

## Viewing Apache2 Error Logs during Apache server and .wsgi set-up
* `sudo journalctl | tail`
* Look into `/var/log/` directories, in my case `/var/log/apache2/error.log`

## OAuth2 Google Login

Google OAuth2 login is not working due to xip.io redirect blocked by Google. 