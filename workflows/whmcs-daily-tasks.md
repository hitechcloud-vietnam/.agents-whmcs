# WHMCS Daily Maintenance Tasks Workflow

## Purpose
Complete daily WHMCS maintenance procedures

## Prerequisites
- WHMCS installed and running
- Admin access

## Step 1: Check System Health

Navigate to: Utilities > System Health Status

Verify:
- [ ] System requirements met
- [ ] Cron running correctly
- [ ] Database connection stable
- [ ] License active
- [ ] No critical errors

## Step 2: Review Activity Log

Navigate to: Utilities > Logs > Activity Log

1. Review last 24 hours
2. Look for:
   - Failed login attempts
   - Error messages
   - Unusual activity

## Step 3: Check Pending Orders

Navigate to: Orders > Pending Orders

1. Review pending orders
2. Process or cancel as needed
3. Check for fraudulent orders

## Step 4: Review Support Tickets

Navigate to: Support > Support Tickets

Check:
- [ ] Open tickets (older than 24h)
- [ ] Awaiting reply tickets
- [ ] Escalated tickets

## Step 5: Check Failed Services

Navigate to: Utilities > Queue

1. Review failed automation tasks
2. Retry or investigate failures
3. Document recurring issues

## Step 6: Verify Backups

```bash
# Check backup directory
ls -la /backup/whmcs/

# Verify recent backup
ls -la /backup/whmcs/*.sql.gz | head -5
```

## Step 7: Review Disk Space

```bash
df -h /var/www/whmcs
du -sh /var/www/whmcs/downloads
du -sh /var/www/whmcs/logs
```

Clean up if needed:
```bash
# Clean old logs
find /var/www/whmcs/logs -name "*.log" -mtime +30 -delete

# Clean temp files
rm -rf /var/www/whmcs/templates_c/smarty_cache/*
```

## Step 8: Check Server Resources

```bash
# CPU and memory
top -n 1 | head -10

# Apache/Nginx status
systemctl status apache2
# OR
systemctl status nginx

# MySQL status
systemctl status mysql
```

## Step 9: Review Revenue

Navigate to: Reports > Revenue

Check:
- Daily revenue
- Outstanding invoices
- Failed payments

## Step 10: Monitor Security

1. Check for unauthorized access
2. Review failed login logins
3. Verify SSL certificate validity
4. Check firewall logs

## Step 11: Database Maintenance

```bash
# Check database size
mysql -u root -p -e "SELECT table_schema 'Database', ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) 'Size (MB)' FROM information_schema.tables GROUP BY table_schema;"

# Check for table issues
mysqlcheck -u root -p --check whmcs_db
```

## Step 12: Clear Cache (if needed)

Navigate to: Utilities > System > Clear Cache

Or via SSH:
```bash
rm -rf /var/www/whmcs/templates_c/*
rm -rf /var/www/whmcs/cache/*
```

## Daily Task Checklist

- [ ] System health checked
- [ ] Activity log reviewed
- [ ] Pending orders processed
- [ ] Support tickets reviewed
- [ ] Failed services checked
- [ ] Backups verified
- [ ] Disk space checked
- [ ] Server resources reviewed
- [ ] Revenue reviewed
- [ ] Security monitored
- [ ] Database maintenance done
- [ ] Cache cleared (if needed)
