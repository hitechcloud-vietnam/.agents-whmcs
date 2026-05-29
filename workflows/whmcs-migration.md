# WHMCS Migration Workflow

## Description
Complete workflow for migrating WHMCS from one server to another.

## Prerequisites
- Source WHMCS installation
- SSH access to both servers
- New server with LAMP/LEMP stack
- WHMCS license transfer capability

## Steps

### Step 1: Prepare Target Server
```bash
# Install required packages
apt update && apt upgrade -y
apt install -y php php-mysql php-gd php-imap php-curl php-xml php-zip php-mbstring php-bcmath php-intl mysql-server nginx

# Configure PHP
sed -i 's/memory_limit = .*/memory_limit = 256M/' /etc/php/*/php.ini
sed -i 's/upload_max_filesize = .*/upload_max_filesize = 64M/' /etc/php/*/php.ini
sed -i 's/post_max_size = .*/post_max_size = 64M/' /etc/php/*/php.ini
```

### Step 2: Backup Source Installation
```bash
# On source server
cd /var/www
tar -czvf whmcs_backup_$(date +%Y%m%d).tar.gz whmcs/

# Backup database
mysqldump -u root -p whmcs > whmcs_database_$(date +%Y%m%d).sql
```

### Step 3: Transfer Files
```bash
# On source server - send to target
rsync -avz -e ssh /var/www/whmcs_backup_*.tar.gz user@newserver:/tmp/
rsync -avz -e ssh /var/www/whmcs_database_*.sql user@newserver:/tmp/
```

### Step 4: Restore on Target Server
```bash
# Extract files
cd /var/www
tar -xzvf /tmp/whmcs_backup_*.tar.gz

# Restore database
mysql -u root -p -e "CREATE DATABASE whmcs CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -u root -p whmcs < /tmp/whmcs_database_*.sql
```

### Step 5: Update Configuration
```php
// Edit configuration.php
// Update database credentials if changed
// Update WHMCS license key
```

### Step 6: Update DNS
1. Update DNS A record to point to new server IP
2. Wait for DNS propagation
3. Verify SSL certificate

### Step 7: Post-Migration Tasks
```bash
# Clear template cache
cd /var/www/whmcs
php -q whmcs/cli/clear-cache.php

# Update permissions
chown -R www-data:www-data /var/www/whmcs
chmod -R 755 /var/www/whmcs
chmod 644 /var/www/whmcs/configuration.php
```

### Step 8: Verify Migration
1. Login to WHMCS admin panel
2. Check System Health
3. Test client area
4. Verify email sending
5. Test payment gateway connections
6. Check cron job functionality

### Step 9: Update License
Log into WHMCS Client Area and update server IP for license.

## Verification Checklist
- [ ] Admin login works
- [ ] Client area accessible
- [ ] Invoices generate correctly
- [ ] Payment processing works
- [ ] Emails send successfully
- [ ] Cron jobs executing
- [ ] All modules functional

## Rollback Plan
1. Revert DNS to old server
2. Restore from backup on old server
3. Contact WHMCS support if needed

## Tags
- migration
- server-transfer
- backup-restore