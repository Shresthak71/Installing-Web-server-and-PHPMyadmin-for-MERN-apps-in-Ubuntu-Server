# Installing-Web-server-and-PHPMyadmin-for-MERN-apps-in-Ubuntu-Server
This works as the guide to install/configure webserver for MERN application to host at Ubuntu server

# LAMP + phpMyAdmin + Node.js on Ubuntu (Step‑by‑Step)

A copy‑paste friendly guide to set up **Apache (LAMP), MySQL, PHP, phpMyAdmin**, and **Node.js (via NVM)** on Ubuntu. Ideal for a fresh VPS/EC2 instance.

> **Tested on**: Ubuntu 22.04/24.04 LTS (should also work on recent Debian‑based distros)

---

## Table of Contents

* [Prerequisites](#prerequisites)
* [1) Install Node.js with NVM](#1-install-nodejs-with-nvm)
* [2) Install Apache (LAMP: A)](#2-install-apache-lamp-a)
* [3) Install MySQL Server (LAMP: M)](#3-install-mysql-server-lamp-m)
* [4) Install PHP (LAMP: P)](#4-install-php-lamp-p)
* [5) (Optional) Create an Apache Virtual Host](#5-optional-create-an-apache-virtual-host)
* [6) Prefer `index.php` over `index.html`](#6-prefer-indexphp-over-indexhtml)
* [7) Common PHP Extensions](#7-common-php-extensions)
* [8) (Optional) Remote MySQL Access — Securely](#8-optional-remote-mysql-access--securely)
* [9) Install & Secure phpMyAdmin](#9-install--secure-phpmyadmin)
* [10) Enable HTTPS with Let’s Encrypt (Recommended)](#10-enable-https-with-lets-encrypt-recommended)
* [11) Alternative: Local‑only phpMyAdmin via SSH Tunnel](#11-alternative-localonly-phpmyadmin-via-ssh-tunnel)
* [Troubleshooting](#troubleshooting)
* [Cheat Sheet](#cheat-sheet)

---

## Prerequisites

* A user with `sudo` privileges on a fresh Ubuntu server
* A domain name (optional but recommended, e.g., `example.com`)
* If using UFW firewall:

  * Ensure **OpenSSH** is allowed before enabling the firewall

```bash
sudo apt update
sudo apt install -y curl ca-certificates lsb-release gnupg
# If UFW is not enabled yet, allow SSH first
sudo ufw allow OpenSSH
sudo ufw enable
```

> Throughout this guide, replace placeholders like `your_domain`, `www.your_domain`, and `server_ip` with your actual values.

---

## 1) Install Node.js with NVM

Go to [https://nodejs.org/en/download](https://nodejs.org/en/download) and choose the NVM option, or run the commands below.

```bash
# Download and install nvm:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

# Load nvm without restarting the shell
. "$HOME/.nvm/nvm.sh"

# Install Node.js 20 LTS
nvm install 20

# Verify versions
node -v   # Should print "v20.19.4"
nvm current  # Should print "v20.19.4"
npm -v    # Should print "10.8.2"
```

---

## 2) Install Apache (LAMP: A)

```bash
sudo apt update
sudo apt install -y apache2

# Allow HTTP through UFW (and HTTPS later if needed)
sudo ufw app list
sudo ufw allow "Apache"        # opens port 80
# sudo ufw allow "Apache Full" # opens 80 & 443 (for HTTPS later)

# Ensure Apache starts on boot
sudo systemctl enable apache2
sudo systemctl status apache2 --no-pager
```

Verify in a browser: `http://server_ip`

---

## 3) Install MySQL Server (LAMP: M)

```bash
sudo apt install -y mysql-server

# Secure MySQL (recommended)
sudo mysql_secure_installation
```

### Set a password for `root` (choose a strong one)

> Use a mix of uppercase, lowercase, numbers, and special characters.

```bash
sudo mysql

ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Your_Str0ng_P@ss!';
FLUSH PRIVILEGES;
EXIT;
```

> You can verify authentication plugins in MySQL with:

```sql
SELECT user, host, plugin FROM mysql.user;
```

---

## 4) Install PHP (LAMP: P)

```bash
sudo apt install -y php libapache2-mod-php php-mysql php-cli
php -v

# (Good practice) enable Apache rewrite for .htaccess support
sudo a2enmod rewrite
sudo systemctl reload apache2
```

Create a quick test page (optional):

```bash
echo "<?php phpinfo();" | sudo tee /var/www/html/info.php > /dev/null
```

Visit `http://server_ip/info.php` to confirm PHP runs, then remove it:

```bash
sudo rm /var/www/html/info.php
```

---

## 5) (Optional) Create an Apache Virtual Host

```bash
# Create a web root for your site
sudo mkdir -p /var/www/your_domain
sudo chown -R $USER:$USER /var/www/your_domain

# Create the vhost file
sudo nano /etc/apache2/sites-available/your_domain.conf
```

Paste:

```apacheconf
<VirtualHost *:80>
    ServerName your_domain
    ServerAlias www.your_domain
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/your_domain

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    <Directory /var/www/your_domain>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Enable the site and (optionally) disable the default:

```bash
sudo a2ensite your_domain
sudo a2dissite 000-default
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Add a simple page:

```bash
echo "<h1>It works!</h1>" | tee /var/www/your_domain/index.html
```

> **Tip:** Put your website files inside `/var/www/your_domain`.

---

## 6) Prefer `index.php` over `index.html`

```bash
sudo nano /etc/apache2/mods-enabled/dir.conf
```

Ensure `index.php` comes first:

```apacheconf
<IfModule mod_dir.c>
    DirectoryIndex index.php index.html index.cgi index.pl index.xhtml index.htm
</IfModule>
```

```bash
sudo systemctl reload apache2
```

---

## 7) Common PHP Extensions

```bash
sudo apt install -y php-cli php-curl php-mbstring php-xml php-zip php-gd php-json
sudo systemctl restart apache2
```

---

## 8) (Optional) Remote MySQL Access — Securely

**Only do this if you truly need remote DB access. Prefer VPN/SSH tunnels in production.**

1. Bind MySQL to listen publicly:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Set:

```ini
bind-address = 0.0.0.0
```

```bash
sudo systemctl restart mysql
```

2. Create a remote user and grant permissions:

```sql
-- Inside MySQL shell (sudo mysql)
CREATE USER 'remote_user'@'%' IDENTIFIED BY 'StrongRemoteP@ss!';
GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

3. Restrict the firewall to a specific IP only (strongly recommended):

```bash
sudo ufw allow from <remote_ip> to any port 3306
```

> **Production Tip:** Consider **VPN** or **SSH tunneling** instead of exposing port 3306.

---

## 9) Install & Secure phpMyAdmin

**Prerequisite:** LAMP is installed and running.

```bash
sudo apt update
sudo apt install -y phpmyadmin php-mbstring php-zip php-gd php-json php-curl

# On some Ubuntu versions, Apache integration isn’t auto-enabled:
sudo a2enconf phpmyadmin
sudo phpenmod mbstring
sudo systemctl restart apache2
php -v
```

### Configure MySQL authentication

From the MySQL shell:

```sql
-- Check current auth plugins
SELECT user, host, plugin FROM mysql.user WHERE user = 'root';

-- Option 1: Use caching_sha2_password (default in MySQL 8)
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'Your_Str0ng_P@ss!';

-- Option 2: Use mysql_native_password (for older clients)
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Your_Str0ng_P@ss!';

-- Re-check
SELECT user, authentication_string, plugin, host FROM mysql.user;
```

Create a dedicated MySQL admin user (recommended):

```sql
CREATE USER 'sammy'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'Another_Strong_P@ss!';
-- Or switch to mysql_native_password if needed:
ALTER USER 'sammy'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Another_Strong_P@ss!';
GRANT ALL PRIVILEGES ON *.* TO 'sammy'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

Visit:

```
http(s)://your_domain_or_IP/phpmyadmin
```

### Add HTTP Basic Auth in front of phpMyAdmin (extra layer)

```bash
# Install htpasswd if missing
sudo apt install -y apache2-utils

# Edit phpMyAdmin’s Apache config to allow overrides
sudo nano /etc/apache2/conf-available/phpmyadmin.conf
```

Ensure it contains:

```apacheconf
<Directory /usr/share/phpmyadmin>
    Options SymLinksIfOwnerMatch
    DirectoryIndex index.php
    AllowOverride All
</Directory>
```

```bash
sudo systemctl restart apache2

# Create .htaccess in phpMyAdmin directory
sudo nano /usr/share/phpmyadmin/.htaccess
```

Paste:

```apacheconf
AuthType Basic
AuthName "Restricted Files"
AuthUserFile /etc/phpmyadmin/.htpasswd
Require valid-user
```

Create users for Basic Auth:

```bash
sudo htpasswd -c /etc/phpmyadmin/.htpasswd username
# Add more users
sudo htpasswd /etc/phpmyadmin/.htpasswd additionaluser

sudo systemctl restart apache2
```

Now phpMyAdmin will prompt for Basic Auth **before** the login page.

---

## 10) Enable HTTPS with Let’s Encrypt (Recommended)

> Requires a domain (`your_domain`) pointing to your server’s IP.

```bash
sudo apt install -y certbot python3-certbot-apache
sudo certbot --apache -d your_domain -d www.your_domain

# Test auto-renewal
sudo certbot renew --dry-run
```

Your site (including `/phpmyadmin`) will now be served over HTTPS.

---

## 11) Alternative: Local‑only phpMyAdmin via SSH Tunnel

If you prefer **not** to expose phpMyAdmin publicly, create a local tunnel from your machine to the server:

```bash
ssh -L 8888:localhost:80 username@your_server_ip
```

Then open:

```
http://localhost:8888/phpmyadmin
```

This is great for development or staging environments.

---

## Troubleshooting

**404 Not Found for `/phpmyadmin`**

* Ensure the config is enabled: `sudo a2enconf phpmyadmin && sudo systemctl restart apache2`
* Check file exists: `/etc/apache2/conf-available/phpmyadmin.conf`
* Verify Apache error log: `sudo tail -f /var/log/apache2/error.log`

**MySQL “auth\_socket” plugin prevents password login**

* Switch root to password auth (see Section 9):

  ```sql
  ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Your_Str0ng_P@ss!';
  FLUSH PRIVILEGES;
  ```

**UFW blocks access**

* Allow Apache: `sudo ufw allow "Apache"` (and `"Apache Full"` for HTTPS)
* For remote MySQL, restrict: `sudo ufw allow from <remote_ip> to any port 3306`

**Enable modules**

* `.htaccess` not working? `sudo a2enmod rewrite && sudo systemctl reload apache2`

**Service not starting on boot**

* `sudo systemctl enable apache2 mysql`

---

## Cheat Sheet

```bash
# Apache basics
sudo systemctl status apache2
sudo systemctl restart apache2
sudo apache2ctl configtest

# MySQL basics
sudo systemctl status mysql
sudo mysql -e "SELECT VERSION();"

# PHP version
php -v

# Firewall
sudo ufw status verbose

# Node.js via NVM
. "$HOME/.nvm/nvm.sh"
nvm install 20
node -v
npm -v
```

---

### Credits & References

* DigitalOcean: How To Install LAMP Stack on Ubuntu
* DigitalOcean: How To Install and Secure phpMyAdmin on Ubuntu

> Feel free to copy this README into your GitHub repository. Replace placeholders and commit! ✅

