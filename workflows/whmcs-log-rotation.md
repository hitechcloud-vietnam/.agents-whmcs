# WHMCS Log Rotation Workflow

## Purpose
Configure and manage log rotation for WHMCS logs

## Prerequisites
- SSH access
- Root/sudo access

## Step 1: Identify WHMCS Logs

```
/var/www/whmcs/logs/
  - activity.log
  - adminlog.log
  - error.log
  - module.log
  - payments.log
  - api.log
```

## Step 2: Configure Logrotate

```bash
nano /etc/logrotate.d/whmcs
```

Add configuration:

```bash
/var/www/whmcs/logs/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 0644 www-data www-data
    sharedscripts
    postrotate
        systemctl reload apache2 > /dev/null 2>&1 || true
    endscript
}
```

## Step 3: Test Logrotate Configuration

```bash
# Test configuration
logrotate -d /etc/logrotate.d/whmcs

# Force rotation
logrotate -f /etc/logrotate.d/whmcs
```

## Step 4: Verify Rotation

```bash
# Check logs
ls -la /var/www/whmcs/logs/

# Check rotated files
ls -la /var/www/whmcs/logs/*.gz
```

## Step 5: Configure MySQL Log Rotation

```bash
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Add:
```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
```

Configure MySQL log rotation:
```bash
nano /etc/logrotate.d/mysql
```

```bash
/var/log/mysql/*.log {
    daily
    rotate 7
    missingok
    compress
    create 600 mysql mysql
    postrotate
        mysqladmin flush-logs
    endscript
}
```

## Step 6: Apache Log Rotation

```bash
nano /etc/logrotate.d/apache2
```

```bash
/var/log/apache2/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 0640 root adm
    sharedscripts
    postrotate
        /usr/sbin/apache2ctl graceful > /dev/null 2>&1
    endscript
}
```

## Step 7: System Cron Log

```bash
nano /etc/logrotate.d/syslog
```

```bash
/var/log/syslog {
    daily
    rotate 14
    missingok
    notifempty
    compress
    sharedscripts
    postrotate
        /usr/lib/rsyslog/rsyslog-rotate
    endscript
}
```

## Step 8: Custom Log Management Script

Create `/usr/local/bin/whmcs-manage-logs.sh`:

```bash
#!/bin/bash
# WHMCS Log Management Script

LOG_DIR="/var/www/whmcs/logs"
DAYS_TO_KEEP=90

echo "Managing WHMCS logs..."

# Archive old logs
cd $LOG_DIR
for log in *.log; do
    if [ -f "$log" ] && [ -s "$log" ]; then
        # Archive
        gzip "$log"
        
        # Create new empty log
        touch "$log"
        chown www-data:www-data "$log"
        chmod 644 "$log"
    fi
done

# Clean old archives
find $LOG_DIR -name "*.gz" -mtime +$DAYS_TO_KEEP -delete

echo "Log management complete"
```

```bash
chmod +x /usr/local/bin/whmcs-manage-logs.sh
```

## Step 9: Schedule Log Management

```bash
crontab -e
```

Add:
```cron
# WHMCS Log Management - Daily at midnight
0 0 * * * /usr/local/bin/whmcs-manage-logs.sh
```

## Step 10: Monitor Log Space Usage

```bash
# Check log directory size
du -sh /var/www/whmcs/logs/

# Check individual log sizes
du -h /var/www/whmcs/logs/*.log

# Find largest logs
find /var/www/whmcs/logs -type f -exec ls -lh {} \; | sort -k5 -h
```

## Log Rotation Checklist

- [ ] WHMCS logs identified
- [ ] Logrotate configured
- [ ] Configuration tested
- [ ] Rotation verified
- [ ] MySQL logs configured
- [ ] Apache logs configured
- [ ] Custom script created
- [ ] Scheduled rotation enabled
- [ ] Space usage monitored
