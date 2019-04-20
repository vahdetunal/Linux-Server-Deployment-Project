# Configuration Steps

## Connecting to the server
- Create an SSH keypair from https://lightsail.aws.amazon.com/ls/webapp/account/profile
- Create an OS only ubuntu instance from https://lightsail.aws.amazon.com/ls/webapp/home/instances
- Download the SSH key
- Connect to the server from Git Bash using ssh -i <PATH_TO_KEY> ubuntu@35.156.147.193

## Updating packages
- Update package list using `sudo apt-get update`
- Update all packages to their most recent versions using sudo apt-get upgrade

  ```
  Reboot the system using sudo reboot
  sudo apt-get dist-upgrade
  reboot again using sudo reboot
  ```
- Choose Install maintainers version for any prompt.

## Changing the SSH port
- Edit ssh configuration file `sudo nano /etc/ssh/sshd_config`
- Edit the port number from 22 to 2200 and uncomment the line
- Restart sshd `sudo service sshd restart`

- All connections from now are done using `ssh -i <PATH_TO_KEY> ubuntu@35.156.147.193 -p 2200`

## Firewall Configuration
- Check firewall status sudo ufw status it should be:
  `Status: inactive`

- Add firewall Rules

  ```
  sudo ufw default deny incoming
  sudo ufw default allow outgoing
  sudo ufw allow 2200/tcp
  sudo ufw allow www
  sudo ufw allow 123/udp
  ```
  
- Enable the firewall using `sudo ufw enable`

- Restart the system

- Allow the same ports on lightsail firewall from
https://lightsail.aws.amazon.com/ls/webapp/eu-central-1/instances/vunalaws/networking

- SSH based login is already enforced by Lightsail. Therefore, additional
  configurations for it are not necessary.


## Create Grader Account

- To create the account run `sudo adduser grader --disabled-password`

- To provide SSH access to the new user following guidelines on:
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/managing-users.html

- Switch to grader user: `sudo su grader `
- Go to grader home folder and create the file for ssh key. Set its permissions correctly.
[Source](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/managing-users.html)
  ```
  cd ~
  mkdir .ssh
  chmod 700 .ssh
  touch .ssh/authorized_keys
  chmod 600 .ssh/authorized_keys
  ```

- Generate a key pair for the grader account on local machine using `ssh-keygen`

- Edit the authorized keys file:
`nano .ssh/authorized_keys`

- Copy the public key from id_rsa.pub file on your local machine to the authorized_keys file and save it.

- Now grader account can be accessed by ssh using `ssh -i id_rsa grader@35.156.147.193 -p 2200`

- Return to our original user terminal using `exit`

- Copy the ubuntu users file to use as a template
  `sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader`

- Edit the newly created file to add grader as a sudoer
  `sudo visudo -f /etc/sudoers.d/grader`
  
- Change username from ubuntu to grader.

## Prevent root login via SSH
- run `sudo nano /etc/ssh/sshd_config`

- Find the line `#PermitRootLogin prohibit-password`

- Change the value from phohibit-password to no and uncomment the line. Save changes.

  `sudo service sshd restart`

- Remove root users authorized keys. [source](https://ubuntuforums.org/showthread.php?t=1703944)
`sudo rm /root/.ssh/authorized_keys`

- Change time zone to UTC
` sudo dpkg-reconfigure tzdata `
Select None of the Above and select UTC.

## Install Required Software

- Install apache web server `sudo apt-get install apache2`

- Install mod_wsgi `sudo apt-get install libapache2-mod-wsgi-py3`

- Install PostgreSQL `sudo apt-get install postgresql`

- Install libpq-dev. Required for psycopg2. `sudo apt-get install libpq-dev`

- By default, PostgreSQL does not allow remote connections. This can be verified
by running ```sudo nano /etc/postgresql/10/main/pg_hba.conf``` and checking file
contents. [source](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)

- Switch to postgres user ```sudo su - postgres```

- Create a new database user named catalog
  `createuser --interactive --pwprompt`
  Shall the new role be a superuser? No.
  Shall the new role be allowed to create databases? No.
  Shall the new role be allowed to create more new roles? No.

- Create the database for the application
```createdb -O catalog historicalsites```
[Source](https://www.a2hosting.com/kb/developer-corner/postgresql/managing-postgresql-databases-and-users-from-the-command-line#Creating-PostgreSQL-users)

- Return to the original user shell.

- Create the folder for the application/
  ```
  cd /var/www
  sudo mkdir catalog
  ```

- Take ownership off the application folder `sudo chown -R ubuntu /var/www/catalog`

- Clone the repository `git clone <ADRESS_TO_MY_REPO>`

- Install pip for python3 `sudo apt-get install python3-pip`

- Install virtual environment for python `sudo pip3 install python3-venv`

- Move into cloned directory `cd turkey-historical-sites`

- Initialize the virtual environment `virtualenv catalogenv`

- Activate the virtual environment `source venv/bin/activate`

- Install dependancies
  ```
  pip3 install sqlalchemy
  pip3 install psycopg
  pip3 install flask_sqlalchemy
  pip3 install oauth2client
  pip3 install requests
  pip3 install Flask
  ```
## Deploy the application
This part required a lot of trial and error. The sources that has helped me to solve major problems are:
-- https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
-- https://stackoverflow.com/questions/23842713/using-python-3-in-virtualenv
-- https://stackoverflow.com/questions/42050982/flask-wsgi-no-module-named-flask

- Configure the virtual host ```sudo nano /etc/apache2/sites-available/catalog.conf```

  ```
  <VirtualHost *:80>
                  ServerName 35.156.147.193
                  ServerAdmin vahdetunal58@gmail.com
                  WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/catalog:/var/www/catalog/catalog/venv/lib/python3.6/site-packages
                  WSGIProcessGroup catalog
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

- Enable the virtual host `sudo a2ensite catalog`

- Write the wsgi file.
  ```
  #!/usr/bin/python3
  import sys
  import logging

  logging.basicConfig(stream=sys.stderr)
  sys.path.insert(0,"/var/www/catalog/")

  from catalog import app as application
  f = open("/var/www/catalog/secret_key.txt", "r")
  application.secret_key = f.read()
  f.close()
  ```

- Rename project.py to __init__.py

- Add full paths for client_secrets and secret_key files. Virtual environment
  has trouble finding them otherwise.

- Generate new secret key and json secrets since the old files were published on
  github

- Restart Apache `sudo service apache2 restart`

- The application is now deployed and can be accessed over web.

## Accessing The server
- **IP ADRESS:** 35.156.147.193
- **SSH PORT:** 2200
- **URL:** http://vunalaws.crabdance.com
 
 - Login works only when the website is accessed from the url. Google Oauth2 does not accept IP adresses as valid redirect adresses.