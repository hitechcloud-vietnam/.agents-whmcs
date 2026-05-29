# WHMCS Server Requirements Workflow

## Purpose
Verify and configure server meeting WHMCS minimum requirements

## Prerequisites
- SSH access to server
- Root or sudo privileges
- Basic knowledge of Linux commands

## Step 1: Check Operating System

**Required:** Linux (CentOS 7+, Ubuntu 18.04+, Debian 9+)

```bash
cat /etc/os-release
uname -a
```

**Recommended:** AlmaLinux 8/9, Ubuntu 20.04/22.04, Debian 11

## Step 2: Verify Web Server

### Apache
```bash
httpd -v
# OR
apache2 -v
```

**Requirements:**
- Apache 2.4+ with mod_ssl
- mod_rewrite enabled
- AllowOverride All for WHMCS directory

```bash
a2enmod rewrite ssl
systemctl restart apache2
```

### Nginx (Alternative)
```bash
nginx -v
```

**Requirements:**
- Nginx 1.18+ with SSL support
- PHP-FPM 8.1+

## Step 3: Check PHP Version

```bash
php -v
```

**Required:** PHP 8.1 minimum
**Recommended:** PHP 8.1 or 8.2

**Install/Update PHP if needed:**

Ubuntu/Debian:
```bash
apt update
apt install software-properties-common
add-apt-repository ppa:ondrej/php
apt update
apt install php8.1 php8.1-cli php8.1-common php8.1-curl php8.1-gd php8.1-mbstring php8.1-mysql php8.1-xml php8.1-zip php8.1-soap php8.1-bcmath php8.1-intl
```

CentOS/AlmaLinux:
```bash
dnf install epel-release
dnf update
dnf install php php-cli php-common php-curl php-gd php-mbstring php-mysqlnd php-xml php-zip php-soap php-bcmath php-intl
```

## Step 4: Verify PHP Extensions

```bash
php -m
```

**Required Extensions:**
- [ ] pdo_mysql or mysqli
- [ ] gd2
- [ ] curl
- [ ] xml
- [ ] zip
- [ ] mbstring
- [ ] openssl
- [ ] ionCube Loader (or Zend Guard Loader)

**Install missing extensions:**
```bash
# Ubuntu/Debian
apt install php8.1-mysql php8.1-gd php8.1-curl php8.1-zip php8.1-mbstring php8.1-bcmath php8.1-intl php8.1-soap

# CentOS/AlmaLinux
dnf install php-mysqlnd php-gd php-curl php-zip php-mbstring php-bcmath php-intl php-soap
```

## Step 5: Check MySQL/MariaDB

```bash
mysql --version
# OR
mariadb --version
```

**Requirements:**
- MySQL 5.7+ or MariaDB 10.3+

**Install if needed:**
```bash
# Ubuntu/Debian
apt install mysql-server

# CentOS/AlmaLinux
dnf install mariadb-server
systemctl enable mariadb
systemctl start mariadb
```

## Step 6: Verify RAM and Disk Space

```bash
free -h
df -h
```

**Minimum Requirements:**
- RAM: 2GB (4GB+ recommended)
- Disk: 20GB+ free space
- Swap: 2GB+ if RAM < 4GB

## Step 7: Check PHP Configuration

```bash
php -i | grep "memory_limit"
php -i | grep "upload_max_filesize"
php -i | grep "post_max_size"
php -i | grep "max_execution_time"
```

**Required PHP Settings:**
```ini
memory_limit = 256M
upload_max_filesize = 10M
post_max_size = 64M
max_execution_time = 300
max_input_time = 300
date.timezone = "UTC"
```

**Update php.ini:**
```bash
# Find php.ini location
php -i | grep "Loaded Configuration File"

# Edit php.ini
nano /etc/php/8.1/apache2/php.ini
# OR
nano /etc/php.ini
```

## Step 8: Verify ionCube Loader

```bash
php -v
# Look for "Zend Engine" with ionCube
```

**Install ionCube if missing:**
```bash
cd /tmp
wget https://downloads.ioncube.com/loader_downloads/ioncube_loaders_lin_x86-64.tar.gz
tar xzf ioncube_loaders_lin_x86-64.tar.gz
cp ioncube/ioncube_loader_lin_8.1.so /usr/lib/php/20210903/
```

**Add to php.ini:**
```ini
zend_extension = /usr/lib/php/20210903/ioncube_loader_lin_8.1.so
```

## Step 9: Configure SSL (Recommended)

```bash
# Install Certbot
apt install certbot python3-certbot-apache
# OR
dnf install certbot python3-certbot-nginx

# Generate certificate
certbot --apache -d yourdomain.com -d www.yourdomain.com
```

## Step 10: Verify All Requirements

Create and run verification script:

```bash
cat > /tmp/whmcs_check.php << 'EOF'
<?php
echo "PHP Version: " . phpversion() . "\n";
echo "Memory Limit: " . ini_get('memory_limit') . "\n";
echo "Upload Max: " . ini_get('upload_max_filesize') . "\n";
echo "Post Max: " . ini_get('post_max_size') . "\n";
echo "Max Exec Time: " . ini_get('max_execution_time') . "\n";
echo "\nExtensions:\n";
$required = ['pdo_mysql', 'mysqli', 'gd', 'curl', 'xml', 'zip', 'mbstring', 'openssl', 'ionCube Loader'];
foreach ($required as $ext) {
    echo "$ext: " . (extension_loaded(strtolower($ext)) || strpos(phpversion(), 'ionCube') !== false ? "OK" : "MISSING") . "\n";
}
?>
EOF
php /tmp/whmcs_check.php
```

## Step 11: Create WHMCS Directory

```bash
mkdir -p /var/www/whmcs
chown -R www-data:www-data /var/www/whmcs
chmod 755 /var/www/whmcs
```

## Step 12: Firewall Configuration

```bash
# UFW
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable

# OR firewalld
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

## Verification Checklist

- [ ] OS version compatible
- [ ] Web server installed and running
- [ ] PHP 8.1+ installed
- [ ] All required PHP extensions loaded
- [ ] MySQL/MariaDB installed
- [ ] Sufficient RAM and disk space
- [ ] ionCube Loader installed
- [ ] SSL certificate ready
- [ ] Firewall configured
- [ ] WHMCS directory created

## Next Steps
Proceed to: `whmcs-prerequisites-setup.md`
