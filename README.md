# deploying-yukon-guide

# Setting up a Droplet

Create a Droplet in Digital Ocean.

Select Ubuntu 18.04
![image](https://user-images.githubusercontent.com/12887961/194722381-e7bd03bb-0d2e-4e11-b01e-f8a571b0c6c1.png)

Select any Basic Plan that works best. I would reccomend the 1GB/1 CPU at minimum.

We do not need any Block Storage as the 25GB included is fine for this purpose.

Select the Data Center location that best suites your needs.

Next we need to generate an SSH Key for connecting to our Droplet via SSH. This varies per OS.
Windows: https://www.howtogeek.com/762863/how-to-generate-ssh-keys-in-windows-10-and-windows-11/
Linux: https://linuxhint.com/generate-ssh-keys-on-linux/
macOS: https://docs.tritondatacenter.com/public-cloud/getting-started/ssh-keys/generating-an-ssh-key-manually/manually-generating-your-ssh-key-in-mac-os-x

Next, click the New SSH Key button on Digital Ocean, and paste the contents of your Private Key. You can name it whatever you like.
Once satisfied with the configuration, you can Create Droplet.

Once the droplet has had a chance to deploy, you can Access the console via the Access tab.
![image](https://user-images.githubusercontent.com/12887961/194722629-8cbce496-7e27-489f-bb12-5643cb3c9ee7.png)

# Installing Apache, mySQL, and NodeJS

First start by ensuring your packages are up to date.
```
sudo apt update
```

Install Apache
```
sudo apt install apache2
```

Add the Firewall rule for Apache
```
sudo ufw allow 'Apache'
```

Add the Firewall rule for OpenSSH
```
sudo ufw allow 'OpenSSH'
```

Verify the Firewall Rules to ensure everything was added.
```
sudo ufw status
```

Verify the status of Apache, it should be running.
```
sudo systemctl status apache2
```

Now, we have to create the Web Server portion that will hold the files exposed by your domain.
We will start by creating a directory in /var/www/ and replace your_domain with your domain name ex: google.com
```
sudo mkdir /var/www/your_domain
```

This command will modify privelages on the domain folder.
```
sudo chown -R $USER:$USER /var/www/your_domain
```

This modified a flag on the domain folder to help prevent any errors.
```
sudo chmod -R 755 /var/www/your_domain
```

Now we need to create a default index.html file in our domain folder for Apache to serve.
This command will open the Nano editor.
```
nano /var/www/your_domain/index.html
```

We can start with this for testing purposes.
```
<html>
    <head>
        <title>Welcome to Your_domain!</title>
    </head>
    <body>
        <h1>Success!  The your_domain virtual host is working!</h1>
    </body>
</html>
```
Use CTRL+X to Save, Y to confirm, and Return to confirm. Ensure the filename is correct. (index.html)

This command will create a Configuration file for Apache specific to this domain.
Again opening in the Nano Editor.
```
sudo nano /etc/apache2/sites-available/your_domain.conf
```

This code essentially exposes the HTTP and Websocket Ports on 80, and 443.
When a request is received to the respective locations, these will be upgraded to WebSocket connections to ensure the game client can function correctly.
Be sure to change your_domain to your actual domain name (ex. google.com)
```
ServerName your_domain

<VirtualHost *:80>
  LoadModule proxy_module modules/mod_proxy.so
  LoadModule proxy_http_module modules/mod_proxy_http.so
  <Directory /var/www/your_domain>
      Options Indexes FollowSymLinks
      AllowOverride All
      Require all granted
  </Directory>

  ServerAdmin webmaster@localhost
  ServerAlias your_domain

  DocumentRoot /var/www/your_domain

  ErrorLog ${APACHE_LOG_DIR}/error.log
  CustomLog ${APACHE_LOG_DIR}/acess.log combined
</VirtualHost>

<VirtualHost *:443>
  RewriteEngine on

  RewriteCond %{REQUEST_URI} ^/world/login [NC]
  RewriteCond %{HTTP:Upgrade} websocket [NC]
  RewriteCond %{HTTP:Connection} upgrade [NC]
  RewriteRule /(.*) ws://localhost:6111/$1 [P,L]
  ProxyPass /world/login http://localhost:6111

  RewriteCond %{REQUEST_URI} ^/world/blizzard [NC]
  RewriteCond %{HTTP:Upgrade} websocket [NC]
  RewriteCond %{HTTP:Connection} upgrade [NC]
  RewriteRule /(.*) ws://localhost:6112/$1 [P,L]
  ProxyPass /world/blizzard http://localhost:6112
</VirtualHost>
```

These commands tell Apache to enable mod_proxy, mod_proxy_http, and mod_proxy_wstunnel.
The config files we created above utilize these for handling of the connection upgrades when required.
```
a2enmod proxy
```
```
a2enmod proxy_http
```
```
a2enmod proxy_wstunnel
```

This command tells Apache to enable our domain configuration, and create a symlink to sites-enabled.
```
sudo a2ensite your_domain.conf
```

This command tells Apache to disable the default configuration that is enabled by default on installation.
Leaving this exposed can be a security risk.
```
sudo a2dissite 000-default.conf
```

This command will test the Apache Configurations to ensure valid syntax, and that there are no errors.
You may get warnings about the Imports given they already exist, but that's fine. Better safe than sorry.
```
sudo apache2ctl configtest
```

Finally, this command will restart Apache with our new configuration settings. Given all went correctly, you should not receive any error messages.
```
sudo systemctl restart apache2
```

# Testing the Web Server:
To make sure everything we just did is actually in effect, you can attempt visiting your site via the ipv4 exposed by your Droplet in Digital Oceans management page.
This should display our Test Page we made in index.html earlier.

# Cloning the Penguin Sauce.
Discord is a valuable resource.
If you have been able to get the Penguin Sauce, we will need to clone that into our site folder.

Start by navigating to your domain folder under /var/www/your_domain.
```
cd /var/www/your_domain
```

Run a git clone on the Penguin sauce
```
git clone <penguin_sauce here>
```

There's probably some fancy command to make Git clone it directly into this folder, but I do not know that one.
So, unfortunatly this requires us to move the assets folder out from the Penguin Sauce directly into the root of the your_domain directory.
```
mvdir PenguinSauce-master /var/www/your_domain/
```

# Cloning the Yukon Client
Now that we got that outta the way, lets clone the yukon client directly from their github.
```
git clone https://github.com/wizguin/yukon.git
```

Now, thing is here. We don't want all that stuff.
We only want the assets folder, and a built copy of the Client.

Start by moving the content of assets and merge that into the root of /var/www/your_domain with the existing assets folder.
This contains essential libraries needed by the client.

Next, run the build command.
```
cd /var/www/your_domain/yukon
```
```
npm run build
```

This will produce a new folder called dist; Move the contents of this folder into /var/www/your_domain.
Overwrite the existing index.html file from earlier testing.

# Setting up the Yukon Server

## Installing mySQL, and NodeJS.
```
sudo apt update
sudo apt install mysql-server
sudo mysql_secure_installation
sudo systemctl start mysql.service
```

## Creating the Database, and Importing the yukon table.
```
sudo mysql -u root -p
```
```
CREATE DATABSE yukon;
```
```
CREATE USER 'sammy'@'localhost' IDENTIFIED BY 'password';
```
```
GRANT ALL PRIVILEGES ON *.* TO 'sammy'@'localhost' WITH GRANT OPTION;
```
```
exit
```

Note: Ensure you are in the yukon-server directory when executing this command to quickly import the Yukon SQL Tables schema into your database.
```
mysql -u username -p yukon < yukon.sql
```

### NodeJS Installation
```
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash - &&\
sudo apt-get install -y nodejs
```

For this, we will just navigate back up to var.
```
cd /var/
```

Now we need to clone the Yukon Server from github and navigate into it.
```
git clone https://github.com/wizguin/yukon-server.git
```
```
cd yukon-server
```

Now we will need to run an NPM install here.
```
npm install
```

Once this has completed, we need to edit the config_example.json file in the config directory.
Rename the file to be config.json.
Make sure to change the mysql user, and password to your credentials for the database we set up earlier.
And, ensure to set cors to your domain.

Once this is done, we need to regenerate a new secret by running:
```
npm run secret-gen
```

Before running the servers, we must build them for production by running:
```
npm run build
```

and finally to start the servers:
```
npm run start
```
