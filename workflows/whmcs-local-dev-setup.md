# WHMCS Local Development Environment Setup

## Overview

This workflow guides you through setting up a local WHMCS development environment for testing, debugging, and developing custom modules and integrations.

## Prerequisites

- PHP 7.4+ (WHMCS 8.x requires PHP 7.4-8.1; WHMCS 8.8+ supports PHP 8.2-8.3)
- MySQL 5.7+ or MariaDB 10.3+
- Composer
- Git
- Node.js 16+ (for asset building)
- Local web server (Apache/Nginx) or Valet/Docker
- WHMCS license (development license available)

## Step-by-Step Instructions

### Step 1: Clone or Extract WHMCS

```bash
# Clone from existing repo (if applicable)
git clone git@github.com:your/whmcs-project.git whmcs-local

# Or extract WHMCS package to directory
mkdir whmcs-local
cd whmcs-local
tar -xzf ../whmcs-8.8.0.tar.gz
```

### Step 2: Configure Environment

Create `configuration.php` or use environment-specific config:

```php
<?php
// configuration.php
$whmcs_db_host = '127.0.0.1';
$whmcs_db_username = 'whmcs_dev';
$whmcs_db_password = 'dev_password';
$whmcs_db_name = 'whmcs_dev';
$whmcs_db_port = '3306';
$whmcs_username = 'admin';
$whmcs_password = 'hashed_password_here';
$whmcs_app_key = 'base64:generate_your_32_byte_key_here';
$whmcs_db_charset = 'utf8mb4';
$whmcs_basedir = __DIR__;
```

### Step 3: Configure Local Domain

Add to `/etc/hosts` (Linux/Mac) or `C:\Windows\System32\drivers\etc\hosts` (Windows):

```
127.0.0.1 whmcs-local.test
```

### Step 4: Create Database

```sql
CREATE DATABASE whmcs_dev CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'whmcs_dev'@'localhost' IDENTIFIED BY 'dev_password';
GRANT ALL PRIVILEGES ON whmcs_dev.* TO 'whmcs_dev'@'localhost';
FLUSH PRIVILEGES;
```

### Step 5: Install WHMCS

Access the installation wizard at `http://whmcs-local.test/install/install.php` and follow the guided setup.

### Step 6: Configure Development Settings

In `configuration.php`, add development flags:

```php
// Enable debugging
$display_errors = true;
ini_set('display_errors', 1);
error_reporting(E_ALL);

// Development mode
$whmcs_dev_mode = true;

// Disable cron for local (run manually)
$disable_cron = true;

// Swap Mail Driver to Log for testing
$whmcs_mail_driver = 'log';
$whmcs_mail_log_path = __DIR__ . '/storage/logs/mail.log';
```

### Step 7: Set Up Cron

Add to crontab (configurable schedule):

```bash
# Every 5 minutes for active development
*/5 * * * * cd /path/to/whmcs && php -q cron.php

# Or disable automatic cron via config
```

### Step 8: Configure IDE Debugging

For Xdebug with VS Code, create `.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Listen for Xdebug",
            "type": "php",
            "request": "launch",
            "port": 9003,
            "pathMappings": {
                "/var/www/whmcs": "${workspaceFolder}"
            }
        }
    ]
}
```

## Configure PHP-FPM

```ini
; php.ini development settings
[xdebug]
xdebug.mode=debug
xdebug.start_with_request=trigger
xdebug.client_host=127.0.0.1
xdebug.client_port=9003

[mail function]
SMTP = localhost
smtp_port = 1025
```

## Nginx Configuration Example

```nginx
server {
    listen 80;
    server_name whmcs-local.test;
    root /var/www/whmcs;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location /storage {
        deny all;
    }
}
```

## Expected Outcomes

- Local WHMCS installation accessible at local domain
- Email sending captured to log file instead of delivered
- PHP errors and warnings displayed in browser
- Xdebug connections accepted from IDE
- Cron jobs can be run manually
- Database ready for testing

## Testing Checklist

- [ ] WHMCS admin login works
- [ ] Local domain resolves correctly
- [ ] Database connection established
- [ ] Email capture to log file functional
- [ ] Cron.php runs without errors
- [ ] Xdebug breakpoint triggers in IDE
- [ ] Modules can be installed/uninstalled
- [ ] Custom hook execution works
- [ ] API endpoints respond correctly
- [ ] Template/asset changes reflect immediately (with cache clear)
