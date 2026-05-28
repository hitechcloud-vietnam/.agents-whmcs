# WHMCS Database Migration Workflow

## Purpose

Safe procedure for migrating WHMCS databases between servers, upgrading schema, or making structural changes. Ensures data integrity, minimal downtime, and rollback capability.

## Prerequisites

- MySQL/MariaDB access on source and destination
- WHMCS admin access
- Full backup capability
- Migration window scheduled
- Downtime communication plan

## Workflow Steps

### Step 1: Pre-Migration Planning

Document current state and requirements:

```bash
# Get current database size
mysql -u root -p -e "
SELECT 
    table_schema AS 'Database',
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)'
FROM information_schema.tables
WHERE table_schema = 'your_whmcs_db'
GROUP BY table_schema;"

# List all tables and row counts
mysql -u root -p -e "
SELECT 
    table_name,
    table_rows,
    ROUND(data_length / 1024 / 1024, 2) AS 'Data (MB)',
    ROUND(index_length / 1024 / 1024, 2) AS 'Index (MB)'
FROM information_schema.tables
WHERE table_schema = 'your_whmcs_db'
ORDER BY data_length DESC;"

# Get WHMCS version
grep "version" /var/www/whmcs/init.php | head -5
```

Create migration checklist document:

```markdown
## Migration Plan

**Date:** YYYY-MM-DD
**Source:** old-server.example.com
**Destination:** new-server.example.com
**Downtime Window:** 2:00 AM - 4:00 AM UTC

### Databases to Migrate:
- whmcs_main (current: 2.3 GB)
- whmcs_logs (current: 1.1 GB)

### Steps:
1. Pre-migration backup
2. Test migration on staging
3. Production migration
4. Verification

### Rollback Plan:
- Keep old server online for 24 hours
- Point DNS back if issues detected
```

### Step 2: Create Full Backup

Always backup before any migration:

```bash
#!/bin/bash
# backup_whmcs_pre_migration.sh

BACKUP_DIR="/var/backups/whmcs/pre_migration_$(date +%Y%m%d_%H%M%S)"
mkdir -p $BACKUP_DIR

# Database backup
mysqldump -u root -p \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --hex-blob \
    --complete-insert \
    whmcs_main > $BACKUP_DIR/whmcs_main.sql

mysqldump -u root -p \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    whmcs_logs > $BACKUP_DIR/whmcs_logs.sql

# File backup
tar -czf $BACKUP_DIR/whmcs_files.tar.gz \
    --exclude='*.log' \
    --exclude='cache/*' \
    --exclude='storage/logs/*' \
    /var/www/whmcs/

# Configuration backup
cp /var/www/whmcs/configuration.php $BACKUP_DIR/

# Verify backup integrity
tar -tzf $BACKUP_DIR/whmcs_files.tar.gz > /dev/null && echo "Files OK"
mysql -u root -p -e "SELECT COUNT(*) FROM whmcs_main.tblclients" > /dev/null && echo "DB OK"

# Generate checksum
cd $BACKUP_DIR
sha256sum *.sql *.tar.gz *.php > checksums.sha256

echo "Backup complete: $BACKUP_DIR"
ls -la $BACKUP_DIR
```

### Step 3: Test Migration on Staging

Validate migration process before production:

```bash
# Create staging database
mysql -u root -p -e "CREATE DATABASE whmcs_staging CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# Test import timing
time mysql -u root -p whmcs_staging < /var/backups/whmcs/pre_migration_*/whmcs_main.sql

# Verify imported data
mysql -u root -p -e "
SELECT 
    (SELECT COUNT(*) FROM whmcs_staging.tblclients) AS clients,
    (SELECT COUNT(*) FROM whmcs_staging.tblhosting) AS services,
    (SELECT COUNT(*) FROM whmcs_staging.tblinvoices) AS invoices,
    (SELECT COUNT(*) FROM whmcs_staging.tblorders) AS orders;"
```

Test WHMCS functionality on staging:

```php
// Run WHMCS upgrade check on staging
// Point staging configuration to staging database
// Update configuration.php:
// $db_host = 'localhost';
// $db_username = 'whmcs_user';
// $db_password = 'staging_password';
// $db_name = 'whmcs_staging';

// Access WHMCS admin, check:
// - Login works
// - Clients list correctly
// - Invoices generate
// - Orders process
// - Modules function
```

### Step 4: Prepare Destination Server

Set up the target environment:

```bash
# Install MySQL/MariaDB (if fresh server)
apt-get update && apt-get install -y mysql-server

# Configure MySQL for WHMCS
cat > /etc/mysql/mysql.conf.d/whmcs.cnf << 'EOF'
[mysqld]
max_allowed_packet = 64M
innodb_buffer_pool_size = 2G
innodb_log_file_size = 512M
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2

[client]
default-character-set = utf8mb4
EOF

systemctl restart mysql
```

Create WHMCS database on destination:

```sql
-- Create database and user
CREATE DATABASE whmcs_production CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'whmcs_user'@'localhost' IDENTIFIED BY 'secure_password_here';
GRANT ALL PRIVILEGES ON whmcs_production.* TO 'whmcs_user'@'localhost';
FLUSH PRIVILEGES;

-- Verify character set
ALTER DATABASE whmcs_production CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### Step 5: Execute Production Migration

During maintenance window:

```bash
#!/bin/bash
# migrate_whmcs_to_production.sh

set -e

SOURCE_DB="whmcs_main"
DEST_DB="whmcs_production"
BACKUP_FILE="/var/backups/whmcs/latest_whmcs_main.sql"

echo "Starting migration at $(date)"
echo "Disabling WHMCS..."

# Disable WHMCS in maintenance mode
# Set /var/www/whmcs/.maintenance file or use WHMCS admin

# Kill active database connections
mysql -u root -p -e "
SELECT id FROM information_schema.processlist 
WHERE db IN ('$SOURCE_DB');"

# Final backup before migration
mysqldump -u root -p \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --hex-blob \
    --complete-insert \
    $SOURCE_DB > "/var/backups/whmcs/final_backup_$(date +%Y%m%d_%H%M%S).sql"

echo "Starting database import..."

# Import to destination
mysql -u root -p $DEST_DB < $BACKUP_FILE

echo "Verifying import..."

# Verification queries
mysql -u root -p -e "
SELECT 
    (SELECT COUNT(*) FROM $DEST_DB.tblclients) AS clients,
    (SELECT COUNT(*) FROM $DEST_DB.tblhosting) AS services,
    (SELECT COUNT(*) FROM $DEST_DB.tblinvoices) AS invoices,
    (SELECT COUNT(*) FROM $DEST_DB.tblorders) AS orders,
    (SELECT COUNT(*) FROM $DEST_DB.tblactivitylog) AS activities;"

echo "Migration complete at $(date)"
```

### Step 6: Migrate Files (If Different Server)

If migrating to new server:

```bash
# Rsync WHMCS files
rsync -avz --progress \
    --exclude='configuration.php' \
    --exclude='*.log' \
    --exclude='cache/*' \
    --exclude='storage/logs/*' \
    --exclude='storage/templates_c/*' \
    /var/www/whmcs/ \
    user@new-server:/var/www/whmcs/

# Transfer configuration
scp /var/www/whmcs/configuration.php user@new-server:/var/www/whmcs/
```

### Step 7: Update Configuration

Update WHMCS configuration for new environment:

```php
// /var/www/whmcs/configuration.php on new server
<?php

$db_host = "localhost";
$db_username = "whmcs_user";
$db_password = "new_secure_password";
$db_name = "whmcs_production";

$cc_encryption_hash = 'YOUR_ENCRYPTION_HASH_FROM_BACKUP';

$systems_url = 'https://billing.yourdomain.com';
$domain = 'billing.yourdomain.com';

$display_errors = false;
$debug = false;

// Session configuration
$session_save_path = '/var/www/whmcs/data/sessions';
```

### Step 8: Post-Migration Verification

Verify everything works correctly:

```bash
#!/bin/bash
# verify_migration.sh

echo "=== Migration Verification ==="

# 1. Database integrity
echo "Checking database integrity..."
mysql -u root -p -e "
SELECT 'Clients' AS tbl, COUNT(*) AS cnt FROM whmcs_production.tblclients
UNION ALL
SELECT 'Services', COUNT(*) FROM whmcs_production.tblhosting
UNION ALL
SELECT 'Invoices', COUNT(*) FROM whmcs_production.tblinvoices
UNION ALL
SELECT 'Orders', COUNT(*) FROM whmcs_production.tblorders;"

# 2. Check for corrupted tables
mysqlcheck -u root -p --optimize whmcs_production

# 3. Verify WHMCS can connect
curl -s http://localhost/ | grep -i "whmcs" && echo "WHMCS responds"

# 4. Check admin login
# Manual: Login to WHMCS admin panel

# 5. Test critical functions
# - Create test invoice
# - Process test order
# - Verify email sending
```

PHP verification script:

```php
<?php
// /var/www/whmcs/verify_installation.php
require_once __DIR__ . '/init.php';

$checks = [];

// Check database connection
try {
    $result = Capsule::table('tblconfiguration')->first();
    $checks['database'] = ['status' => 'OK', 'message' => 'Connected'];
} catch (Exception $e) {
    $checks['database'] = ['status' => 'FAIL', 'message' => $e->getMessage()];
}

// Check file permissions
$writableDirs = [
    'storage/',
    'downloads/',
    'attachments/',
    'templates_c/'
];
foreach ($writableDirs as $dir) {
    $path = __DIR__ . '/' . $dir;
    $checks['permissions'][$dir] = is_writable($path) ? 'OK' : 'FAIL';
}

// Check required tables
$requiredTables = [
    'tblclients', 'tblhosting', 'tblinvoices',
    'tblorders', 'tblproducts', 'tblconfig'
];
foreach ($requiredTables as $table) {
    $exists = Capsule::schema()->hasTable($table);
    $checks['tables'][$table] = $exists ? 'OK' : 'MISSING';
}

header('Content-Type: application/json');
echo json_encode($checks, JSON_PRETTY_PRINT);
```

### Step 9: DNS and DNS Propagation

Update DNS to point to new server:

```bash
# Update DNS records (use your DNS provider's method)
# Common records to update:
# - A record for billing.yourdomain.com -> new_server_ip
# - MX record if using subdomain for email

# After DNS update, verify propagation
dig billing.yourdomain.com
nslookup billing.yourdomain.com

# Check SSL certificate
openssl s_client -connect billing.yourdomain.com:443 -servername billing.yourdomain.com
```

### Step 10: Cleanup and Monitoring

Post-migration tasks:

```bash
# Remove temporary files
rm -f /var/www/whmcs/verify_installation.php

# Clear WHMCS cache
php /var/www/whmcs/crons/cron.php?a=clearCache

# Re-enable WHMCS if in maintenance mode
# Remove .maintenance file or disable in admin

# Set up monitoring
# - Monitor database connections
# - Monitor disk space
# - Monitor error logs

# Keep old server on standby for 24-48 hours
# In case rollback is needed
```

## Verification Checklist

- [ ] Pre-migration backup completed and verified
- [ ] Staging migration successful
- [ ] Database imported without errors
- [ ] All tables present with correct data counts
- [ ] WHMCS admin accessible
- [ ] Client portal functional
- [ ] Invoice generation working
- [ ] Order processing functional
- [ ] Email notifications sending
- [ ] Payment gateways operational
- [ ] DNS updated and propagated
- [ ] SSL certificate valid
- [ ] Monitoring configured
- [ ] Documentation updated with new server details

## Rollback Procedure

If issues detected:

```bash
#!/bin/bash
# rollback_migration.sh

echo "Initiating rollback..."

# Point DNS back to old server
# Update DNS A record to old_server_ip

# Or on same server:
# Restore database from pre-migration backup
mysql -u root -p whmcs_main < /var/backups/whmcs/pre_migration_*/whmcs_main.sql

# Restore configuration
cp /var/backups/whmcs/pre_migration_*/configuration.php /var/www/whmcs/

echo "Rollback complete"
```

## Related Skills and Documentation

- [WHMCS Disaster Recovery](whmcs-disaster-recovery-workflow.md)
- [WHMCS Deployment Best Practices](whmcs-deployment-best-practices.md)
- [WHMCS Performance Audit](whmcs-performance-audit.md)
- WHMCS System Requirements: https://docs.whmcs.com/System_Requirements
- WHMCS Database Schema: https://developers.whmcs.com/advanced/database-schema/

## Notes

- Schedule migrations during low-traffic periods
- Communicate downtime to customers in advance
- Always test on staging first
- Keep backups for at least 7 days post-migration
- Document any customizations that need to be reapplied
- Verify timezone settings match source server
