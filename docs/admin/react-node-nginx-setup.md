---
id: odoo-nginx
title: Setting Up Odoo with Nginx
---

# Setting Up Odoo with Nginx

This guide will walk you through setting up Odoo with Nginx as a reverse proxy.

## Prerequisites

- A server with Ubuntu 18.04 or later
- Root access to the server
- A domain name pointed to your server

## Step 1: Update and Upgrade the System

First, update and upgrade your system packages to the latest versions:

```sh
sudo apt update
sudo apt upgrade -y
```

## Step 2: Install Dependencies

Odoo requires several dependencies to be installed. Run the following commands to install them:

```sh
sudo apt install python3-pip build-essential wget git python3-dev python3-venv python3-wheel libxslt-dev libzip-dev libldap2-dev libsasl2-dev python3-setuptools node-less libjpeg-dev libpq-dev -y
```

## Step 3: Install PostgreSQL

Odoo uses PostgreSQL as its database backend. Install PostgreSQL with the following command:

```sh
sudo apt install postgresql -y
```

Once installed, create a new PostgreSQL user for Odoo:

```sh
sudo -u postgres createuser -s odoo
```

## Step 4: Create a System User for Odoo

Create a new system user for Odoo:

```sh
sudo adduser --system --home=/opt/odoo --group odoo
```

## Step 5: Install Wkhtmltopdf

Odoo uses Wkhtmltopdf for generating PDF reports. Download and install it with the following commands:

```sh
sudo apt install wget -y
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6-1/wkhtmltox_0.12.6-1.bionic_amd64.deb
sudo apt install ./wkhtmltox_0.12.6-1.bionic_amd64.deb
```

## Step 6: Install and Configure Odoo

Clone the Odoo repository from GitHub:

```sh
sudo git clone https://www.github.com/odoo/odoo --depth 1 --branch 17.0 --single-branch /opt/odoo/odoo
```

Create a Python virtual environment for Odoo:

```sh
cd /opt/odoo
python3 -m venv odoo-venv
```

Activate the virtual environment and install the required Python packages:

```sh
source odoo-venv/bin/activate
pip3 install wheel
pip3 install -r odoo/requirements.txt
deactivate
```

## Step 7: Configure Odoo

Create a configuration file for Odoo:

```sh
sudo cp /opt/odoo/odoo/debian/odoo.conf /etc/odoo.conf
```

Edit the configuration file to set the necessary parameters:

```sh
sudo nano /etc/odoo.conf
```

Here is an example configuration:

```ini
[options]
; This is the password that allows database operations:
admin_passwd = your_admin_password
db_host = False
db_port = False
db_user = odoo
db_password = False
addons_path = /opt/odoo/odoo/addons
logfile = /var/log/odoo/odoo.log
```

## Step 8: Create a Systemd Service File

Create a systemd service file to manage the Odoo service:

```sh
sudo nano /etc/systemd/system/odoo.service
```

Add the following content to the service file:

```makefile
[Unit]
Description=Odoo
Documentation=http://www.odoo.com
[Service]
# Ubuntu/Debian convention:
Type=simple
User=odoo
ExecStart=/opt/odoo/odoo-venv/bin/python3 /opt/odoo/odoo/odoo-bin -c /etc/odoo.conf
[Install]
WantedBy=default.target
```

## Step 9: Start and Enable the Odoo Service

Start the Odoo service and enable it to start on boot:

```sh
sudo systemctl daemon-reload
sudo systemctl start odoo
sudo systemctl enable odoo
```

## Step 10: Install Nginx

If Nginx is not already installed, install it using the following command:

```sh
sudo apt update
sudo apt install nginx
```

## Step 11: Configure Nginx

Create a new Nginx configuration file for Odoo:

```sh
sudo nano /etc/nginx/sites-available/odoo
```

Add the following content to the file:

```nginx
server {
    listen 80;
    server_name staff.mannyslab.com;

    # Redirect HTTP to HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name staff.mannyslab.com;

    ssl_certificate /etc/letsencrypt/live/staff.mannyslab.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/staff.mannyslab.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Odoo
    location / {
        proxy_pass http://127.0.0.1:8069;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
    }
    
    location /odoo/static/ {
        alias /opt/odoo/odoo/odoo/addons/web/static/;
        access_log off;
        expires max;
    }

    # Odoo Web Service Worker
    location /web/service-worker.js {
        proxy_pass http://127.0.0.1:8069/web/service-worker.js;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Replace `staff.mannyslab.com` with your actual domain name.

## Step 12: Enable the Nginx Configuration

Enable the configuration by creating a symlink:

```sh
sudo ln -s /etc/nginx/sites-available/odoo /etc/nginx/sites-enabled/
```

## Step 13: Test and Restart Nginx

Test the Nginx configuration for syntax errors:

```sh
sudo nginx -t
```

If the test is successful, restart Nginx to apply the changes:

```sh
sudo systemctl restart nginx
```

## Step 14: Secure the Server with SSL (Optional)

It's highly recommended to use SSL to secure your Odoo instance. You can use Let's Encrypt to obtain a free SSL certificate. Install Certbot:

```sh
sudo apt install certbot python3-certbot-nginx
```

Obtain and install the SSL certificate:

```sh
sudo certbot --nginx -d staff.mannyslab.com
```

Certbot will automatically configure SSL for your Nginx configuration and reload Nginx.

## Script to execute the steps above
```sh
#!/bin/bash

# Update and upgrade the system
sudo apt update
sudo apt upgrade -y

# Install dependencies
sudo apt install python3-pip build-essential wget git python3-dev python3-venv python3-wheel libxslt-dev libzip-dev libldap2-dev libsasl2-dev python3-setuptools node-less libjpeg-dev libpq-dev -y

# Install PostgreSQL
sudo apt install postgresql -y

# Create PostgreSQL user for Odoo
sudo -u postgres createuser -s odoo

# Create system user for Odoo
sudo adduser --system --home=/opt/odoo --group odoo

# Install Wkhtmltopdf
sudo apt install wget -y
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6-1/wkhtmltox_0.12.6-1.bionic_amd64.deb
sudo apt install ./wkhtmltox_0.12.6-1.bionic_amd64.deb

# Clone the Odoo repository
sudo git clone https://www.github.com/odoo/odoo --depth 1 --branch 17.0 --single-branch /opt/odoo/odoo

# Create Python virtual environment for Odoo
cd /opt/odoo
python3 -m venv odoo-venv

# Activate virtual environment and install required Python packages
source odoo-venv/bin/activate
pip3 install wheel
pip3 install pandas
pip3 install -r odoo/requirements.txt
deactivate

# Create Odoo configuration file
sudo cp /opt/odoo/odoo/debian/odoo.conf /etc/odoo.conf
mkdir -p /opt/odoo/odoo/custom-addons

# Configure Odoo
sudo bash -c 'cat <<EOF > /etc/odoo.conf
[options]
; This is the password that allows database operations:
admin_passwd = admin
db_host = False
db_port = False
db_user = odoo
db_password = False
addons_path = /opt/odoo/odoo/addons,/opt/odoo/odoo/custom-addons
logfile = /var/log/odoo/odoo.log
EOF'

# Create systemd service file for Odoo
sudo bash -c 'cat <<EOF > /etc/systemd/system/odoo.service
[Unit]
Description=Odoo
Documentation=http://www.odoo.com
[Service]
# Ubuntu/Debian convention:
Type=simple
User=odoo
ExecStart=/opt/odoo/odoo-venv/bin/python3 /opt/odoo/odoo/odoo-bin -c /etc/odoo.conf
[Install]
WantedBy=default.target
EOF'

# Start and enable the Odoo service
sudo systemctl daemon-reload
sudo systemctl start odoo
sudo systemctl enable odoo

# Install Nginx
sudo apt install nginx -y

# Configure Nginx for Odoo
sudo bash -c 'cat <<EOF > /etc/nginx/sites-available/odoo
server {
    listen 80;
    server_name staff.mannyslab.com;

    # Redirect HTTP to HTTPS
    location / {
        return 301 https://\$host\$request_uri;
    }
}

server {
    server_name staff.mannyslab.com;

    # Odoo
    location / {
        proxy_pass http://127.0.0.1:8069;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        proxy_redirect off;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
    }
    
    location /odoo/static/ {
        alias /opt/odoo/odoo/odoo/addons/web/static/;
        access_log off;
        expires max;
    }

    # Odoo Web Service Worker
    location /web/service-worker.js {
        proxy_pass http://127.0.0.1:8069/web/service-worker.js;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF'

# Enable the Nginx configuration
sudo ln -s /etc/nginx/sites-available/odoo /etc/nginx/sites-enabled/

# Test and restart Nginx
sudo nginx -t
sudo systemctl restart nginx

# Secure the server with SSL using Let's Encrypt
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d staff.mannyslab.com --non-interactive --agree-tos --email sec@mannyslab.com

echo "Odoo installation and configuration complete. Access your Odoo instance at http://staff.mannyslab.com"
```

## Conclusion

You have successfully set up Odoo with Nginx as a reverse proxy. Your Odoo instance should now be accessible via your domain name.

sudo certbot --nginx -d sendlibra-prod.duckdns.org --non-interactive --agree-tos --email sec@mannyslab.com