# WHMCS Cache Clearing Workflow

## Purpose
Clear various caches in WHMCS for troubleshooting and optimization

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Clear Template Cache

### Via Admin Panel
Navigate to: Utilities > System > Clear Cache

Click "Clear Template Cache"

### Via SSH
```bash
rm -rf /var/www/whmcs/templates_c/*
rm -rf /var/www/whmcs/cache/smarty_cache/*
rm -rf /var/www/whmcs/cache/smarty_compile/*
```

## Step 2: Clear System Cache

### Via Admin Panel
Navigate to: Utilities > System > Clear Cache

Select "Clear System Cache"

### Via SSH
```bash
rm -rf /var/www/whmcs/cache/*
rm -rf /var/www/whmcs/tmp/*
```

## Step 3: Clear Asset Cache

```bash
# Clear minified assets
rm -rf /var/www/whmcs/cache/assets/*

# Clear compiled CSS/JS
rm -rf /var/www/whmcs/cache/*.css
rm -rf /var/www/whmcs/cache/*.js
```

## Step 4: Clear Database Query Cache

```bash
mysql -u root -p -e "FLUSH QUERY CACHE;"
mysql -u root -p -e "RESET QUERY CACHE;"
```

## Step 5: Clear Opcode Cache

```bash
# For PHP-FPM
systemctl restart php8.1-fpm

# For Apache mod_php
systemctl restart apache2

# For OPcache
php -r "opcache_reset();"
```

## Step 6: Clear CDN Cache (if used)

If using CloudFlare or similar:
1. Log in to CDN dashboard
2. Purge cache for WHMCS domain

## Step 7: Clear Browser Cache

For users experiencing issues:
1. Ctrl+Shift+Delete (Windows)
2. Cmd+Shift+Delete (Mac)
3. Select "Cached images and files"
4. Clear browsing data

## Step 8: Automated Cache Clear Script

Create script at `/usr/local/bin/whmcs-clear-cache.sh`:

```bash
#!/bin/bash
# WHMCS Cache Clear Script

WHMCS_DIR="/var/www/whmcs"

echo "Clearing WHMCS cache..."

# Clear template cache
rm -rf $WHMCS_DIR/templates_c/*
echo "Template cache cleared"

# Clear system cache
rm -rf $WHMCS_DIR/cache/*
echo "System cache cleared"

# Clear temp files
rm -rf $WHMCS_DIR/tmp/*
echo "Temp files cleared"

# Reset PHP opcode
/usr/bin/php -r "if(function_exists('opcache_reset')){opcache_reset();}"
echo "PHP opcode cache cleared"

# Reset MySQL query cache
mysql -u root -p -e "FLUSH QUERY CACHE;" 2>/dev/null
echo "MySQL query cache cleared"

echo "All caches cleared successfully"
```

```bash
chmod +x /usr/local/bin/whmcs-clear-cache.sh
```

## Step 9: Selective Cache Clearing

### Clear Specific Product Cache
```bash
rm -rf /var/www/whmcs/cache/products/*
```

### Clear Specific Module Cache
```bash
rm -rf /var/www/whmcs/cache/modules/*
```

### Clear DNS Cache
```bash
rm -rf /var/www/whmcs/cache/dns/*
```

## Step 10: Verify Cache Cleared

Navigate to: Utilities > System Health Status

Refresh and verify no cache-related warnings.

## Cache Locations Reference

| Type | Location |
|------|----------|
| Template | /templates_c/ |
| Smarty | /cache/smarty_cache/ |
| System | /cache/ |
| Assets | /cache/assets/ |
| Temp | /tmp/ |

## Cache Checklist

- [ ] Template cache cleared
- [ ] System cache cleared
- [ ] Asset cache cleared
- [ ] Database cache cleared
- [ ] Opcode cache cleared
- [ ] CDN cache cleared (if applicable)
- [ ] Browser cache cleared (users)
- [ ] Verification completed
