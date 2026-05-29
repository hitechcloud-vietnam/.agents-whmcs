# WHMCS Permissions Setup Workflow

## Purpose
Configure correct file and directory permissions for WHMCS security

## Prerequisites
- SSH access to server
- WHMCS installed
- Root or www-data access

## Step 1: Understanding WHMCS Permissions

### Key Directories
| Directory | Purpose | Permission |
|-----------|---------|------------|
| / | WHMCS root | 755 |
| configuration.php | Config file | 400 |
| /admin | Admin area | 755 |
| /templates_c | Smarty cache | 755 |
| /downloads | Downloads | 755 |
| /attachments | File uploads | 755 |
| /cache | Cache files | 755 |

## Step 2: Set Ownership

```bash
cd /var/www/whmcs
chown -R www-data:www-data .
```

For Plesk:
```bash
chown -R psacln:psaserv .
```

For cPanel:
```bash
chown -R username:username .
```

## Step 3: Set Base Directory Permissions

```bash
# Set all directories to 755
find . -type d -exec chmod 755 {} \;

# Set all files to 644
find . -type f -exec chmod 644 {} \;
```

## Step 4: Secure configuration.php

```bash
chmod 400 configuration.php
chown www-data:www-data configuration.php
```

Verify:
```bash
ls -la configuration.php
# Should show: -r-------- 1 www-data www-data
```

## Step 5: Set Admin Directory Permissions

```bash
cd /var/www/whmcs/admin
chmod 755 .
chmod 755 ../admin

# Protect sensitive admin files
chmod 640 ../admin/settings.php 2>/dev/null || true
chmod 640 ../admin/license.php 2>/dev/null || true
```

## Step 6: Set Templates Cache Permissions

```bash
cd /var/www/whmcs
chmod 755 templates_c
chmod 644 templates_c/index.html 2>/dev/null || true

# Clear and rebuild templates_c
rm -f templates_c/*.php
rm -rf templates_c/smarty_cache
rm -rf templates_c/smarty_compile
```

## Step 7: Set Attachments Permissions

```bash
cd /var/www/whmcs
chmod 755 attachments
chmod 644 attachments/index.html 2>/dev/null || true
```

## Step 8: Set Downloads Permissions

```bash
cd /var/www/whmcs
chmod 755 downloads
chmod 644 downloads/index.html 2>/dev/null || true
```

## Step 9: Set Cache Permissions

```bash
cd /var/www/whmcs
chmod 755 cache
chmod 644 cache/index.html 2>/dev/null || true
```

## Step 10: Set Logs Directory Permissions

```bash
cd /var/www/whmcs
chmod 755 logs
chmod 640 logs/*.log 2>/dev/null || true
```

## Step 11: Secure .htaccess Files

```bash
cd /var/www/whmcs
# Ensure .htaccess is readable but protected
chmod 644 .htaccess
```

## Step 12: Protect Sensitive Files

```bash
cd /var/www/whmcs
# Protect configuration files
chmod 400 configuration.php
chmod 404 .htaccess

# Protect admin includes
chmod 640 admin/includes/*.php 2>/dev/null || true
```

## Step 13: Create Permission Script

```bash
nano /var/www/whmcs/fix-permissions.sh
```

```bash
#!/bin/bash
# WHMCS Permission Fix Script

echo "Setting WHMCS permissions..."

cd /var/www/whmcs

# Set ownership
chown -R www-data:www-data .

# Set directory permissions
find . -type d -exec chmod 755 {} \;

# Set file permissions
find . -type f -exec chmod 644 {} \;

# Special permissions
chmod 400 configuration.php
chmod 755 templates_c
chmod 755 attachments
chmod 755 downloads
chmod 755 cache
chmod 755 logs
chmod 755 admin

echo "Permissions fixed!"
```

```bash
chmod +x /var/www/whmcs/fix-permissions.sh
```

## Step 14: Verify Permissions

```bash
cd /var/www/whmcs
ls -la
```

Expected output:
```
drwxr-xr-x  4 www-data www-data  4096 May 29 12:00 .
drwxr-xr-x 11 www-data www-data  4096 May 29 12:00 admin
-r--------  1 www-data www-data  4096 May 29 12:00 configuration.php
drwxr-xr-x  5 www-data www-data  4096 May 29 12:00 includes
drwxr-xr-x  3 www-data www-data  4096 May 29 12:00 templates
drwxr-xr-x  7 www-data www-data  4096 May 29 12:00 templates_c
...
```

## Step 15: Test File Operations

### Test Template Cache
Navigate to: Utilities > System Health Status

Click "Clear Template Cache"

### Test File Upload
Navigate to: Setup > Email Templates > Manage Attachments

Upload a test file.

## Permission Reference Table

| Path | Permission | Owner | Notes |
|------|------------|-------|-------|
| / | 755 | www-data | Root directory |
| configuration.php | 400 | www-data | Critical security |
| /admin | 755 | www-data | Admin area |
| /templates_c | 755 | www-data | Must be writable |
| /attachments | 755 | www-data | Must be writable |
| /downloads | 755 | www-data | Must be writable |
| /cache | 755 | www-data | Must be writable |
| /logs | 755 | www-data | Must be writable |
| *.php | 644 | www-data | All PHP files |
| *.html | 644 | www-data | All HTML files |
| .htaccess | 404 | www-data | Apache config |

## Troubleshooting

### 500 Internal Server Error
```bash
# Check error log
tail -f /var/log/apache2/error.log

# Usually permission issue - verify
ls -la /var/www/whmcs
```

### Template Cache Not Writing
```bash
# Fix templates_c
chmod 755 /var/www/whmcs/templates_c
chown -R www-data:www-data /var/www/whmcs/templates_c
rm -rf /var/www/whmcs/templates_c/*
```

### Upload Failed
```bash
# Fix attachments
chmod 755 /var/www/whmcs/attachments
chown -R www-data:www-data /var/www/whmcs/attachments
```

## Verification Checklist

- [ ] Ownership set to www-data
- [ ] Directories set to 755
- [ ] Files set to 644
- [ ] configuration.php secured (400)
- [ ] templates_c writable
- [ ] attachments writable
- [ ] downloads writable
- [ ] cache writable
- [ ] logs writable
- [ ] .htaccess protected
