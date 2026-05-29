# WHMCS Weekly Maintenance Workflow

## Purpose
Complete weekly WHMCS maintenance and optimization

## Prerequisites
- WHMCS installed
- SSH access
- Admin access

## Step 1: Perform Full Backup

```bash
# Database backup
mysqldump -u root -p whmcs_db > /backup/whmcs_db_$(date +%Y%m%d).sql

# Compress backup
gzip /backup/whmcs_db_$(date +%Y%m%d).sql

# Files backup
tar -czf /backup/whmcs_files_$(date +%Y%m%d).tar.gz /var/www/whmcs --exclude=/var/www/whmcs/templates_c --exclude=/var/www/whmcs/cache

# Verify backup
ls -lh /backup/whmcs*
```

## Step 2: Update WHMCS

Navigate to: Utilities > System > Check for Updates

If update available:
1. Create backup first
2. Review changelog
3. Apply update
4. Test functionality

## Step 3: Update Modules and Addons

Navigate to: Setup > Addon Modules

1. Check for module updates
2. Update any outdated modules
3. Test after update

## Step 4: Review System Logs

```bash
# Apache error log
tail -100 /var/log/apache2/error.log

# WHMCS error log
tail -100 /var/www/whmcs/logs/error.log

# Cron log
tail -100 /var/www/whmcs/logs/cron.log
```

Look for:
- PHP errors
- Database connection issues
- Cron failures

## Step 5: Optimize Database

```bash
# Optimize all tables
mysqlcheck -u root -p --optimize --all-databases

# Analyze tables
mysqlcheck -u root -p --analyze whmcs_db

# Repair if needed
mysqlcheck -u root -p --repair whmcs_db
```

## Step 6: Review Client Reports

Navigate to: Reports > Clients

Review:
- New clients this week
- Cancelled clients
- Upgrades/downgrades
- Payment issues

## Step 7: Review Financial Reports

Navigate to: Reports > Revenue

Generate weekly report:
- Total revenue
- Outstanding amounts
- Refunds issued
- Failed payments

## Step 8: Check Domain Expirations

Navigate to: Clients > Domains

Review:
- Domains expiring in 30 days
- Domains expiring in 7 days
- Expired domains status

## Step 9: Review Support Metrics

Navigate to: Reports > Support

Review:
- Tickets created
- Average response time
- Resolution time
- Client satisfaction

## Step 10: Clean Up Storage

```bash
# Clean old backups (keep last 4 weeks)
find /backup -name "whmcs*.sql.gz" -mtime +28 -delete
find /backup -name "whmcs*.tar.gz" -mtime +28 -delete

# Clean temp files
rm -rf /var/www/whmcs/templates_c/*
rm -rf /var/www/whmcs/cache/*

# Clean old download files
find /var/www/whmcs/downloads -type f -mtime +90 -delete
```

## Step 11: Review Security

1. Check admin access logs
2. Review failed login patterns
3. Verify SSL certificates
4. Check firewall rules
5. Review API access

## Step 12: Performance Review

Navigate to: Utilities > System Health Status

Check:
- Response times
- Database queries
- Memory usage
- Server load

## Step 13: Update Documentation

Update any internal documentation with:
- System changes
- New configurations
- Known issues
- Solutions implemented

## Weekly Task Checklist

- [ ] Full backup performed
- [ ] WHMCS updated
- [ ] Modules updated
- [ ] System logs reviewed
- [ ] Database optimized
- [ ] Client reports reviewed
- [ ] Financial reports reviewed
- [ ] Domain expirations checked
- [ ] Support metrics reviewed
- [ ] Storage cleaned
- [ ] Security reviewed
- [ ] Performance reviewed
- [ ] Documentation updated
