# WHMCS System Cloning Workflow

## Description
Step-by-step guide for cloning a WHMCS installation for staging/testing.

## Prerequisites
- Source WHMCS installation
- New server or directory for clone
- SSH access
- Database credentials

## Use Cases
- Staging environment for testing
- Development environment
- Training environment
- Migration testing

## Steps

### Step 1: Prepare Target Environment
```bash
# Create target directory
mkdir -p /var/www/whmcs_staging

# Create database
mysql -u root -p -e "CREATE DATABASE whmcs_staging CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -u root -p -e "CREATE USER 'whmcs_stage'@'localhost' IDENTIFIED BY 'staging_password';"
mysql -u root -p -e "GRANT ALL PRIVILEGES ON whmcs_staging.* TO 'whmcs_stage'@'localhost';"
mysql -u root -p -e "FLUSH PRIVILEGES;"
```

### Step 2: Backup Source Installation
```bash
# Backup files
cd /var/www
tar -czvf whmcs_source_backup.tar.gz whmcs/

# Backup database
mysqldump -u root -p whmcs > whmcs_source_db.sql
```

### Step 3: Copy Files to Target
```bash
# Extract to new location
tar -xzvf whmcs_source_backup.tar.gz -C /var/www/
mv /var/www/whmcs /var/www/whmcs_staging

# Or use rsync
rsync -avz /var/www/whmcs/ /var/www/whmcs_staging/
```

### Step 4: Create Staging Configuration
```php
// Create /var/www/whmcs_staging/configuration.php
// Copy from production and modify:

<?php
// Staging Environment Configuration
$db_host = "localhost";
$db_username = "whmcs_stage";
$db_password = "staging_password";
$db_name = "whmcs_staging";

$whmcs =WHMCS\Application::factory();
$whmcs->set_config("Domain", "staging.whmcs.example.com");
$whmcs->set_config("SystemURL", "https://staging.whmcs.example.com");
$whmcs->set_config("SystemSSLURL", "https://staging.whmcs.example.com");
$whmcs->set_config("AdminURL", "https://staging.whmcs.example.com/admin");
```

### Step 5: Restore Database
```bash
# Import database to staging
mysql -u root -p whmcs_staging < whmcs_source_db.sql
```

### Step 6: Update Database for Staging
```sql
-- Update URLs in database
UPDATE tblconfiguration SET value = 'https://staging.whmcs.example.com' WHERE setting IN ('SystemURL','SystemSSLURL','Domain');

-- Disable real payment processing
UPDATE tblconfiguration SET value = '1' WHERE setting = 'PaymentsOffline';

-- Disable email sending (or redirect)
UPDATE tblconfiguration SET value = 'demo@example.com' WHERE setting = 'Email';

-- Mark as staging environment
INSERT INTO tblconfiguration (setting, value) VALUES ('StagingEnvironment', '1');
```

### Step 7: Configure Email Redirection (Optional)
```php
// Add to configuration.php for staging
// Redirect all emails to staging box
add_hook('EmailPreSend', 1, function($vars) {
    $vars['email'] = array(
        'recipient' => 'staging-emails@example.com',
        'subject' => '[STAGING] ' . $vars['subject'],
    );
    return $vars;
});
```

### Step 8: Set Permissions
```bash
cd /var/www/whmcs_staging

chown -R www-data:www-data .
chmod 755 .
chmod 644 configuration.php
chmod 755 templates_c attachments downloads language cache
```

### Step 9: Clear Cache
```bash
cd /var/www/whmcs_staging
rm -rf templates_c/*
rm -rf cache/*
php -q whmcs/cli/clear-cache.php
```

### Step 10: Configure Web Server
```nginx
# Nginx configuration for staging
server {
    listen 443 ssl;
    server_name staging.whmcs.example.com;
    root /var/www/whmcs_staging;
    index index.php;
    
    ssl_certificate /path/to/staging-cert.pem;
    ssl_certificate_key /path/to/staging-key.pem;
    
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    
    # IP restriction (optional)
    allow 192.168.1.0/24;  # Your IP range
    allow 10.0.0.0/8;       # Internal network
    deny all;
}
```

### Step 11: Test Staging Environment
1. Access staging URL
2. Login to admin panel
3. Verify all features work
4. Test payment gateway (sandbox mode)
5. Test email system

## Security Considerations
- Use separate database credentials
- Consider IP-based access restriction
- Use self-signed or separate SSL certificate
- Never connect staging to production payment gateways
- Sanitize sensitive data before cloning

## Data Sanitization Script
```bash
# Sanitize cloned database
mysql -u root -p whmcs_staging << 'EOF'
-- Replace real emails with test emails
UPDATE tblclients SET email = CONCAT('test_', id, '@staging.example.com');

-- Remove real passwords (require password reset)
UPDATE tblclients SET password = MD5('staging_only_reset_me');

-- Replace admin passwords
UPDATE tbladmins SET password = MD5('staging_admin_reset');

-- Clear API tokens
UPDATE tbladmins SET api_key = CONCAT('staging_key_', id);
EOF
```

## Verification Checklist
- [ ] Admin login works
- [ ] All modules load correctly
- [ ] Payment gateways work in sandbox mode
- [ ] Emails are redirected
- [ ] Data is sanitized
- [ ] Staging URL is accessible

## Tags
- cloning
- staging
- development
- testing