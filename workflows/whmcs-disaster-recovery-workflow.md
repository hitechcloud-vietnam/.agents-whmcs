# WHMCS Disaster Recovery Workflow

## Purpose

Comprehensive procedures for recovering WHMCS from various disaster scenarios including data loss, server failure, corruption, and security incidents. Ensures business continuity with minimal data loss.

## Prerequisites

- Current backups stored off-site
- Recovery environment prepared
- Contact information for key personnel
- Communication templates ready
- Vendor support contacts

## Workflow Steps

### Step 1: Document Current Infrastructure

Maintain accurate infrastructure documentation:

```markdown
## WHMCS Infrastructure Documentation

**Last Updated:** YYYY-MM-DD
**Document Owner:** IT Team

### Production Environment
- **Server:** whmcs-prod-01.example.com
- **IP:** 192.168.1.100
- **OS:** Ubuntu 22.04 LTS
- **WHMCS Version:** 8.8.0

### Database Server
- **Server:** mysql-prod-01.example.com
- **IP:** 192.168.1.101
- **Database:** whmcs_main
- **Replication:** Master-Slave

### Storage
- **Primary:** /var/www/whmcs (500GB SSD)
- **Backups:** s3://company-whmcs-backups/
- **Attachments:** /var/www/whmcs/attachments

### Dependencies
- PHP 8.1
- MySQL 8.0
- Nginx 1.24
- Redis 7.0

### Critical Access Information
[Store securely with encryption - do not commit to git]
```

### Step 2: Establish Backup Strategy

Configure comprehensive backups:

```bash
#!/bin/bash
# /opt/scripts/whmcs-backup.sh
# Automated backup script - run daily via cron

set -euo pipefail

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_ROOT="/var/backups/whmcs"
RETENTION_DAYS=30
S3_BUCKET="s3://company-whmcs-backups"
S3_REGION="us-east-1"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

# Create backup directory
mkdir -p "$BACKUP_ROOT/daily/$DATE"

log "Starting WHMCS backup"

# Database backup
log "Backing up database..."
mysqldump -u root -p"$DB_ROOT_PASSWORD" \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    --hex-blob \
    --master-data=2 \
    --flush-logs \
    whmcs_main | gzip > "$BACKUP_ROOT/daily/$DATE/whmcs_db.sql.gz"

# Verify database backup
if [ $(gzip -dc "$BACKUP_ROOT/daily/$DATE/whmcs_db.sql.gz" | wc -l) -lt 100 ]; then
    log "ERROR: Database backup appears to be empty"
    exit 1
fi

# File backup
log "Backing up files..."
tar -czf "$BACKUP_ROOT/daily/$DATE/whmcs_files.tar.gz" \
    --exclude='cache/*' \
    --exclude='storage/logs/*' \
    --exclude='storage/templates_c/*' \
    --exclude='*.log' \
    /var/www/whmcs/

# Configuration backup
log "Backing up configuration..."
cp /var/www/whmcs/configuration.php "$BACKUP_ROOT/daily/$DATE/configuration.php"

# Generate checksums
log "Generating checksums..."
cd "$BACKUP_ROOT/daily/$DATE"
sha256sum *.gz *.php > checksums.sha256

# Upload to S3
log "Uploading to S3..."
aws s3 sync "$BACKUP_ROOT/daily/$DATE" "$S3_BUCKET/daily/$DATE/" \
    --storage-class STANDARD_IA \
    --region "$S3_REGION"

# Upload to secondary backup location
log "Uploading to secondary location..."
aws s3 sync "$BACKUP_ROOT/daily/$DATE" "s3://company-whmcs-backups-secondary/daily/$DATE/" \
    --region "$S3_REGION"

# Cleanup old local backups
log "Cleaning up old backups..."
find "$BACKUP_ROOT/daily" -type d -mtime +$RETENTION_DAYS -exec rm -rf {} \;

# Verify backup
BACKUP_SIZE=$(du -sh "$BACKUP_ROOT/daily/$DATE" | cut -f1)
log "Backup complete. Size: $BACKUP_SIZE"

# Send notification
if command -v mail &> /dev/null; then
    echo "WHMCS Backup completed successfully on $DATE. Size: $BACKUP_SIZE" | \
    mail -s "WHMCS Backup Success" admin@example.com
fi

log "All backup operations completed"
```

### Step 3: Configure Automated Testing

Verify backups are restorable:

```bash
#!/bin/bash
# /opt/scripts/verify_backup.sh
# Verify backup integrity - run after each backup

set -euo pipefail

BACKUP_DIR="/var/backups/whmcs/daily"
TEST_DB="whmcs_backup_test"
DATE=$(ls -td "$BACKUP_DIR"/*/ | head -1 | xargs basename)

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "Starting backup verification for $DATE"

# Verify checksums
log "Verifying checksums..."
cd "$BACKUP_ROOT/daily/$DATE"
if ! sha256sum -c checksums.sha256; then
    log "ERROR: Checksum verification failed!"
    exit 1
fi
log "Checksums verified"

# Test database restore
log "Testing database restore..."
mysql -u root -p"$DB_ROOT_PASSWORD" -e "DROP DATABASE IF EXISTS $TEST_DB"
mysql -u root -p"$DB_ROOT_PASSWORD" -e "CREATE DATABASE $TEST_DB"
gzip -dc whmcs_db.sql.gz | mysql -u root -p"$DB_ROOT_PASSWORD" $TEST_DB

# Verify critical tables
TABLES=("tblclients" "tblhosting" "tblinvoices" "tblorders" "tblproducts")
for table in "${TABLES[@]}"; do
    COUNT=$(mysql -u root -p"$DB_ROOT_PASSWORD" -N -e "SELECT COUNT(*) FROM $TEST_DB.$table")
    if [ "$COUNT" -eq 0 ] && [ "$table" != "tblproducts" ]; then
        log "WARNING: $table appears to be empty"
    else
        log "$table: $COUNT records"
    fi
done

# Cleanup test database
mysql -u root -p"$DB_ROOT_PASSWORD" -e "DROP DATABASE $TEST_DB"

log "Backup verification complete"
```

### Step 4: Develop Recovery Playbooks

#### Scenario A: Complete Server Failure

```bash
#!/bin/bash
# /opt/scripts/recover_complete_failure.sh

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "Starting complete server recovery"

# 1. Provision new server
# (Use your provisioning system - Terraform, Ansible, etc.)
log "Provisioning new server..."

# 2. Install dependencies
apt-get update && apt-get install -y \
    nginx \
    php8.1-fpm \
    php8.1-mysql \
    php8.1-curl \
    php8.1-gd \
    php8.1-mbstring \
    php8.1-xml \
    php8.1-zip \
    php8.1-redis

# 3. Setup MySQL
apt-get install -y mysql-server

# 4. Restore latest backup
LATEST_BACKUP=$(aws s3 ls s3://company-whmcs-backups/daily/ | tail -n 1 | awk '{print $2}')
aws s3 cp --recursive "s3://company-whmcs-backups/daily/$LATEST_BACKUP" /tmp/whmcs_backup/

# 5. Restore database
mysql -u root -p"$DB_ROOT_PASSWORD" -e "CREATE DATABASE whmcs_main"
gunzip < /tmp/whmcs_backup/whmcs_db.sql.gz | mysql -u root -p"$DB_ROOT_PASSWORD" whmcs_main

# 6. Restore files
tar -xzf /tmp/whmcs_backup/whmcs_files.tar.gz -C /var/www/

# 7. Restore configuration
cp /tmp/whmcs_backup/configuration.php /var/www/whmcs/

# 8. Set permissions
chown -R www-data:www-data /var/www/whmcs
find /var/www/whmcs -type d -exec chmod 755 {} \;
find /var/www/whmcs -type f -exec chmod 644 {} \;

# 9. Clear cache
php /var/www/whmcs/crons/cron.php?a=clearCache

# 10. Verify
curl -f http://localhost || log "ERROR: Health check failed"

log "Recovery complete"
```

#### Scenario B: Database Corruption

```bash
#!/bin/bash
# /opt/scripts/recover_database.sh

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "Starting database recovery"

# Stop WHMCS (maintenance mode)
touch /var/www/whmcs/.maintenance

# Identify corrupted tables
log "Checking for corrupted tables..."
mysqlcheck -u root -p"$DB_ROOT_PASSWORD" --check --all-databases

# Try repair on corrupted tables
log "Attempting repairs..."
mysqlcheck -u root -p"$DB_ROOT_PASSWORD" --repair --use-frm whmcs_main

# If repair fails, restore from backup
LATEST_DB_BACKUP=$(ls -t /var/backups/whmcs/daily/*/whmcs_db.sql.gz | head -1)
log "Restoring from: $LATEST_DB_BACKUP"

mysql -u root -p"$DB_ROOT_PASSWORD" -e "DROP DATABASE IF EXISTS whmcs_main"
mysql -u root -p"$DB_ROOT_PASSWORD" -e "CREATE DATABASE whmcs_main"
gunzip < "$LATEST_DB_BACKUP" | mysql -u root -p"$DB_ROOT_PASSWORD" whmcs_main

# Optimize tables after restore
log "Optimizing tables..."
mysqlcheck -u root -p"$DB_ROOT_PASSWORD" --optimize whmcs_main

# Remove maintenance mode
rm /var/www/whmcs/.maintenance

log "Database recovery complete"
```

#### Scenario C: Security Incident

```php
<?php
// /opt/scripts/security_recovery.php
// Security incident response

class SecurityIncidentRecovery
{
    public function respond($incidentType)
    {
        switch ($incidentType) {
            case 'malware_detected':
                $this->handleMalware();
                break;
            case 'unauthorized_access':
                $this->handleUnauthorizedAccess();
                break;
            case 'data_breach':
                $this->handleDataBreach();
                break;
            default:
                throw new Exception("Unknown incident type");
        }
    }
    
    private function handleMalware()
    {
        // 1. Take immediate backup of infected state (for forensics)
        $this->captureEvidence();
        
        // 2. Take site offline
        rename('/var/www/whmcs', '/var/www/whmcs_compromised_' . date('Ymd_His'));
        
        // 3. Restore from known-good backup
        $this->restoreFromCleanBackup();
        
        // 4. Update all passwords
        $this->forcePasswordReset();
        
        // 5. Review and patch vulnerabilities
        $this->patchVulnerabilities();
        
        // 6. Notify users if necessary
        $this->sendNotifications();
    }
    
    private function captureEvidence()
    {
        $evidenceDir = '/var/forensics/' . date('Ymd_His');
        mkdir($evidenceDir, 0700, true);
        
        // Copy infected files
        exec("cp -r /var/www/whmcs/* $evidenceDir/");
        
        // Copy database
        exec("mysqldump -u root -p whmcs_main > $evidenceDir/database.sql");
        
        // Copy logs
        exec("cp /var/log/nginx/access.log $evidenceDir/");
        exec("cp /var/log/nginx/error.log $evidenceDir/");
        
        // Store evidence location securely
        $this->logEvidenceLocation($evidenceDir);
    }
    
    private function forcePasswordReset()
    {
        // Force password reset for all admin accounts
        Capsule::table('tbladmins')
            ->update([
                'password' => md5('temp_reset_' . time()),
                'lastlogin' => null,
                'failedloginattempts' => 0
            ]);
        
        // Log out all sessions
        Capsule::table('tbladmin_session')->truncate();
        
        // Force client password reset for critical accounts
        Capsule::table('tblclients')
            ->where('status', 'Active')
            ->whereRaw("email IN (SELECT email FROM compromised_emails)")
            ->update(['password' => md5('FORCE_RESET_' . time())]);
    }
}
```

### Step 5: Test Recovery Procedures

Regular testing ensures reliability:

```bash
#!/bin/bash
# /opt/scripts/test_disaster_recovery.sh
# Run quarterly or after any infrastructure change

set -euo pipefail

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "Starting disaster recovery test"

# 1. Create test environment
log "Creating test environment..."
# Spin up isolated test server

# 2. Test file restoration
log "Testing file restoration..."
aws s3 cp s3://company-whmcs-backups/daily/$(date +%Y%m%d)/whmcs_files.tar.gz /tmp/test_backup/
tar -tzf /tmp/test_backup/whmcs_files.tar.gz | head -20

# 3. Test database restoration
log "Testing database restoration..."
mysql -u root -p -e "CREATE DATABASE whmcs_test_restore"
gzip -dc /tmp/test_backup/whmcs_db.sql.gz | mysql -u root -p whmcs_test_restore
mysql -u root -p -e "SELECT COUNT(*) FROM whmcs_test_restore.tblclients"

# 4. Verify WHMCS functionality
log "Testing WHMCS functionality..."
# Point test instance to restored database
# Run smoke tests

# 5. Measure recovery time
log "Measuring recovery time..."
START_TIME=$(date +%s)
# ... perform recovery steps ...
END_TIME=$(date +%s)
RECOVERY_TIME=$((END_TIME - START_TIME))
log "Recovery completed in $RECOVERY_TIME seconds"

# 6. Cleanup test environment
log "Cleaning up test environment..."
mysql -u root -p -e "DROP DATABASE whmcs_test_restore"

# 7. Document results
log "Test completed. Documenting results..."
```

### Step 6: Document Recovery Procedures

Create runbooks for each scenario:

```markdown
## WHMCS Disaster Recovery Runbook

### Scenario: Complete Server Loss

**Estimated Recovery Time:** 2-4 hours
**RTO:** 4 hours
**RPO:** 24 hours

#### Immediate Actions (First 15 minutes)
1. Confirm server failure
2. Activate incident response team
3. Notify stakeholders
4. Initiate recovery procedure

#### Short-term Actions (15 minutes - 2 hours)
1. Provision replacement server
2. Restore from latest backup
3. Verify data integrity
4. Test basic functionality

#### Recovery Steps
[Detailed step-by-step instructions]

#### Verification Steps
1. Admin login functional
2. Client portal accessible
3. Invoices render correctly
4. Orders process correctly
5. Emails send successfully

### Contact Information
- **On-call Engineer:** [Contact]
- **WHMCS Support:** support@whmcs.com
- **Hosting Provider:** support@hostingprovider.com

### Post-Incident
- Conduct post-mortem
- Update procedures if needed
- Schedule additional tests
```

## Verification Checklist

- [ ] All backup jobs running on schedule
- [ ] Backup verification tests passing
- [ ] Backup copies stored in multiple locations
- [ ] Recovery documentation current and accessible
- [ ] Team trained on recovery procedures
- [ ] Recovery time objectives documented
- [ ] Recovery point objectives documented
- [ ] Runbooks tested quarterly
- [ ] Contact information up to date
- [ ] Off-site backup accessible
- [ ] Security incident response plan in place
- [ ] Stakeholder notification templates ready

## Related Skills and Documentation

- [WHMCS Database Migration](whmcs-database-migration-workflow.md)
- [WHMCS Deployment Best Practices](whmcs-deployment-best-practices.md)
- [WHMCS Security Audit](whmcs-security-audit.md)
- WHMCS System Requirements: https://docs.whmcs.com/System_Requirements
- WHMCS Backup Guide: https://docs.whmcs.com/Backup

## Notes

- Test backups regularly - untested backups are not reliable backups
- Keep at least 3 backup copies: daily, weekly, monthly
- Store backups in multiple geographic locations
- Document all manual steps required during recovery
- Include recovery time estimates in SLAs
