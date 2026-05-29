# WHMCS Local Development Workflow

## Overview
This workflow guides developers through setting up a local WHMCS development environment for module development and testing.

## Prerequisites
- Docker and Docker Compose (recommended)
- Or LAMP/LEMP stack
- PHP 8.1+
- MySQL 8.0+
- Composer
- Git

## Option A: Docker Setup (Recommended)

### Step 1: Create Docker Compose Configuration

```yaml
# docker-compose.yml
version: '3.8'

services:
  # WHMCS Application
  whmcs:
    image: php:8.1-apache
    container_name: whmcs-dev
    volumes:
      - ./whmcs:/var/www/html
      - ./whmcs-storage:/var/www/html/storage
      - ./whmcs-module:/var/www/html/modules/addons/your_module
    ports:
      - "8080:80"
    environment:
      - APACHE_DOCUMENT_ROOT=/var/www/html
    depends_on:
      - mysql
    networks:
      - whmcs-network
    # Mount custom Apache config
    # volumes_from:
    #   - phpfpm

  # PHP-FPM for better performance
  phpfpm:
    image: php:8.1-fpm
    container_name: whmcs-phpfpm
    volumes:
      - ./whmcs:/var/www/html
      - ./whmcs-storage:/var/www/html/storage
      - ./whmcs-module:/var/www/html/modules/addons/your_module
    depends_on:
      - mysql
    networks:
      - whmcs-network

  # MySQL Database
  mysql:
    image: mysql:8.0
    container_name: whmcs-mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: whmcs
      MYSQL_USER: whmcs
      MYSQL_PASSWORD: whmcspassword
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - ./mysql/init:/docker-entrypoint-initdb.d
    networks:
      - whmcs-network
    command: --default-authentication-plugin=mysql_native_password

  # phpMyAdmin (optional)
  phpmyadmin:
    image: phpmyadmin:latest
    container_name: whmcs-phpmyadmin
    environment:
      PMA_HOST: mysql
      PMA_USER: root
      PMA_PASSWORD: rootpassword
    ports:
      - "8081:80"
    depends_on:
      - mysql
    networks:
      - whmcs-network

  # Redis for caching (optional)
  redis:
    image: redis:7-alpine
    container_name: whmcs-redis
    ports:
      - "6379:6379"
    networks:
      - whmcs-network

volumes:
  mysql-data:

networks:
  whmcs-network:
    driver: bridge
```

### Step 2: Apache Configuration

```apache
# apache-whmcs.conf

<VirtualHost *:80>
    ServerName whmcs.local
    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Security headers
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-Content-Type-Options "nosniff"

    # Error and access logs
    ErrorLog ${APACHE_LOG_DIR}/whmcs-error.log
    CustomLog ${APACHE_LOG_DIR}/whmcs-access.log combined
</VirtualHost>
```

### Step 3: PHP Configuration

```ini
; php-development.ini
[PHP]
memory_limit = 256M
max_execution_time = 300
upload_max_filesize = 100M
post_max_size = 100M

; Extensions
extension=pdo_mysql
extension=mysqli
extension=json
extension=mbstring
extension=curl
extension=gd
extension=intl
extension=zip
extension=openssl

[Date]
date.timezone = UTC

[Session]
session.save_handler = redis
session.save_path = "tcp://redis:6379"

[opcache]
opcache.enable = 1
opcache.memory_consumption = 128
opcache.max_accelerated_files = 10000
opcache.revalidate_freq = 0
opcache.validate_timestamps = 1

[xdebug]
xdebug.mode = debug
xdebug.client_host = host.docker.internal
xdebug.client_port = 9003
xdebug.idekey = PHPSTORM
```

## Option B: Traditional LAMP Setup

### Step 1: Install System Dependencies

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install -y \
    apache2 \
    mysql-server \
    php8.1 \
    php8.1-mysql \
    php8.1-mbstring \
    php8.1-gd \
    php8.1-curl \
    php8.1-intl \
    php8.1-zip \
    php8.1-xml \
    php8.1-cli \
    php8.1-common \
    php8.1-opcache \
    redis-server \
    composer \
    git

# Enable required PHP modules
sudo phpenmod pdo_mysql mysqli mbstring gd curl intl zip xml opcache

# Configure PHP
sudo cp /etc/php/8.1/apache2/php.ini /etc/php/8.1/apache2/php.ini.bak
sudo sed -i 's/memory_limit = .*/memory_limit = 256M/' /etc/php/8.1/apache2/php.ini
sudo sed -i 's/upload_max_filesize = .*/upload_max_filesize = 100M/' /etc/php/8.1/apache2/php.ini
sudo sed -i 's/post_max_size = .*/post_max_size = 100M/' /etc/php/8.1/apache2/php.ini
sudo sed -i 's/;date.timezone =/date.timezone = UTC/' /etc/php/8.1/apache2/php.ini
```

### Step 2: Configure MySQL

```bash
# Secure MySQL installation
sudo mysql_secure_installation

# Create WHMCS database and user
sudo mysql -u root -p << 'EOF'
CREATE DATABASE whmcs CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'whmcs'@'localhost' IDENTIFIED BY 'your_secure_password';
GRANT ALL PRIVILEGES ON whmcs.* TO 'whmcs'@'localhost';
FLUSH PRIVILEGES;
EOF
```

### Step 3: Apache Virtual Host

```bash
# Create virtual host configuration
sudo tee /etc/apache2/sites-available/whmcs.conf << 'EOF'
<VirtualHost *:80>
    ServerName whmcs.local
    ServerAlias www.whmcs.local
    DocumentRoot /var/www/whmcs

    <Directory /var/www/whmcs>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/whmcs-error.log
    CustomLog ${APACHE_LOG_DIR}/whmcs-access.log combined
</VirtualHost>
EOF

# Enable site and rewrite module
sudo a2ensite whmcs.conf
sudo a2enmod rewrite
sudo systemctl restart apache2
```

## Step 4: WHMCS Installation

```bash
# Create WHMCS directory
sudo mkdir -p /var/www/whmcs
sudo chown -R $USER:$USER /var/www/whmcs

# Download WHMCS (you need a license)
# Place the whmcs.zip file in /var/www/whmcs
cd /var/www/whmcs
unzip whmcs.zip

# Set permissions
sudo chown -R www-data:www-data /var/www/whmcs
find /var/www/whmcs -type d -exec chmod 755 {} \;
find /var/www/whmcs -type f -exec chmod 644 {} \;
chmod 777 /var/www/whmcs/storage
chmod 777 /var/www/whmcs/vendor
```

## Step 5: WHMCS Configuration

```php
<?php
// configuration.php - Generated during installation

<?php
/**
 * WHMCS Configuration
 */

use WHMCS\Config\Application;
use WHMCS\Config\Database;

define('ROOTDIR', dirname(__DIR__);

if (!defined("WHMCS")) {
    define("WHMCS", ROOTDIR);
}

// Database Configuration
define('DB_HOST', getenv('DB_HOST') ?: 'localhost');
define('DB_NAME', getenv('DB_NAME') ?: 'whmcs');
define('DB_USERNAME', getenv('DB_USER') ?: 'whmcs');
define('DB_PASSWORD', getenv('DB_PASSWORD') ?: '');
define('DB_TYPE', 'mysql');
define('DB_PORT', getenv('DB_PORT') ?: '3306');

// Cryptographic Constants
define('AUTH_KEY', 'your-auth-key-here');
define('AUTH_SALT', 'your-auth-salt-here');

// System URLs
define('WHMCS_SYSTEM_URL', 'http://whmcs.local');
define('WHMCS_SYSTEM_SSLURL', '');

// Development Settings
$display_errors = true;
$debug = true;
$mysql_cache_time = 0;

// Additional configuration
$disable_auto_ip_to_hostname_check = true;
```

## Step 6: Module Development Setup

```bash
# Create module directory structure
mkdir -p modules/addons/your_module/{src/{Controller,Service,Helper},templates/admin,assets/{css,js},tests/{Unit,Feature},migrations}

# Initialize composer
cd modules/addons/your_module
composer init
composer require \
    whmcs/whmcs:^8.0 \
    --no-update
composer update

# Create autoloader
cat > src/autoload.php << 'EOF'
<?php
spl_autoload_register(function ($class) {
    $prefix = 'WHMCS\\Module\\Addon\\YourModule\\';
    $base_dir = __DIR__ . '/';

    $len = strlen($prefix);
    if (strncmp($prefix, $class, $len) !== 0) {
        return;
    }

    $relative_class = substr($class, $len);
    $file = $base_dir . str_replace('\\', '/', $relative_class) . '.php';

    if (file_exists($file)) {
        require $file;
    }
});
EOF
```

## Step 7: Development Tools

### Xdebug Configuration

```ini
; Add to php.ini
[xdebug]
zend_extension=xdebug.so
xdebug.mode=debug,develop
xdebug.client_host=127.0.0.1
xdebug.client_port=9003
xdebug.idekey=PHPSTORM
xdebug.start_with_request=trigger
```

### VS Code Launch Configuration

```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Listen for Xdebug",
            "type": "php",
            "request": "launch",
            "port": 9003,
            "pathMappings": {
                "/var/www/whmcs": "${workspaceFolder}/whmcs"
            },
            "xdebugSettings": {
                "max_children": 256,
                "max_data": 1024,
                "max_depth": 5
            }
        },
        {
            "name": "Launch WHMCS",
            "type": "php",
            "request": "launch",
            "program": "${workspaceFolder}/whmcs/index.php",
            "cwd": "${workspaceFolder}/whmcs",
            "url": "http://whmcs.local/index.php",
            "webRoot": "${workspaceFolder}/whmcs"
        }
    ]
}
```

## Step 8: Testing Locally

```bash
# Run tests locally
cd /var/www/whmcs/modules/addons/your_module

# Run unit tests
./vendor/bin/phpunit --testsuite Unit

# Run all tests with coverage
./vendor/bin/phpunit --coverage-html coverage/

# Watch mode
./vendor/bin/phpunit --watch
```

## Step 9: Database Development

### Import Sample Data

```sql
-- Create test clients
INSERT INTO `tblclients` (`firstname`, `lastname`, `email`, `companyname`, `password`, `currency`, `status`, `datecreated`)
VALUES
    ('John', 'Doe', 'john@example.com', 'Acme Corp', SHA1('password'), 1, 'Active', NOW()),
    ('Jane', 'Smith', 'jane@example.com', 'Tech Inc', SHA1('password'), 1, 'Active', NOW()),
    ('Bob', 'Wilson', 'bob@example.com', 'Startup LLC', SHA1('password'), 1, 'Active', NOW());

-- Create test products
INSERT INTO `tblproducts` (`type`, `name`, `description`, `welcomeemail`, `stockcontrol`, `qty`, `paytype`, `pricing`, `tax`, `status`)
VALUES
    ('hostingaccount', 'Starter Hosting', 'Basic hosting package', 0, 0, 0, 'recurring', '{"1":{"monthly":"5.00","quarterly":"13.50","annually":"48.00"}}', 1, 'Active'),
    ('hostingaccount', 'Professional Hosting', 'Professional hosting package', 0, 0, 0, 'recurring', '{"1":{"monthly":"10.00","quarterly":"27.00","annually":"96.00"}}', 1, 'Active');

-- Create test orders
INSERT INTO `tblorders` (`userid`, `ordernum`, `date`, `status`, `paymentmethod`)
SELECT id, CONCAT('ORD-', id, '-001'), NOW(), 'Active', 'paypal'
FROM `tblclients` LIMIT 1;
```

## Step 10: Hot Reload Development

### BrowserSync for Auto-Refresh

```javascript
// bs-config.js
module.exports = {
    proxy: "whmcs.local",
    port: 3000,
    files: [
        "whmcs/modules/addons/your_module/**/*.php",
        "whmcs/modules/addons/your_module/**/*.tpl",
        "whmcs/modules/addons/your_module/**/*.css",
        "whmcs/modules/addons/your_module/**/*.js"
    ],
    watchOptions: {
        ignored: ["node_modules", "vendor"]
    }
};
```

## Step 11: Troubleshooting

### Common Issues

**Cannot connect to MySQL:**
```bash
# Check MySQL is running
sudo systemctl status mysql

# Test connection
mysql -u whmcs -p whmcs

# Check permissions
sudo mysql -u root -p -e "SHOW GRANTS FOR 'whmcs'@'localhost';"
```

**Permission denied errors:**
```bash
# Fix WHMCS permissions
sudo chown -R www-data:www-data /var/www/whmcs
find /var/www/whmcs -type d -exec chmod 755 {} \;
find /var/www/whmcs -type f -exec chmod 644 {} \;
chmod 777 /var/www/whmcs/storage
chmod 777 /var/www/whmcs/vendor
```

**Xdebug not connecting:**
```bash
# Verify Xdebug is loaded
php -m | grep xdebug

# Check configuration
php -i | grep xdebug
```

## Verification Checklist

- [ ] Docker installed and running (if using Docker)
- [ ] LAMP stack installed (if using traditional setup)
- [ ] WHMCS installed and accessible
- [ ] MySQL database created
- [ ] Apache/Nginx configured
- [ ] PHP configured with required extensions
- [ ] Module directory structure created
- [ ] Composer dependencies installed
- [ ] Xdebug configured
- [ ] Development tools set up
- [ ] Test data imported
- [ ] First test passing
- [ ] Can activate module in WHMCS admin
