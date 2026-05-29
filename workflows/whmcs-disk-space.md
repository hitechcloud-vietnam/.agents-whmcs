# WHMCS Disk Space Management Workflow

## Purpose
Manage and optimize disk space usage

## Prerequisites
- SSH access
- Root access

## Step 1: Check Disk Usage

```bash
df -h
```

Check all filesystems.

## Step 2: Check WHMCS Directory Size

```bash
du -sh /var/www/whmcs/
```

## Step 3: Find Largest Directories

```bash
du -h /var/www/whmcs/ | sort -rh | head -20
```

## Step 4: Check Database Size

```bash
mysql -u root -p -e "
SELECT 
    table_schema AS 'Database',
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)'
FROM information_schema.tables 
GROUP BY table_schema;"
```

## Step 5: Analyze Downloads Directory

```bash
du -sh /var/www/whmcs/downloads/
find /var/www/whmcs/downloads -type f -mtime +90 -ls
```

## Step 6: Analyze Attachments

```bash
du -sh /var/www/whmcs/attachments/
find /var/www/whmcs/attachments -type f -mtime +365 -ls
```

## Step 7: Clean Cache Directories

```bash
# Template cache
du -sh /var/www/whmcs/templates_c/
rm -rf /var/www/whmcs/templates_c/*

# System cache
du -sh /var/www/whmcs/cache/
rm -rf /var/www/whmcs/cache/*
```

## Step 8: Clean Log Files

```bash
# Archive and clear logs
mkdir -p /var/www/whmcs/logs/archive/$(date +%Y%m)
mv /var/www/whmcs/logs/*.log /var/www/whmcs/logs/archive/$(date +%Y%m)/ 2>/dev/null

# Restart services to create new logs
systemctl restart apache2
```

## Step 9: Clean System Temp

```bash
du -sh /tmp/
rm -rf /tmp/*
du -sh /var/tmp/
rm -rf /var/tmp/*
```

## Step 10: Remove Old Backups

```bash
# Find old backups
find /backup -type f -mtime +30 -ls

# Remove backups older than 30 days
find /backup -type f -mtime +30 -delete
```

## Step 11: Optimize Database

```bash
mysqlcheck -u root -p --optimize whmcs_db
```

## Step 12: Set Up Monitoring

Create script at `/usr/local/bin/whmcs-disk-monitor.sh`:

```bash
#!/bin/bash
THRESHOLD=80
USAGE=$(df /var/www/whmcs | tail -1 | awk '{print $5}' | sed 's/%//')

if [ $USAGE -gt $THRESHOLD ]; then
    echo "WARNING: Disk usage is at ${USAGE}%" | mail -s "WHMCS Disk Alert" admin@domain.com
fi
```

```bash
chmod +x /usr/local/bin/whmcs-disk-monitor.sh
```

Add to cron:
```cron
# Check disk space daily
0 6 * * * /usr/local/bin/whmcs-disk-monitor.sh
```

## Disk Space Checklist

- [ ] Disk usage checked
- [ ] WHMCS directory analyzed
- [ ] Largest directories found
- [ ] Database size checked
- [ ] Downloads analyzed
- [ ] Attachments analyzed
- [ ] Cache cleaned
- [ ] Logs archived
- [ ] System temp cleaned
- [ ] Old backups removed
- [ ] Database optimized
- [ ] Monitoring set up
