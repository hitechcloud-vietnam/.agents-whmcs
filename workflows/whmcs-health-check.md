# WHMCS System Health Check Workflow

## Purpose
Perform comprehensive health check of WHMCS installation

## Prerequisites
- WHMCS installed
- Admin access
- SSH access

## Step 1: Run WHMCS Health Check

Navigate to: Utilities > System Health Status

Review all checks:
- [ ] System requirements
- [ ] Cron status
- [ ] Database connection
- [ ] License status
- [ ] File permissions
- [ ] PHP requirements

## Step 2: Check PHP Configuration

```bash
php -i | grep -E "memory_limit|upload_max_filesize|post_max_size|max_execution_time"
```

Required settings:
```ini
memory_limit = 256M
upload_max_filesize = 10M
post_max_size = 64M
max_execution_time = 300
```

## Step 3: Verify Database Connection

```bash
mysql -u whmcs_user -p -e "SELECT COUNT(*) FROM whmcs_db.tblclients;"
```

Check:
- Connection successful
- Proper permissions
- UTF8MB4 encoding

## Step 4: Check Cron Status

Navigate to: Utilities > System Health Status

Look for:
- "Last Cron Run" should be recent
- Cron running without errors

Manual test:
```bash
/usr/bin/php /var/www/whmcs/crons/cron.php
```

## Step 5: Verify License

Navigate to: Utilities > System Health Status

Check:
- License active
- License matches domain
- No license errors

## Step 6: Check Disk Space

```bash
df -h
```

Ensure:
- Root filesystem > 20% free
- /var/www > 10% free
- /tmp > adequate space

## Step 7: Check Server Resources

```bash
# Memory
free -h

# Load average
uptime

# CPU
top -n 1 | head -5
```

## Step 8: Review Error Logs

```bash
# WHMCS error log
tail -50 /var/www/whmcs/logs/error.log

# Apache error log
tail -50 /var/log/apache2/error.log

# MySQL error log
tail -50 /var/log/mysql/error.log
```

Look for:
- PHP errors
- Database errors
- Permission errors

## Step 9: Check Services Status

```bash
# Web server
systemctl status apache2
# OR
systemctl status nginx

# Database
systemctl status mysql
# OR
systemctl status mariadb

# PHP-FPM
systemctl status php8.1-fpm
```

## Step 10: Verify SSL Certificate

```bash
openssl s_client -connect yourdomain.com:443 2>/dev/null | openssl x509 -noout -dates
```

Check:
- Certificate not expired
- Valid for current domain
- Proper chain

## Step 11: Check File Permissions

```bash
ls -la /var/www/whmcs/configuration.php
# Should be: -r-------- 1 www-data www-data

ls -la /var/www/whmcs/admin
# Should be: drwxr-xr-x 1 www-data www-data
```

## Step 12: Verify ionCube Loader

```bash
php -v
```

Should show ionCube in version string.

## Step 13: Check Required Extensions

```bash
php -m | grep -E "pdo_mysql|gd|curl|xml|zip|mbstring|openssl|ionCube"
```

Required:
- pdo_mysql or mysqli
- gd
- curl
- xml
- zip
- mbstring
- openssl
- ionCube Loader

## Step 14: Test Email Sending

Navigate to: Utilities > System > Send Test Email

Send test to verify email works.

## Step 15: Test Payment Gateway

Place test order with test payment to verify gateway works.

## Step 16: Check Backup Status

```bash
ls -la /backup/whmcs/ | tail -5
```

Verify recent backups exist.

## Step 17: Review Integration Status

Check:
- Payment gateways configured
- Domain registrars connected
- Server modules working
- External APIs responding

## Health Check Report

Generate report with:
- All checks passed/failed
- Issues identified
- Actions required
- Recommendations

## Health Check Checklist

- [ ] WHMCS health check run
- [ ] PHP configuration verified
- [ ] Database connection verified
- [ ] Cron status checked
- [ ] License verified
- [ ] Disk space checked
- [ ] Server resources checked
- [ ] Error logs reviewed
- [ ] Services status verified
- [ ] SSL certificate verified
- [ ] File permissions verified
- [ ] ionCube verified
- [ ] Extensions verified
- [ ] Email tested
- [ ] Payment tested
- [ ] Backup status verified
- [ ] Integrations verified
