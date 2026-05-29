# WHMCS Monthly Maintenance Workflow

## Purpose
Complete comprehensive monthly WHMCS maintenance

## Prerequisites
- WHMCS installed
- SSH access
- Root/admin access

## Step 1: Full System Backup

```bash
# Create monthly backup directory
mkdir -p /backup/monthly/$(date +%Y%m)

# Database backup
mysqldump -u root -p whmcs_db | gzip > /backup/monthly/$(date +%Y%m)/whmcs_db_$(date +%Y%m%d).sql.gz

# Files backup
tar -czf /backup/monthly/$(date +%Y%m)/whmcs_files_$(date +%Y%m%d).tar.gz /var/www/whmcs --exclude=/var/www/whmcs/templates_c --exclude=/var/www/whmcs/cache

# Verify backup integrity
tar -tzf /backup/monthly/$(date +%Y%m)/whmcs_files_$(date +%Y%m%d).tar.gz | head
gunzip -t /backup/monthly/$(date +%Y%m)/whmcs_db_$(date +%Y%m%d).sql.gz

# Upload to off-site storage
# rsync -avz /backup/monthly/ user@backupserver:/backups/whmcs/
```

## Step 2: Update All Software

### WHMCS Core
Navigate to: Utilities > System > Check for Updates
Apply any available updates.

### Server Software
```bash
# Ubuntu/Debian
apt update && apt upgrade -y

# CentOS/AlmaLinux
dnf update -y

# Restart services
systemctl restart apache2
systemctl restart mysql
```

### PHP Updates
```bash
# Check PHP version
php -v

# Update if needed
apt install php8.1 php8.1-*
# OR
dnf install php-8.1-*
```

## Step 3: Database Maintenance

```bash
# Full database check
mysqlcheck -u root -p --check --all-databases

# Optimize all tables
mysqlcheck -u root -p --optimize --all-databases

# Repair if issues found
mysqlcheck -u root -p --repair whmcs_db

# Analyze for query optimization
mysqlcheck -u root -p --analyze --all-databases
```

## Step 4: Review Monthly Reports

### Revenue Report
Navigate to: Reports > Revenue > Monthly Revenue

Generate and save report for:
- MRR (Monthly Recurring Revenue)
- New revenue
- Churned revenue
- Net revenue change

### Client Report
Navigate to: Reports > Clients

Review:
- New clients
- Churned clients
- Client retention rate

### Support Report
Navigate to: Reports > Support

Metrics:
- Total tickets
- Average resolution time
- CSAT scores

## Step 5: Review Storage Usage

```bash
# Overall disk usage
df -h

# Directory sizes
du -sh /var/www/whmcs/*
du -sh /var/www/whmcs/downloads/*
du -sh /var/www/whmcs/attachments/*
du -sh /var/www/whmcs/logs/*

# MySQL data size
mysql -u root -p -e "SELECT table_schema AS 'Database', table_name AS 'Table', ROUND(data_length / 1024 / 1024, 2) AS 'Data (MB)', ROUND(index_length / 1024 / 1024, 2) AS 'Index (MB)' FROM information_schema.tables WHERE table_schema = 'whmcs_db' ORDER BY (data_length + index_length) DESC LIMIT 20;"
```

## Step 6: Security Audit

### Review Admin Access
Navigate to: Configuration > System Settings > Admin Activity Log

Review:
- Admin login patterns
- Permission changes
- Bulk operations

### Review Failed Attempts
```bash
# Check failed logins
grep "Failed login" /var/www/whmcs/logs/activity.log | tail -100

# Check Apache auth log
tail -100 /var/log/apache2/error.log | grep -i auth
```

### SSL Certificate Review
```bash
# Check certificate expiration
openssl s_client -connect yourdomain.com:443 -servername yourdomain.com 2>/dev/null | openssl x509 -noout -dates
```

## Step 7: Performance Analysis

### Review Server Load
```bash
# Average load
uptime

# Memory usage
free -h

# Apache/Nginx status
systemctl status apache2
```

### Review WHMCS Performance
Navigate to: Utilities > System Health Status

Check:
- Response times
- Database queries
- Memory usage

## Step 8: Review Outstanding Invoices

Navigate to: Billing > Invoices

Review:
- Overdue invoices
- Pending invoices
- Refund requests

Send reminders for overdue invoices.

## Step 9: Review Domain Portfolio

Navigate to: Clients > Domains

Review:
- Expiring domains
- Renewal revenue
- Domain transfers out

## Step 10: Clean Up Historical Data

```bash
# Archive old logs
mkdir -p /var/www/whmcs/logs/archive/$(date +%Y%m)
mv /var/www/whmcs/logs/*.log /var/www/whmcs/logs/archive/$(date +%Y%m)/

# Clean attachments older than 1 year
find /var/www/whmcs/attachments -type f -mtime +365 -delete

# Clean downloads older than 6 months
find /var/www/whmcs/downloads -type f -mtime +180 -delete

# Vacuum MySQL tables
mysql -u root -p -e "OPTIMIZE TABLE whmcs_db.tblactivitylog;"
```

## Step 11: Review Integrations

Check all external integrations:
- Payment gateways
- Domain registrars
- Server modules
- Third-party APIs

Verify all are functioning correctly.

## Step 12: Disaster Recovery Test

```bash
# Test backup restoration
mkdir /tmp/test_restore
cd /tmp/test_restore
gunzip < /backup/monthly/$(date +%Y%m)/whmcs_db_*.sql.gz | mysql -u root -p test_restore
```

## Step 13: Generate Monthly Summary

Create internal report with:
- Total revenue
- New clients
- Churned clients
- Support metrics
- System issues
- Recommendations

## Monthly Task Checklist

- [ ] Full backup performed
- [ ] Backup verified and stored
- [ ] All software updated
- [ ] Database optimized
- [ ] Monthly reports generated
- [ ] Storage analyzed
- [ ] Security audit completed
- [ ] Performance reviewed
- [ ] Outstanding invoices processed
- [ ] Domain portfolio reviewed
- [ ] Historical data cleaned
- [ ] Integrations verified
- [ ] Disaster recovery tested
- [ ] Monthly summary generated
