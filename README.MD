# Linux Server Configuration

This is the final project for Udacity's [Full Stack Web Developer Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004). 

In this project, the [Item Catalog Web Application](https://github.com/jskrishnamurthy/ItemCatalog) created earlier in the same program will be hosted by a Ubuntu Linux server on an [Amazon Lightsail](https://lightsail.aws.amazon.com). A series of instructions will be presented below. You can visit http://54.148.203.211.xip.io to access the deployed website. 

## Get a server

### Step 1: Start a new Ubuntu Linux server instance on Amazon Lightsail 

- Login into [Amazon Lightsail](https://lightsail.aws.amazon.com/ls/webapp/home/resources) using an Amazon Web Services account.
- Once you are login into the site, click `Create instance`. 
- Choose `Linux/Unix` platform, `OS Only` and  `Ubuntu 18.04 LTS`.
- Choose a instance plan.
- Name our instance.
- Click the `Create` button to create the instance.
- Wait for the instance status to turn to `Running` 

### Step 2: SSH into the server

- Download private key from the **SSH keys** section in the **Account** section on Amazon Lightsail. The file name will be something like _LightsailDefaultPrivateKey-us-west-2.pem_
- Create a new file named **lightsail_key.rsa** under ~/.ssh folder on your local machine
- Copy and paste content from downloaded private key file to **lightsail_key.rsa**
- Open Terminal or Git Bash on your local machine
- Set file permission as owner only : `$ chmod 600 ~/.ssh/lightsail_key.rsa`
- SSH into the instance: `ssh -i ~/.ssh/lightsail_key.rsa ubuntu@54.148.203.211`

## Secure the server

### Step 3: Update and upgrade installed packages

- Run `sudo apt-get update` to update packages
- Run `sudo apt-get upgrade` to install newest versions of packages
- Set for future updates: `sudo apt-get dist-upgrade`



### Step 4: Change the SSH port from 22 to 2200

- Edit the `/etc/ssh/sshd_config` file: `sudo nano /etc/ssh/sshd_config`.
- Change the port number from `22` to `2200`.
- Save and exit the file.
- Restart SSH: `sudo service ssh restart`.

### Step 5: Configure the Uncomplicated Firewall (UFW)

- Configure the default firewall for Ubuntu to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).
  ```
  sudo ufw status                  # The UFW should be inactive.
  sudo ufw default deny incoming   # Deny any incoming traffic.
  sudo ufw default allow outgoing  # Enable outgoing traffic.
  sudo ufw allow 2200/tcp          # Allow incoming tcp packets on port 2200.
  sudo ufw allow www               # Allow HTTP traffic in.
  sudo ufw allow 123/udp           # Allow incoming udp packets on port 123.
  sudo ufw deny 22                 # close port 22.
  sudo ufw enable                  # Enable firewall
  sudo ufw status                  # Check current firewall status
  ```
- Exit the SSH connection: `exit`.
- Click on the `Manage` option of the Amazon Lightsail Instance, 
then the `Networking` tab, and delete default SSH port 22 and Allow ports **80(TCP), 123(UDP), and 2200(TCP)**
- Open a new terminal/git bash from your local machine, run: `ssh -i ~/.ssh/lightsail_key.rsa ubuntu@54.148.203.211 -p 2200` to SSH into the server.


## Give `grader` access


### Step 6: Create a new user account named `grader`

- While logged in as `ubuntu`, add user: `sudo adduser grader`. 
- Enter a password (twice) and fill out information for this new user.


### Step 7: Give `grader` the permission to sudo

- Edits the sudoers file: `sudo visudo`.
- Search for the line that looks like this:
  ```
  root    ALL=(ALL:ALL) ALL
  ```

- Below this line, add a new line to give sudo privileges to `grader` user.
  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  ```
- Save and exit the file.
- Verify that `grader` has sudo permissions by running `su - grader`

### Step 8: Create an SSH key pair for `grader` using the `ssh-keygen` tool

- On the local machine Terminal/Git bash:
  - Run `ssh-keygen`
  - Save the key as `grader_key` in the local directory `~/.ssh`
  - Enter in a passphrase twice. Two files will be generated (  `~/.ssh/grader_key` and `~/.ssh/grader_key.pub`)
  - copy the contents of the grader_key.pub file
  - Log in to the grader's virtual machine
- On the grader's virtual machine, run below commands:
  - `mkdir /home/grader/.ssh`
  - `touch /home/grader/.ssh/authorized_keys`
  - `nano /home/grader/.ssh/authorized_keys`
  - Paste the public key contents copied from local machine grader_key.pub to this authorized_keys file and save
  - Give the permissions: `chmod 700 /home/grader/.ssh` and `chmod 644 /home/grader/.ssh/authorized_keys`
  - Check if `PasswordAuthentication` is set to `no` in `/etc/ssh/sshd_config` file
  - Restart SSH: `sudo service ssh restart`
- On the local machine,you are now able to SSH into virtual machine using: `ssh -i ~/.ssh/grader_key grader@54.148.203.211 -p 2200`.




## Prepare to deploy the project

### Step 9: Configure the local timezone to UTC

- While logged in as `grader`, Run `$ sudo dpkg-reconfigure tzdata`
- Choose **None of the above** => UTC to set timezone to UTC

### Step 10: Install and configure Apache to serve a Python mod_wsgi application

- While logged in as `grader`, install Apache: `sudo apt-get install apache2`.
- Go to http://54.148.203.211/, if Apache is working correctly, a **Apache2 Ubuntu Default Page** will show up
- Install the **mod_wsgi** package: `$ sudo apt-get install libapache2-mod-wsgi python-dev`
- Enable **mod_wsgi**: `$ sudo a2enmod wsgi`
- Restart **Apache**: `$ sudo service apache2 restart`
- Check if Python is installed: `$ python`

### Step 11: Install and configure PostgreSQL

- While logged in as `grader`, install PostgreSQL:
 `sudo apt-get install postgresql`.
- Check to make sure remote connections to PostgreSQL is not allowed in   `/etc/postgresql/10/main/pg_hba.conf` file:
 ```
   # Database administrative login by Unix domain socket
   local   all             postgres                                peer
   # TYPE  DATABASE        USER            ADDRESS                 METHOD
   # "local" is for Unix domain socket connections only
   local   all             all                                     peer
   # IPv4 local connections:
   host    all             all             127.0.0.1/32            md5
   # IPv6 local connections:
   host    all             all             ::1/128                 md5
   ```

- Switch to the `postgres` user: `sudo su - postgres`.
- Open PostgreSQL interactive terminal with `psql`.
- Create the `catalog` user with a password and give them the ability to create databases:
  ```
  postgres=# CREATE ROLE catalog WITH LOGIN PASSWORD 'catalog';
  postgres=# ALTER ROLE catalog CREATEDB;
  postgre=#  ALTER ROLE "catalog" WITH LOGIN;
  ```

- List the existing roles: `\du`. The output should be like this:
  ```
                                     List of roles
   Role name |                         Attributes                         | Member of 
  -----------+------------------------------------------------------------+-----------
   catalog   | Create DB                                                  | {}
   postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
  ```

- Exit psql: `\q`.
- Switch back to the `grader` user: `exit`.
- Create a new Linux user called `catalog`: `sudo adduser catalog`. Enter password and fill out information.
- Give to `catalog` user the permission to sudo. Run: `sudo visudo`.
- Search for the lines that looks like this:
  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  ```

- Below this line, add a new line to give sudo privileges to `catalog` user.
  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  catalog  ALL=(ALL:ALL) ALL
  ```

- Save and exit the file.
- Verify that `catalog` has sudo permissions by running `su - catalog`

- While logged in as `catalog`, create a database: `createdb catalog`.
- Run `psql` and then run `\l` to see that the new database has been created. The output should be like this:
  ```
                                    List of databases
     Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
  -----------+----------+----------+-------------+-------------+-----------------------
   catalog   | catalog  | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
   postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
   template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
             |          |          |             |             | postgres=CTc/postgres
   template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
             |          |          |             |             | postgres=CTc/postgres
  (4 rows)
  ```
- Exit psql: `\q`.
- Switch back to the `grader` user: `exit`.

### Step 12: Install git and clone Item catalog application from github

- While logged in as `grader`, install `git`: `sudo apt-get install git`.
- While logged in as `grader`, create `/var/www/catalog/` directory.
- Change to that directory and clone the catalog project:
`sudo git clone https://github.com/jskrishnamurthy/ItemCatalog.git catalog`.
- From the `/var/www` directory, change the ownership of the `catalog` directory to `grader` using: `sudo chown -R grader:grader catalog/`.
- Change to the `/var/www/catalog/catalog` directory.
- Rename the `application.py` file to `__init__.py` using: `mv application.py __init__.py`.

- In `__init__.py`, replace :
  ```
  # app.run(host="0.0.0.0", port=8000)
  app.run()
  ```
  ```
  # engine = create_engine('sqlite:///catalog.db',connect_args={'check_same_thread': False})
  engine = create_engine('postgresql://catalog:password@localhost/catalog')
  ```

- In `models.py`:
   ```
   # engine = create_engine('sqlite:///catalog.db')
   engine = create_engine('postgresql://catalog:password@localhost/catalog')
   ```
 - In `dbStarterItems.py`:
   ```
   # engine = create_engine('sqlite:///catalog.db')
   engine = create_engine('postgresql://catalog:password@localhost/catalog')
   ```
  - Run: `python dbStarterItems.py` to populate database

### Step 13: Authenticate login through Google

- Go to [Google Cloud Plateform](https://console.cloud.google.com/).
- Click `APIs & services` on left menu.
- Click `Credentials`.
- Create an OAuth Client ID (under the Credentials tab), and add http://54.148.203.211.xip.io as authorized JavaScript 
origins and authorized redirect URI.
- Download the corresponding JSON file and copy the file contents.
- Open `/var/www/catalog/catalog/client_secret.json` and paste the contents into the this file.
- In `__init__.py`, make following changes:
  ```
  # CLIENT_ID = json.loads(open('client_secrets.json','r').read())['web']['client_id']
  APP_PATH = '/var/www/catalog/catalog/'
  CLIENT_ID = json.loads(open(APP_PATH + 'client_secrets.json','r').read())['web']['client_id']
  ```
  ```
  # oauth_flow = flow_from_clientsecrets('client_secrets.json', scope='')
  oauth_flow = flow_from_clientsecrets(APP_PATH + 'client_secrets.json', scope='')
  ```
- Replace the client ID from google api in `templates/login.html` file.
### Step 14: Install the virtual environment and dependencies

While logged in as `grader`, 
- From /var/www/catalog/catalog directory install `pip`: 
    ```
    sudo apt-get install python3-pip.
    ```
- Install the virtual environment: 
    ```
    sudo apt-get install python-virtualenv
    ```
- Create the virtual environment: 
    ```
    sudo virtualenv -p python3 venv3.
    ```
- Change the ownership to grader with: 
    ```
    sudo chown -R grader:grader venv3/.
    ```
- Install the following dependencies:
  ```
  pip install httplib2
  pip install requests
  pip install --upgrade oauth2client
  pip install sqlalchemy
  pip install flask
  sudo apt-get install libpq-dev
  pip install psycopg2
  ```

- Run `python3 __init__.py` and you should see:
  ```
  * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
  ```


### Step 15: Set up and enable a virtual host

- Create `/etc/apache2/sites-available/catalog.conf` and add the 
following lines to configure the virtual host:

  ```
    <VirtualHost *:80>
        ServerName 54.148.203.211
        ServerAdmin ubuntu@54.148.203.211
        WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/catalog/venv3/lib/python3.6/site-packa$
       WSGIScriptAlias / /var/www/catalog/catalog.wsgi
        WSGIProcessGroup catalog
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

- Enable virtual host: `sudo a2ensite catalog`. 
- Reload Apache: `sudo service apache2 reload`.

### Step 16: Set up the Flask application

- Create `/var/www/catalog/catalog.wsgi` file add the following lines:

  ```
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0, "/var/www/catalog/")
    sys.path.insert(1, "/var/www/catalog/catalog/")
    from catalog import app as application
    application.secret_key = "supersecretkey"
  ```

- Restart Apache: `sudo service apache2 restart`.

### Step 17: Disable the default Apache site

- Disable the default Apache site: `sudo a2dissite 000-default.conf`. 
The following prompt will be returned:

  ```
  Site 000-default disabled.
  To activate the new configuration, you need to run:
    service apache2 reload
  ```

- Reload Apache: `sudo service apache2 reload`.

### Step 18: Launch the Web Application

- Open your browser to http://54.148.203.211.xip.io

### Step 19: To disable remote login of the root user

- While logged in as `grader`,``` sudo nano  /etc/ssh/sshd_config ```
- uncomment the directive `PermitRootLogin` and set its value to `no`
- Save and Exit
- To Verify, ``` grep -i PermitRootLogin /etc/ssh/sshd_config ```
- The command should return something like this:
```
PermitRootLogin no
# the setting of "PermitRootLogin without-password".
```

## Useful commands

 - To get log messages from Apache server: `sudo tail /var/log/apache2/error.log`.

## References

- DigitalOcean [How To Deploy a Flask Application on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
- Troubleshoot mod_wsgi [Link](https://github.com/jungleBadger/-nanodegree-linux-server-troubleshoot/tree/master/python3%2Bvenv%2Bwsgi)
- [4 Ways to Disable Root Account in Linux](https://www.tecmint.com/disable-root-login-in-linux/)
