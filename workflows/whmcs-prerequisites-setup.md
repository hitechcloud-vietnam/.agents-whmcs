# WHMCS Prerequisites Setup Workflow

## Purpose
Install and configure all prerequisite software for WHMCS installation

## Prerequisites
- Root or sudo access
- Clean server installation
- Domain pointed to server IP

## Step 1: Update System Packages

### Ubuntu/Debian
```bash
apt update && apt upgrade -y
apt install -y software-properties-common curl wget unzip git nano htop
```

### CentOS/AlmaLinux/Rocky
```bash
dnf update -y
dnf install -y epel-release curl wget unzip git nano htop
```

## Step 2: Install Apache Web Server

### Ubuntu/Debian
```bash
apt install -y apache2
a2enmod rewrite headers ssl mime
systemctl enable apache2
systemctl start apache2
```

### CentOS/AlmaLinux
```bash
dnf install -y httpd
systemctl enable httpd
systemctl start httpd
```

### Apache Configuration for WHMCS

Create virtual host:
```bash
nano /etc/apache2/sites-available/whmcs.conf
```

```apache
<VirtualHost *:80>
    ServerName whmcs.yourdomain.com
    ServerAlias www.whmcs.yourdomain.com
    DocumentRoot /var/www/whmcs

    <Directory /var/www/whmcs>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/whmcs_error.log
    CustomLog ${APACHE_LOG_DIR}/whmcs_access.log combined
</VirtualHost>
```

Enable site and restart:
```bash
a2ensite whmcs.conf
a2dissite 000-default.conf
systemctl reload apache2
```

## Step 3: Install PHP 8.1

### Ubuntu 22.04+
```bash
apt install -y php8.1 php8.1-cli php8.1-common php8.1-curl php8.1-gd php8.1-mbstring php8.1-mysql php8.1-xml php8.1-zip php8.1-soap php8.1-bcmath php8.1-intl php8.1-imagick
```

### Ubuntu 20.04
```bash
apt install -y software-properties-common
add-apt-repository ppa:ondrej/php
apt update
apt install -y php8.1 php8.1-cli php8.1-common php8.1-curl php8.1-gd php8.1-mbstring php8.1-mysql php8.1-xml php8.1-zip php8.1-soap php8.1-bcmath php8.1-intl
```

### CentOS/AlmaLinux
```bash
dnf install -y php php-cli php-common php-curl php-gd php-mbstring php-mysqlnd php-xml php-zip php-soap php-bcmath php-intl php-pear php-devel
```

## Step 4: Configure PHP Settings

```bash
nano /etc/php/8.1/apache2/php.ini
```

Update these values:
```ini
memory_limit = 256M
upload_max_filesize = 10M
post_max_size = 64M
max_execution_time = 300
max_input_time = 300
date.timezone = UTC
display_errors = Off
allow_url_fopen = On
```

Save and restart Apache:
```bash
systemctl restart apache2
```

## Step 5: Install ionCube Loader

```bash
cd /tmp
wget https://downloads.ioncube.com/loader_downloads/ioncube_loaders_lin_x86-64.tar.gz
tar xzf ioncube_loaders_lin_x86-64.tar.gz

# Find PHP extension directory
php -i | grep "extension_dir"

# Copy ionCube loader
cp ioncube/ioncube_loader_lin_8.1.so /usr/lib/php/20210903/

# Add to php.ini
echo "zend_extension = /usr/lib/php/20210903/ioncube_loader_lin_8.1.so" >> /etc/php/8.1/apache2/php.ini
echo "zend_extension = /usr/lib/php/20210903/ioncube_loader_lin_8.1.so" >> /etc/php/8.1/cli/php.ini

# Restart Apache
systemctl restart apache2

# Verify
php -v
```

## Step 6: Install MySQL/MariaDB

### Ubuntu
```bash
apt install -y mariadb-server
systemctl enable mariadb
systemctl start mariadb
mysql_secure_installation
```

### CentOS/AlmaLinux
```bash
dnf install -y mariadb-server
systemctl enable mariadb
systemctl start mariadb
mysql_secure_installation
```

## Step 7: Create MySQL Database and User

```bash
mysql -u root -p
```

```sql
CREATE DATABASE whmcs_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'whmcs_user'@'localhost' IDENTIFIED BY 'StrongPassword123!';
GRANT ALL PRIVILEGES ON whmcs_db.* TO 'whmcs_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**Important:** Replace `StrongPassword123!` with a strong unique password.

## Step 8: Verify ionCube Installation

```bash
php -v
```

Output should show ionCube in the version string.

Create test file:
```bash
cat > /var/www/whmcs/test.php << 'EOF'
<?php
echo "PHP Version: " . phpversion() . "\n";
echo "ionCube: " . (extension_loaded("ionCube Loader") ? "Loaded" : "Not Loaded") . "\n";
?>
EOF
```

Visit `http://yourdomain.com/test.php` to verify.

## Step 9: Set Directory Permissions

```bash
chown -R www-data:www-data /var/www/whmcs
chmod 755 /var/www/whmcs
chmod 644 /var/www/whmcs/*.php
chmod 755 /var/www/whmcs/templates_c
chmod 755 /var/www/whmcs/downloads
chmod 755 /var/www/whmcs/screenshots
```

## Step 10: Configure SSL Certificate

### Using Let's Encrypt
```bash
apt install -y certbot python3-certbot-apache
certbot --apache -d whmcs.yourdomain.com -d www.whmcs.yourdomain.com
```

### Manual SSL Configuration
```bash
a2enmod ssl
systemctl restart apache2
```

Create SSL virtual host:
```bash
nano /etc/apache2/sites-available/whmcs-ssl.conf
```

```apache
<VirtualHost *:443>
    ServerName whmcs.yourdomain.com
    DocumentRoot /var/www/whmcs

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/your-cert.crt
    SSLCertificateKeyFile /etc/ssl/private/your-key.key
    SSLCertificateChainFile /etc/ssl/certs/your-chain.crt

    <Directory /var/www/whmcs>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

```bash
a2ensite whmcs-ssl.conf
systemctl reload apache2
```

## Step 11: Configure Firewall

```bash
# Ubuntu
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable

# CentOS
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

## Step 12: Install Additional Tools

```bash
# Install Composer (for module development)
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer

# Install Node.js (for admin tools)
curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
apt install -y nodejs

# Install WP-CLI alternative tools
wget https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm
```

## Step 13: Create Required Directories

```bash
mkdir -p /var/www/whmcs/templates_c
mkdir -p /var/www/whmcs/downloads
mkdir -p /var/www/whmcs/screenshots
mkdir -p /var/www/whmcs/attachments
mkdir -p /var/www/whmcs/.htaccess_backup

chown -R www-data:www-data /var/www/whmcs
chmod 755 /var/www/whmcs/templates_c
chmod 755 /var/www/whmcs/downloads
chmod 755 /var/www/whmcs/screenshots
chmod 755 /var/www/whmcs/attachments
```

## Step 14: Configure .htaccess for WHMCS

```bash
nano /var/www/whmcs/.htaccess
```

```apache
# WHMCS Required .htaccess
DirectoryIndex index.php index.html

# Protect configuration files
<FilesMatch "^\.ht">
    Require all denied
</FilesMatch>

<FilesMatch "\.ini$">
    Require all denied
</FilesMatch>

# Prevent directory listing
Options -Indexes

# Enable rewrite engine
RewriteEngine On

# Deny access to sensitive files
RewriteRule ^\.htaccess$ - [F]
RewriteRule ^configuration\.php$ - [F]
RewriteRule ^cron\.sh$ - [F]

# Clean URLs
RewriteBase /
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^([^/]+)/?$ index.php?rp=$1 [L,QSA]
```

## Verification Checklist

- [ ] System packages updated
- [ ] Apache installed and configured
- [ ] PHP 8.1+ installed with all extensions
- [ ] PHP settings configured correctly
- [ ] ionCube Loader installed and verified
- [ ] MySQL/MariaDB installed
- [ ] WHMCS database and user created
- [ ] SSL certificate configured
- [ ] Firewall configured
- [ ] Directory permissions set
- [ ] .htaccess configured

## Next Steps
Proceed to: `whmcs-installation-wizard.md`
