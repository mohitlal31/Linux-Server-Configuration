# Linux Server Configuration

A project to set up a web and database server on the cloud hosting
service [Amazon Lightsail](https://aws.amazon.com/lightsail/).
A baseline Ubuntu instance was set up on Amazon Lightsail
to host the [item-catalog web application](https://github.com/mohitlal31/Essential-Contacts).
[Apache](https://httpd.apache.org/) was used
to set up the web server and [Postgresql](https://www.postgresql.org/) for the database
server on the ubuntu virtual host.

Below, I have listed all the steps I undertook to configure this server.

### 1. SSH from local machine to the Lightsail virtual server instance

  - Download the private key file from Lighsail to the .ssh folder in your local
    machine.
  - From your shell terminal, run
    </br>`ssh -i ".ssh/<private_key_file>" ubuntu@<ip_address> -v -p 22`

### 2. Install Updates

  - Update the list of available upgrade packages
    `sudo apt-get update`
  - Install the updates for the packages
    `sudo apt-get upgrade`

### 3. Change the default SSH port 22 to non-standard port 2200

  - Add a custom firewall rule in the lightsail networking tab to
    allow connections on Port 2200</br>
    **Application: Custom, Protocol: TCP, Port: 2200**
  - Edit the server config file `sudo nano /etc/ssh/sshd_config`
    Add the line `Port 2200` and save(Ctrl-O, Enter, Ctrl-X)
  - Restart SSH `sudo service ssh restart`

  - Add a custom firewall rule in the lightsail networking tab to
    allow HTTP connections on Port 80</br>
    **Application: HTTP, Protocol: TCP, Port: 80**

  - Configure the Uncomplicated Firewall (UFW) to only allow incoming connections
    for SSH (port 2200), HTTP (port 80), and NTP (port 123)
    ```
      sudo ufw default deny incoming
      sudo ufw default allow outgoing
      sudo ufw allow 2200/tcp
      sudo ufw allow ntp
      sudo ufw allow www
      sudo ufw show added
      sudo ufw enable
      sudo ufw status
    ```

  Now, you should be able to SSH into the lightsail instance from port 2200
  with </br>`ssh -i ".ssh/<private_key_file>" ubuntu@<ip_address> -v -p 2200`</br>
  If you connect successfully, delete the firewall rule in the lightsail console
  that allows connections on port 22

### 4. Create a new user: grader

  - Create user grader `sudo adduser grader`
  - Edit the sudoers file `sudo nano /etc/sudoers.d/grader`
    Add line `grader ALL=(ALL) NOPASSWD:ALL`
    **Note:** I made a typing mistake here. I had to delete the linux instance and start all over again

### 5. Create SSH key-pair for grader login

  - Create a new key-pair on Lightsail account page named grader
  - Download it and place it in the .ssh directory of local computer.
  - On the local terminal (not the server),
  run `ssh-keygen -y -f .ssh/grader.pem`
  - Copy the output and paste it in the linux instance as below
    ```
      su - grader
      mkdir .ssh
      nano .ssh/authorized_keys
    ```
  - Paste the output and save
  
### 6. Prepare to deploy the project

  - Install apache and mod_wsgi
  ```
      sudo apt-get install apache2
      sudo apt-get install libapache2-mod-wsgi
  ```

  - Copy Paste the lightsail IP Address into your browser. The default
  apache homepage should open up.

  - Install Postgresql
  </br>`sudo apt-get install postgresql`

  - Create a database 'catalog' with requisite permissions
  ```
      sudo su - postgresql
      psql
      CREATE DATABASE CATALOG
      CREATE USER catalog WITH PASSWORD 'catalog'
      GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog
  ```

  - Create a project folder and clone the project repository
  ```
      sudo cd /var/www
      sudo mkdir catalog
      sudo git clone https://github.com/mohitlal31/Essential-Contact catalog
      cd catalog
  ```

  - Rename file essential_services.py present inside the repo</br>
  `sudo mv essential_services.py __init__.py`

  - change SQLAlchemy engine in database.setup.py to handle postgresql databases
  </br>From `engine = create_engine('sqlite:///itemcatalog.db')`
  </br>To `create_engine('postgresql://catalog:catalog@localhost/catalog')`

  - Setup virtual Environment
  ```
    sudo apt-get install python-pip
    sudo pip install virtualenv
  ```
   Inside /var/www/catalog/catalog
  ```
    sudo virtualenv venv
    source venv/bin/activate
  ```

  - Install all python project dependencies
  </br>`pip install -r requirements.txt`

### 7. Enable the catalog app to run instead of the default apache HTML page

  - Create a catalog.conf file
  </br>`sudo nano /etc/apache2/sites-available/catalog.conf`

  Paste the following code
  ```
    <VirtualHost *:80>
      ServerName 13.234.78.91.xip.io
      ServerAdmin admin@13.234.78.91
      WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
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

  - Create the wsgi file
  ```
    cd /var/www/catalog
    sudo nano catalog.wsgi
  ```

  - Paste the following code into the wsgi file
  ```
     #!/usr/bin/python
     import sys
     import logging
     logging.basicConfig(filename='/var/www/catalog/python_logs.logs', stream=sys.stderr)
     sys.path.insert(0,"/var/www/catalog/")

     from catalog import app as application
     application.secret_key = 'super_secret_key'
  ```

  - Enable the catalog conf file
  </br>`sudo a2ensite catalog`

  - Restart the apache service
  </br>`sudo service apache2 restart`

### 8. Setup the database tables and run the application
  
  ```
  cd /var/www/catalog/catalog
  python database_setup.py
  python populate_database_with_users.py
  ```

  - You're All good to go. Go to your browser and enter the lightsail IP address.
  The app should run successfully

### 9. Google Sign In Modifications

  - Add the lightsail IP address to the javascript origins in the google developers
  console. Append xip.io to the IP address. (ex. http://1.1.1.1.xip.io)
  - Add the same (ex. http://1.1.1.1.xip.io) to the redirect URIs

### Credits

  - [xip.io](http://xip.io/)
  - [How To Deploy a Flask Application on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
  - [Flask Docs](http://flask.pocoo.org/docs/1.0/deploying/mod_wsgi/)
