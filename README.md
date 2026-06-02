## Video Explainer

Watch the full video walkthrough here: https://youtu.be/FstuiqhFNn4

# ICT171 Assignment 3 - Cloud Server Project

**Student:** Muhammad Mustafa  
**Student Number:** 35107637  
**Unit:** ICT171 - Introduction to Server Environments and Architectures  
**Live Site:** https://mustafablog.ddns.net  
**Server IP:** 20.5.9.25  

---

## Overview

For this assignment I set up a personal blog website on a Linux virtual machine hosted in Microsoft Azure. I installed the LAMP stack manually and deployed WordPress as the content management system. I also configured a DNS hostname and secured the site with an SSL certificate.

The goal was to build a server from scratch using Infrastructure as a Service, document the process, and make the site publicly accessible.

---

## Server Specifications

- **Cloud Provider:** Microsoft Azure
- **OS:** Ubuntu 24.04 LTS
- **Web Server:** Apache2
- **Database:** MySQL
- **Language:** PHP
- **CMS:** WordPress
- **DNS:** mustafablog.ddns.net (No-IP free DNS)
- **SSL:** Let's Encrypt via Certbot

---

## Step 1: Create the Azure Virtual Machine

I created an Ubuntu 24.04 LTS virtual machine through the Azure portal. During setup I made sure to allow inbound traffic on ports 22 (SSH), 80 (HTTP), and 443 (HTTPS) in the network security group settings.

Once the VM was running I connected to it via SSH from my local machine:

```bash
ssh azureuser@20.5.9.25
```

---

## Step 2: Install the LAMP Stack

The first thing I did on the server was update the package list and install Apache, MySQL, and PHP along with the PHP extensions that WordPress needs:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install apache2 mysql-server php php-mysql libapache2-mod-php php-curl php-gd php-mbstring php-xml php-xmlrpc -y
```

I then started both Apache and MySQL and set them to start automatically on reboot:

```bash
sudo systemctl start apache2
sudo systemctl enable apache2
sudo systemctl start mysql
sudo systemctl enable mysql
```

---

## Step 3: Set Up the MySQL Database

I logged into MySQL and created a dedicated database and user for WordPress:

```bash
sudo mysql
```

```sql
CREATE DATABASE wordpress;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'WpPass123';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

I kept the database separate from the root user as a basic security practice.

---

## Step 4: Download and Configure WordPress

I downloaded the latest version of WordPress into the /tmp directory, extracted it, and moved it to the web root:

```bash
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xvzf latest.tar.gz
sudo mv wordpress /var/www/html/wordpress
sudo chown -R www-data:www-data /var/www/html/wordpress
sudo chmod -R 755 /var/www/html/wordpress
```

I then copied the sample config file and edited it with my database credentials:

```bash
sudo cp /var/www/html/wordpress/wp-config-sample.php /var/www/html/wordpress/wp-config.php
sudo nano /var/www/html/wordpress/wp-config.php
```

I updated these lines:

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'WpPass123' );
define( 'DB_HOST', 'localhost' );
```

---

## Step 5: Configure Apache Virtual Host

I created a new Apache virtual host config file for WordPress rather than using the default one:

```bash
sudo nano /etc/apache2/sites-available/wordpress.conf
```

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/wordpress

    <Directory /var/www/html/wordpress>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

I enabled the new site, enabled the rewrite module (needed for WordPress permalinks), disabled the default site, and restarted Apache:

```bash
sudo a2ensite wordpress.conf
sudo a2enmod rewrite
sudo a2dissite 000-default.conf
sudo systemctl restart apache2
```

I also moved the old default index.html out of the way so WordPress would load instead:

```bash
sudo mv /var/www/html/index.html /var/www/html/index.html.bak
```

---

## Step 6: Complete WordPress Setup

I visited `http://20.5.9.25` in a browser and the WordPress installation wizard appeared. I filled in the site title, created an admin account, and completed the installation.

---

## Step 7: Set Up DNS

I created a free account on No-IP and set up a hostname pointing to my server's public IP:

- **Hostname:** mustafablog.ddns.net
- **Record Type:** A
- **IP Address:** 20.5.9.25

After a short wait the domain resolved correctly and my WordPress site was accessible at `http://mustafablog.ddns.net`.

---

## Step 8: Install SSL/TLS Certificate

I installed Certbot and used it to automatically obtain and configure an SSL certificate for my domain:

```bash
sudo apt install certbot python3-certbot-apache -y
sudo certbot --apache -d mustafablog.ddns.net
```

Certbot handled the Apache configuration automatically. The certificate was issued by Let's Encrypt and is set to auto-renew.

I then updated the WordPress site URL in the database so it would use HTTPS:

```bash
mysql -u wpuser -pWpPass123 wordpress -e "UPDATE wp_options SET option_value='https://mustafablog.ddns.net' WHERE option_name='siteurl';"
mysql -u wpuser -pWpPass123 wordpress -e "UPDATE wp_options SET option_value='https://mustafablog.ddns.net' WHERE option_name='home';"
```

The site now loads securely at `https://mustafablog.ddns.net`.

---

## Step 9: Database Backup Script

I wrote a bash script to automate backing up the WordPress database. The script creates a timestamped SQL dump file in a dedicated backup directory. This is useful because it means I can restore the site quickly if anything goes wrong.

File saved at: `/usr/local/bin/wp-backup.sh`

```bash
#!/bin/bash
# WordPress Database Backup Script
# Author: Muhammad Mustafa - Student 35107637
# Description: Backs up the WordPress MySQL database with a timestamp
# Usage: sudo /usr/local/bin/wp-backup.sh

DATE=$(date +%Y-%m-%d_%H-%M-%S)
BACKUP_DIR="/var/backups/wordpress"
DB_NAME="wordpress"
DB_USER="wpuser"
DB_PASS="WpPass123"

# Create backup directory if it doesn't exist
mkdir -p $BACKUP_DIR

# Dump the database to a timestamped SQL file
mysqldump -u $DB_USER -p$DB_PASS $DB_NAME > $BACKUP_DIR/wp-backup-$DATE.sql

echo "Backup completed: $BACKUP_DIR/wp-backup-$DATE.sql"
```

To make the script executable and run it:

```bash
sudo chmod +x /usr/local/bin/wp-backup.sh
sudo /usr/local/bin/wp-backup.sh
```

The output confirms the backup file was created successfully:
The backup file can be verified by listing the directory:

```bash
ls -lh /var/backups/wordpress/
```

This showed a 1.2MB SQL file, confirming the database was exported correctly.

---

## References

- WordPress Documentation: https://wordpress.org/documentation/
- Let's Encrypt: https://letsencrypt.org
- No-IP Free DNS: https://www.noip.com
- Ubuntu Apache Documentation: https://ubuntu.com/server/docs/web-servers-apache
