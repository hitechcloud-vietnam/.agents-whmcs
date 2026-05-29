# WHMCS Temporary File Cleanup Workflow

## Purpose
Clean up temporary files and free disk space

## Prerequisites
- SSH access
- Root/sudo access

## Step 1: Identify Temp Locations

```
/var/www/whmcs/templates_c/
/var/www/whmcs/cache/
/var/www/whmcs/tmp/
/var/www/whmcs/downloads/
/tmp/
/var/tmp/
```

## Step 2: Check Temp Directory Sizes

```bash
du -sh /var/www/whmcs/templates_c/
du -sh /var/www/whmcs/cache/
du -sh /var/www/whmcs/tmp/
du -sh /tmp/
du -sh /var/tmp/
```

## Step 3: Clear Template Cache

```bash
rm -rf /var/www/whmcs/templates_c/smarty_cache/*
rm -rf /var/www/whmcs/templates_c/smarty_compile/*

ls -la /var/www/whmcs/templates_c/
```

## Step 4: Clear System Cache

```bash
rm -rf /var/www/whmcs/cache/*
rm -rf /var/www/whmcs/cache/assets/*

mkdir -p /var/www/whmcs/cache
chown www-data:www-data /var/www/whmcs/cache
chmod 755 /var/www/whmcs/cache
```

## Step 5: Clear Temp Directory

```bash
rm -rf /var/www/whmcs/tmp/*
rm -rf /tmp/*
rm -rf /var/tmp/*
```

## Step 6: Clear Old Downloads

```bash
# Find old downloads
find /var/www/whmcs/downloads -type f -mtime +90 -ls

# Delete old downloads
find /var/www/whmcs/downloads -type f -mtime +90 -delete

# Clear empty directories
find /var/www/whmcs/downloads -type d -empty -delete
```

## Step 7: Clear Old Attachments

```bash
# Find old attachments
find /var/www/whmcs/attachments -type f -mtime +365 -ls

# Delete old attachments (be careful!)
find /var/www/whmcs/attachments -type f -mtime +365 -delete
```

## Step 8: Clear Old Logs

```bash
# Archive and clear old logs
mkdir -p /var/www/whmcs/logs/archive/$(date +%Y%m)

for log in /var/www/whmcs/logs/*.log; do
    if [ -s "$log" ]; then
        gzip "$log"
        mv "${log}.gz" /var/www/whmcs/logs/archive/$(date +%Y%m)/
        touch "$log"
        chown www-data:www-data "$log"
    fi
done

# Delete very old archives
find /var/www/whmcs/logs/archive -type f -mtime +180 -delete
```

## Step 9: Clear System Temp

```bash
# Clear system temp directories
rm -rf /tmp/*
rm -rf /var/tmp/*

# Set proper permissions
chmod 1777 /tmp
chmod 1777 /var/tmp
```

## Step 10: Clear PHP Session Files

```bash
# Find session directory
php -i | grep session.save_path

# Clear old sessions
find /var/lib/php/sessions -type f -mmin +60 -delete
```

## Step 11: Clear OPcache

```bash
# If using PHP-FPM
systemctl restart php8.1-fpm

# Or via command line
php -r "if(function_exists('opcache_reset')){opcache_reset();}"
```

## Step 12: Create Cleanup Script

```bash
nano /usr/local/bin/whmcs-cleanup.sh
```

```bash
#!/bin/bash
# WHMCS Temporary Files Cleanup Script

echo "Starting WHMCS cleanup..."

# Clear caches
rm -rf /var/www/whmcs/templates_c/*
rm -rf /var/www/whmcs/cache/*
echo "Caches cleared"

# Clear temp files
rm -rf /var/www/whmcs/tmp/*
rm -rf /tmp/*
rm -rf /var/tmp/*
echo "Temp files cleared"

# Clear old downloads (90+ days)
find /var/www/whmcs/downloads -type f -mtime +90 -delete
echo "Old downloads cleared"

# Clear old attachments (1+ year)
find /var/www/whmcs/attachments -type f -mtime +365 -delete
echo "Old attachments cleared"

# Clear PHP session files
find /var/lib/php/sessions -type f -mmin +60 -delete
echo "PHP sessions cleared"

# Restart PHP-FPM
systemctl restart php8.1-fpm
echo "PHP-FPM restarted"

echo "Cleanup complete"
```

```bash
chmod +x /usr/local/bin/whmcs-cleanup.sh
```

## Step 13: Schedule Cleanup

```bash
crontab -e
```

Add:
```cron
# Weekly cleanup - Sunday at 4 AM
0 4 * * 0 /usr/local/bin/whmcs-cleanup.sh >> /var/log/whmcs-cleanup.log 2>&1
```

## Step 14: Verify Disk Space

```bash
# Check available space
df -h

# Check WHMCS directory size
du -sh /var/www/whmcs/
```

## Cleanup Checklist

- [ ] Temp locations identified
- [ ] Directory sizes checked
- [ ] Template cache cleared
- [ ] System cache cleared
- [ ] Temp directory cleared
- [ ] Old downloads removed
- [ ] Old attachments removed
- [ ] Old logs archived
- [ ] System temp cleared
- [ ] PHP sessions cleared
- [ ] OPcache cleared
- [ ] Cleanup script created
- [ ] Cleanup scheduled
- [ ] Disk space verified
