# WHMCS Installation Workflow

## Description
Step-by-step guide for fresh WHMCS installation on a web server.

## Prerequisites
- Web server (Apache/Nginx) with PHP 8.1+ and MySQL 8.0+
- WHMCS license key
- SSL certificate ready
- Domain pointed to server

## Steps

### Step 1: Download WHMCS
```bash
# Download latest WHMCS from official source
wget https://download.whmcs.com/whmcs.zip
unzip whmcs.zip -d /var/www/
```

### Step 2: Set Permissions
```bash
cd /var/www/whmcs
chmod 755 .
chmod 755 configuration.php 2>/dev/null || true
chmod 755 /var/www/whmcs/{templates_c,attachments,downloads,language}/ 2>/dev/null || true
```

### Step 3: Create Database
```sql
CREATE DATABASE whmcs CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'whmcs'@'localhost' IDENTIFIED BY 'strong_password';
GRANT ALL PRIVILEGES ON whmcs.* TO 'whmcs'@'localhost';
FLUSH PRIVILEGES;
```

### Step 4: Web Server Configuration

**Apache VirtualHost:**
```apache
<VirtualHost *:443>
    ServerName whmcs.example.com
    DocumentRoot /var/www/whmcs
    SSLEngine on
    SSLCertificateFile /path/to/cert.pem
    SSLCertificateKeyFile /path/to/key.pem
    
    <Directory /var/www/whmcs>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

**Nginx:**
```nginx
server {
    listen 443 ssl;
    server_name whmcs.example.com;
    root /var/www/whmcs;
    index index.php;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

### Step 5: Run Installation Wizard
1. Navigate to `https://whmcs.example.com/install/install.php`
2. Accept license agreement
3. Enter database credentials
4. Enter admin credentials
5. Enter license key
6. Complete setup

### Step 6: Post-Installation Security
```bash
# Remove install directory
rm -rf /var/www/whmcs/install/

# Set secure permissions
chmod 444 configuration.php
find /var/www/whmcs/templates_c -type f -exec chmod 644 {} \;
find /var/www/whmcs/attachments -type f -exec chmod 644 {} \;
```

### Step 7: Configure Cron Jobs
```bash
# Add to crontab
crontab -e

# Add this line (runs every 5 minutes)
*/5 * * * * php -q /var/www/whmcs/crons/cron.php
```

### Step 8: Initial WHMCS Configuration
1. Login to admin area
2. Go to Configuration > System Settings > General
3. Set company details
4. Configure email templates
5. Setup payment gateways
6. Configure tax rules

## Verification
- Access WHMCS admin panel
- Check System Health in Utilities
- Verify cron is running
- Test email sending

## Troubleshooting
- Check `configuration.php` has correct credentials
- Verify PHP extensions: `pdo_mysql`, `gd`, `imap`, `curl`, `xml`, `zip`
- Check PHP memory_limit minimum 256M
- Review `logs/` directory for errors

## Tags
- installation
- setup
- initial-configuration