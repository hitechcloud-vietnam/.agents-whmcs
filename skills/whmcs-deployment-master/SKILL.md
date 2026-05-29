# WHMCS Deployment Master

## Overview
Master skill for WHMCS deployment strategies. Covers deployment automation, environment management, backups, and recovery procedures.

## Deployment Structure

```
/deployment/
├── config/
│   ├── production/
│   │   ├── config.php
│   │   └── database.php
│   ├── staging/
│   │   ├── config.php
│   │   └── database.php
│   └── development/
│       ├── config.php
│       └── database.php
├── scripts/
│   ├── deploy.sh
│   ├── rollback.sh
│   ├── backup.sh
│   └── migrate.sh
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── ansible/
│   └── playbook.yml
└── terraform/
    └── main.tf
```

## Deployment Script

```bash
#!/bin/bash
# /deployment/scripts/deploy.sh

set -e
set -u

# Configuration
APP_NAME="whmcs"
DEPLOY_DIR="/var/www/whmcs"
RELEASE_DIR="/var/www/releases"
SHARED_DIR="/var/www/shared"
BACKUP_DIR="/var/www/backups"
CURRENT_LINK="/var/www/current"

# Timestamps
TIMESTAMP=$(date +%Y%m%d%H%M%S)
RELEASE_NAME="${APP_NAME}-${TIMESTAMP}"

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')]${NC} $1"
}

error() {
    echo -e "${RED}[ERROR]${NC} $1"
    exit 1
}

warn() {
    echo -e "${YELLOW}[WARNING]${NC} $1"
}

# Step 1: Pre-deployment checks
log "Step 1: Running pre-deployment checks..."

# Check if deployment directory exists
if [ ! -d "$DEPLOY_DIR" ]; then
    error "Deployment directory does not exist: $DEPLOY_DIR"
fi

# Check if git repository exists
if [ ! -d "$DEPLOY_DIR/.git" ]; then
    error "Not a git repository: $DEPLOY_DIR"
fi

# Check disk space
DISK_SPACE=$(df -BG "$DEPLOY_DIR" | tail -1 | awk '{print $4}' | sed 's/G//')
if [ "$DISK_SPACE" -lt 5 ]; then
    error "Low disk space: ${DISK_SPACE}GB available. Need at least 5GB."
fi

# Step 2: Create release directory
log "Step 2: Creating release directory..."
mkdir -p "${RELEASE_DIR}/${RELEASE_NAME}"

# Step 3: Clone repository
log "Step 3: Cloning repository..."
cd "$DEPLOY_DIR"
git pull origin main

# Copy files to release directory
rsync -av --exclude='.git' \
      --exclude='node_modules' \
      --exclude='vendor' \
      --exclude='.env' \
      "${DEPLOY_DIR}/" "${RELEASE_DIR}/${RELEASE_NAME}/"

# Step 4: Install dependencies
log "Step 4: Installing dependencies..."
cd "${RELEASE_DIR}/${RELEASE_NAME}"
composer install --no-dev --optimize-autoloader --no-interaction

# Step 5: Set permissions
log "Step 5: Setting permissions..."
chmod -R 755 "${RELEASE_DIR}/${RELEASE_NAME}"
chmod -R 775 "${RELEASE_DIR}/${RELEASE_NAME}/attachments"
chmod -R 775 "${RELEASE_DIR}/${RELEASE_NAME}/downloads"
chmod -R 775 "${RELEASE_DIR}/${RELEASE_NAME}/templates_c"
chmod 600 "${RELEASE_DIR}/${RELEASE_NAME}/configuration.php"

# Step 6: Create symlinks for shared files
log "Step 6: Creating symlinks for shared files..."
mkdir -p "${SHARED_DIR}/attachments"
mkdir -p "${SHARED_DIR}/downloads"
mkdir -p "${SHARED_DIR}/uploads"

rm -rf "${RELEASE_DIR}/${RELEASE_NAME}/attachments"
rm -rf "${RELEASE_DIR}/${RELEASE_NAME}/downloads"
rm -rf "${RELEASE_DIR}/${RELEASE_NAME}/uploads"

ln -s "${SHARED_DIR}/attachments" "${RELEASE_DIR}/${RELEASE_NAME}/attachments"
ln -s "${SHARED_DIR}/downloads" "${RELEASE_DIR}/${RELEASE_NAME}/downloads"
ln -s "${SHARED_DIR}/uploads" "${RELEASE_DIR}/${RELEASE_NAME}/uploads"

# Step 7: Backup current release
log "Step 7: Creating backup..."
if [ -L "$CURRENT_LINK" ]; then
    CURRENT_RELEASE=$(readlink -f "$CURRENT_LINK")
    if [ -d "$CURRENT_RELEASE" ]; then
        BACKUP_NAME="${APP_NAME}-$(basename $CURRENT_RELEASE)-backup-${TIMESTAMP}"
        cp -r "$CURRENT_RELEASE" "${BACKUP_DIR}/${BACKUP_NAME}"
        log "Backup created: ${BACKUP_NAME}"
    fi
fi

# Step 8: Run database migrations
log "Step 8: Running database migrations..."
cd "${RELEASE_DIR}/${RELEASE_NAME}"
php artisan migrate --force

# Step 9: Clear and rebuild cache
log "Step 9: Clearing and rebuilding cache..."
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Step 10: Update symlink
log "Step 10: Updating symlink..."
ln -sfn "${RELEASE_DIR}/${RELEASE_NAME}" "$CURRENT_LINK"

# Step 11: Verify deployment
log "Step 11: Verifying deployment..."
if [ -f "${CURRENT_LINK}/configuration.php" ]; then
    log "Deployment verified successfully!"
else
    error "Deployment verification failed!"
fi

# Step 12: Restart services
log "Step 12: Restarting services..."
systemctl restart php-fpm
systemctl restart nginx

# Step 13: Cleanup old releases
log "Step 13: Cleaning up old releases..."
cd "${RELEASE_DIR}"
ls -1t | tail -n +6 | xargs -r rm -rf

log "Deployment completed successfully!"
log "Release: ${RELEASE_NAME}"
log "Timestamp: ${TIMESTAMP}"
```

## Rollback Script

```bash
#!/bin/bash
# /deployment/scripts/rollback.sh

set -e

RELEASE_DIR="/var/www/releases"
BACKUP_DIR="/var/www/backups"
CURRENT_LINK="/var/www/current"
APP_NAME="whmcs"

log() {
    echo -e "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

error() {
    echo -e "[ERROR] $1"
    exit 1
}

if [ $# -eq 0 ]; then
    # List available releases
    log "Available releases:"
    ls -1t "${RELEASE_DIR}" | head -5
    echo ""
    log "Available backups:"
    ls -1t "${BACKUP_DIR}" | head -5
    exit 1
fi

RELEASE=$1
log "Rolling back to: $RELEASE"

# Check if release or backup exists
if [ -d "${RELEASE_DIR}/${RELEASE}" ]; then
    SOURCE="${RELEASE_DIR}/${RELEASE}"
elif [ -d "${BACKUP_DIR}/${RELEASE}" ]; then
    SOURCE="${BACKUP_DIR}/${RELEASE}"
else
    error "Release or backup not found: $RELEASE"
fi

# Create backup of current
if [ -L "$CURRENT_LINK" ]; then
    CURRENT_RELEASE=$(readlink -f "$CURRENT_LINK")
    TIMESTAMP=$(date +%Y%m%d%H%M%S)
    BACKUP_NAME="${APP_NAME}-rollback-${TIMESTAMP}"
    cp -r "$CURRENT_RELEASE" "${BACKUP_DIR}/${BACKUP_NAME}"
    log "Current release backed up as: ${BACKUP_NAME}"
fi

# Rollback database
log "Rolling back database..."
if [ -f "${SOURCE}/database/migrations/rollback.sql" ]; then
    mysql -u root -p whmcs < "${SOURCE}/database/migrations/rollback.sql"
fi

# Update symlink
log "Updating symlink..."
ln -sfn "${SOURCE}" "$CURRENT_LINK"

# Clear cache
cd "$CURRENT_LINK"
php artisan config:clear
php artisan cache:clear

# Restart services
log "Restarting services..."
systemctl restart php-fpm
systemctl restart nginx

log "Rollback completed successfully!"
```

## Docker Configuration

```dockerfile
# /deployment/docker/Dockerfile
FROM php:8.1-fpm

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    libzip-dev \
    zip \
    unzip \
    supervisor \
    nginx \
    && rm -rf /var/lib/apt/lists/*

# Install PHP extensions
RUN docker-php-ext-install pdo_mysql mbstring zip exif pcntl bcmath gd

# Install Redis extension
RUN pecl install redis && docker-php-ext-enable redis

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Set working directory
WORKDIR /var/www/html

# Copy application
COPY . .

# Install dependencies
RUN composer install --no-dev --optimize-autoloader --no-interaction

# Set permissions
RUN chown -R www-data:www-data /var/www/html
RUN chmod -R 755 /var/www/html
RUN chmod -R 775 /var/www/html/attachments
RUN chmod -R 775 /var/www/html/downloads
RUN chmod -R 775 /var/www/html/templates_c

# Copy nginx configuration
COPY deployment/docker/nginx.conf /etc/nginx/sites-available/default

# Copy supervisor configuration
COPY deployment/docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Expose port
EXPOSE 80 443

# Start services
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

## Docker Compose

```yaml
# /deployment/docker/docker-compose.yml
version: '3.8'

services:
  whmcs:
    build:
      context: .
      dockerfile: deployment/docker/Dockerfile
    container_name: whmcs
    ports:
      - "80:80"
      - "443:443"
    environment:
      - APP_ENV=production
      - DB_HOST=whmcs_mysql
      - DB_DATABASE=whmcs
      - DB_USER=whmcs_user
      - DB_PASSWORD=${DB_PASSWORD}
    volumes:
      - whmcs_data:/var/www/html
      - ./configuration.php:/var/www/html/configuration.php:ro
    depends_on:
      - whmcs_mysql
      - whmcs_redis
    networks:
      - whmcs_network
    restart: unless-stopped

  whmcs_mysql:
    image: mysql:8.0
    container_name: whmcs_mysql
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=whmcs
      - MYSQL_USER=whmcs_user
      - MYSQL_PASSWORD=${DB_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - whmcs_network
    restart: unless-stopped
    command: --default-authentication-plugin=mysql_native_password

  whmcs_redis:
    image: redis:7-alpine
    container_name: whmcs_redis
    volumes:
      - redis_data:/data
    networks:
      - whmcs_network
    restart: unless-stopped

volumes:
  whmcs_data:
  mysql_data:
  redis_data:

networks:
  whmcs_network:
    driver: bridge
```

## Backup Script

```bash
#!/bin/bash
# /deployment/scripts/backup.sh

set -e

# Configuration
BACKUP_DIR="/var/backups/whmcs"
RETENTION_DAYS=30
TIMESTAMP=$(date +%Y%m%d%H%M%S)

# Database configuration
DB_HOST="${DB_HOST:-localhost}"
DB_NAME="${DB_NAME:-whmcs}"
DB_USER="${DB_USER:-root}"
DB_PASS="${DB_PASS:-}"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

error() {
    echo "[ERROR] $1" >&2
    exit 1
}

# Create backup directory
mkdir -p "${BACKUP_DIR}/daily"
mkdir -p "${BACKUP_DIR}/weekly"
mkdir -p "${BACKUP_DIR}/monthly"

# Determine backup type
DAY_OF_WEEK=$(date +%u)
DAY_OF_MONTH=$(date +%d)

if [ "$DAY_OF_MONTH" -eq 1 ]; then
    BACKUP_TYPE="monthly"
elif [ "$DAY_OF_WEEK" -eq 7 ]; then
    BACKUP_TYPE="weekly"
else
    BACKUP_TYPE="daily"
fi

log "Starting ${BACKUP_TYPE} backup..."

# Create database backup
log "Backing up database..."
if [ -z "$DB_PASS" ]; then
    mysqldump -h "$DB_HOST" -u "$DB_USER" "$DB_NAME" > "${BACKUP_DIR}/${BACKUP_TYPE}/db-${TIMESTAMP}.sql"
else
    mysqldump -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" > "${BACKUP_DIR}/${BACKUP_TYPE}/db-${TIMESTAMP}.sql"
fi

# Compress database backup
gzip "${BACKUP_DIR}/${BACKUP_TYPE}/db-${TIMESTAMP}.sql"

# Backup files
log "Backing up files..."
tar -czf "${BACKUP_DIR}/${BACKUP_TYPE}/files-${TIMESTAMP}.tar.gz" \
    /var/www/html/attachments \
    /var/www/html/downloads \
    /var/www/html/uploads \
    /var/www/html/configuration.php 2>/dev/null || true

# Create manifest
cat > "${BACKUP_DIR}/${BACKUP_TYPE}/manifest-${TIMESTAMP}.txt" << EOF
Backup Date: $(date)
Backup Type: ${BACKUP_TYPE}
Hostname: $(hostname)
WHMCS Version: $(cat /var/www/html/version.php 2>/dev/null | grep 'version' | cut -d"'" -f4)
Database: ${DB_NAME}
Database Host: ${DB_HOST}
Files Included:
  - /var/www/html/attachments
  - /var/www/html/downloads
  - /var/www/html/uploads
  - /var/www/html/configuration.php
EOF

# Upload to remote storage (example: S3)
if command -v aws &> /dev/null; then
    log "Uploading to S3..."
    aws s3 sync "${BACKUP_DIR}/${BACKUP_TYPE}/" "s3://your-bucket/whmcs/${BACKUP_TYPE}/"
fi

# Clean up old backups
log "Cleaning up old backups (retention: ${RETENTION_DAYS} days)..."
find "${BACKUP_DIR}" -name "*.sql.gz" -mtime +${RETENTION_DAYS} -delete
find "${BACKUP_DIR}" -name "*.tar.gz" -mtime +${RETENTION_DAYS} -delete
find "${BACKUP_DIR}" -name "manifest-*.txt" -mtime +${RETENTION_DAYS} -delete

# Create symbolic link to latest backup
ln -sf "${BACKUP_DIR}/${BACKUP_TYPE}/db-${TIMESTAMP}.sql.gz" "${BACKUP_DIR}/latest-db.sql.gz"
ln -sf "${BACKUP_DIR}/${BACKUP_TYPE}/files-${TIMESTAMP}.tar.gz" "${BACKUP_DIR}/latest-files.tar.gz"

log "Backup completed successfully!"
log "Backup location: ${BACKUP_DIR}/${BACKUP_TYPE}/"
```

## Environment Configuration

```php
<?php
// /deployment/config/environments/production.php

return [
    // Application
    'app' => [
        'debug' => false,
        'log_level' => 'error',
        'timezone' => 'UTC',
        'locale' => 'en_us',
    ],

    // Database
    'database' => [
        'host' => getenv('DB_HOST') ?: 'localhost',
        'port' => getenv('DB_PORT') ?: 3306,
        'database' => getenv('DB_NAME') ?: 'whmcs',
        'username' => getenv('DB_USER') ?: 'whmcs',
        'password' => getenv('DB_PASSWORD'),
        'charset' => 'utf8mb4',
        'collation' => 'utf8mb4_unicode_ci',
        'prefix' => 'tbl',
    ],

    // Cache
    'cache' => [
        'driver' => 'redis',
        'redis' => [
            'host' => getenv('REDIS_HOST') ?: 'localhost',
            'port' => getenv('REDIS_PORT') ?: 6379,
            'password' => getenv('REDIS_PASSWORD'),
            'database' => 0,
        ],
    ],

    // Session
    'session' => [
        'driver' => 'redis',
        'lifetime' => 120,
        'expire_on_close' => false,
        'encrypt' => true,
        'cookie_name' => 'WHMCSSID',
        'cookie_path' => '/',
        'cookie_domain' => '',
        'cookie_secure' => true,
        'cookie_httponly' => true,
        'cookie_samesite' => 'Lax',
    ],

    // Security
    'security' => [
        ' csrf_protection' => true,
        'xss_protection' => true,
        'hsts_enabled' => true,
        'hsts_max_age' => 31536000,
        'force_https' => true,
    ],

    // Performance
    'performance' => [
        'minify_css' => true,
        'minify_js' => true,
        'cache_templates' => true,
        'gzip_compression' => true,
        'browser_cache' => 31536000,
    ],
];
```

## Best Practices

1. **Environment Separation**: Always use separate environments for dev, staging, and production
2. **Configuration Management**: Never commit sensitive configuration to version control
3. **Automated Backups**: Schedule regular automated backups with retention policies
4. **Rollback Plan**: Always have a tested rollback plan before deployment
5. **Zero Downtime**: Plan deployments to minimize downtime
6. **Health Checks**: Implement health check endpoints
7. **Monitoring**: Monitor deployment metrics and errors
8. **Documentation**: Document deployment procedures
9. **Version Control**: Tag releases in version control
10. **Testing**: Always test in staging before production
