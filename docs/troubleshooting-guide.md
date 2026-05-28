# WHMCS Troubleshooting Guide

**Version:** 8.x
**Updated:** 2026-05-28
**Related Skills:** `error-codes-reference`, `performance-optimization`, `security-best-practices`

## Overview

This guide covers common WHMCS issues, their diagnosis, and resolution strategies. Use this as a reference when encountering problems with your WHMCS installation.

## Common Issues

### Installation Issues

#### Blank White Screen

**Symptoms:** White screen after installation or upgrade.

**Diagnosis:**
```php
// Enable error display
// Add to configuration.php
ini_set('display_errors', 1);
error_reporting(E_ALL);
ini_set('display_startup_errors', 1);

// Or check error log
// /storage/logs/*.log
```

**Solutions:**
1. Check PHP version compatibility
2. Verify file permissions (755 dirs, 644 files)
3. Clear templates_c directory
4. Check for syntax errors in custom code

```bash
# Clear templates cache
rm -rf /path/to/whmcs/templates_c/*

# Check PHP syntax
find /path/to/whmcs -name "*.php" -exec php -l {} \;

# Verify file permissions
chmod 755 /path/to/whmcs
chmod 644 /path/to/whmcs/*.php
chmod 755 /path/to/whmcs/includes
chmod 644 /path/to/whmcs/includes/*.php
```

#### Database Connection Errors

**Error:** `Could not connect to database`

**Diagnosis:**
```php
// Verify configuration.php settings
$db_host = 'localhost';
$db_username = 'whmcs_user';
$db_password = 'your_password';
$db_name = 'whmcs_database';

// Test connection
try {
    $pdo = new PDO(
        "mysql:host=$db_host;dbname=$db_name",
        $db_username,
        $db_password
    );
    echo "Connected successfully";
} catch (PDOException $e) {
    echo "Connection failed: " . $e->getMessage();
}
```

**Solutions:**
1. Verify MySQL service is running
2. Check credentials in configuration.php
3. Verify database user has proper permissions
4. Check firewall settings for MySQL port (3306)
5. Ensure SELinux/AppArmor not blocking connection

```bash
# Check MySQL status
systemctl status mysql
# or
service mysql status

# Test MySQL connection
mysql -u whmcs_user -p whmcs_database

# Verify database exists
mysql -u root -p -e "SHOW DATABASES;"

# Grant permissions if needed
GRANT ALL PRIVILEGES ON whmcs_database.* TO 'whmcs_user'@'localhost';
FLUSH PRIVILEGES;
```

### Email Issues

#### Emails Not Sending

**Diagnosis:**
```php
// Check mail configuration in WHMCS Admin
// Configuration > System Settings > Mail Settings

// Test SMTP connection
$smtp = fsockopen('smtp.example.com', 587, $errno, $errstr, 30);
if ($smtp) {
    echo "SMTP connection successful";
    fclose($smtp);
} else {
    echo "SMTP connection failed: $errstr ($errno)";
}

// Check email log
// storage/logs/
```

**Solutions:**

1. **SMTP Configuration**
```php
// Add to configuration.php
$mail_config = [
    'mailer' => 'smtp',
    'smtp_host' => 'smtp.example.com',
    'smtp_port' => 587,
    'smtp_username' => 'your-email@example.com',
    'smtp_password' => 'your-password',
    'smtp_ssl' => 'tls',
    'smtp_timeout' => 30,
];
```

2. **Common SMTP Issues**
```bash
# Test SMTP manually
telnet smtp.example.com 587
EHLO localhost
AUTH LOGIN
# (enter credentials in base64)

# Check SSL/TLS
openssl s_client -connect smtp.example.com:587 -starttls
```

3. **PHP mail() Issues**
```php
// Check PHP mail configuration
php -i | grep -i mail

// Verify sendmail path in php.ini
sendmail_path = "/usr/sbin/sendmail -t -i"
```

4. **Email Queue Issues**
```sql
-- Check email queue
SELECT * FROM tblemails WHERE status = 'queued' LIMIT 10;

-- Count pending emails
SELECT COUNT(*) as pending FROM tblemails WHERE status = 'queued';

-- Clear stuck queue
UPDATE tblemails SET status = 'unsent' WHERE status = 'queued' AND created < DATE_SUB(NOW(), INTERVAL 1 DAY);
```

#### Emails Going to Spam

**Solutions:**
1. Configure proper SPF, DKIM, and DMARC records
2. Use dedicated sending IPs
3. Warm up new sending domains
4. Authenticate with major email providers
5. Keep bounce rates below 2%

```bash
# SPF Record
v=spf1 include:_spf.yourmailprovider.com ~all

# DKIM Record (add DNS TXT record)
v=DKIM1; k=rsa; p=YOUR_PUBLIC_KEY;

# DMARC Record
v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@example.com;
```

### Payment Gateway Issues

#### Gateway Not Processing

**Diagnosis:**
```php
// Check gateway logs
// storage/logs/gatewayname.log

// Verify gateway credentials
// Configuration > Payments > Payment Gateways

// Test callback URL manually
$callbackUrl = 'https://yourwhmcs.com/gateways/callback/gatewayname.php';
$testData = [
    'invoice_id' => 123,
    'amount' => 100.00,
    'transaction_id' => 'TEST_' . time(),
];
```

**Common Gateway Issues:**

1. **PayPal Issues**
```php
// Verify PayPal settings
$paypal_config = [
    'BusinessEmail' => 'your@email.com',
    'GatewayFee' => 0,
    'SandboxMode' => false, // Set true only for testing
];

// Test PayPal IPN
// Use https://developer.paypal.com/ipn/simulator/

// Check PayPal log
logActivity("PayPal IPN received: " . print_r($_POST, true));
```

2. **Stripe Issues**
```php
// Verify Stripe settings
$stripe_config = [
    'publishableKey' => 'pk_live_xxx',
    'secretKey' => 'sk_live_xxx',
    'webhookSecret' => 'whsec_xxx',
];

// Test webhook locally with Stripe CLI
stripe listen --forward-to localhost/whmcs/gateways/callback/stripe.php
```

3. **Credit Card Processing Issues**
```php
// Verify SSL certificate
// Payment pages must use HTTPS with valid SSL

// Check PCI compliance requirements
// Ensure CVV and card data are not stored locally
```

### Module Issues

#### Module Not Activating

**Diagnosis:**
```bash
# Check module directory structure
ls -la /path/to/whmcs/modules/servers/yourmodule/

# Verify module.php exists and has required functions
cat /path/to/whmcs/modules/servers/yourmodule/yourmodule.php | head -50

# Check module requirements
# Usually requires:
# - MetaData function
# - getConfigArray function
# - Required module functions (CreateAccount, SuspendAccount, etc.)
```

**Solutions:**
1. Verify module files are complete
2. Check PHP version compatibility
3. Verify required functions exist
4. Enable module debug logging

```php
// Enable module debug in module file
function yourmodule_MetaData(): array
{
    return [
        'DisplayName' => 'Your Module',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'ServiceSingleSignOn' => false,
    ];
}

function yourmodule_ConfigOptions(array $params): array
{
    // Your configuration options
    return [
        'param1' => [
            'Type' => 'text',
            'Size' => '25',
            'Default' => '',
            'Description' => 'Parameter 1',
        ],
    ];
}

// Add debug logging
function yourmodule_CreateAccount(array $params): array
{
    logModuleCall(
        'yourmodule',
        'CreateAccount',
        print_r($params, true),
        'Starting account creation',
        ['password' => '***'] // Mask sensitive data
    );

    // Your code...

    return ['success' => true, 'description' => 'Account created'];
}
```

#### Service Not Provisioning

**Diagnosis:**
```sql
-- Check module queue
SELECT * FROM tblmodule_queue ORDER BY created_at DESC LIMIT 10;

-- Check hosting table status
SELECT id, domain, domainstatus, nextduedate FROM tblhosting
WHERE domainstatus = 'Pending';

-- View module command history
SELECT * FROM tblmodulelog
ORDER BY id DESC LIMIT 20;
```

**Solutions:**
```php
// Force retry module command
add_hook('AdminAreaPage', 1, function($vars) {
    // In admin, you can use ModuleQueue utility
    // Utilities > System > Module Queue
});

// Manual command execution
// Configuration > Products/Services > Module Queue
// Click "Process" for individual items
```

### Database Issues

#### Slow Queries

**Diagnosis:**
```sql
-- Find slow queries
SHOW FULL PROCESSLIST;

-- Check query cache
SHOW STATUS LIKE 'Qcache%';

-- Analyze slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- Use EXPLAIN on slow queries
EXPLAIN SELECT * FROM tblclients WHERE email LIKE '%@example.com';
```

**Solutions:**
```php
// Add indexes for common queries
// See database-indexing-guide.md

// Optimize query in custom code
$clients = Capsule::table('tblclients')
    ->select('id', 'firstname', 'lastname', 'email')
    ->where('status', 'Active')
    ->where('email', 'LIKE', '%example.com')
    ->limit(100)
    ->get();

// Instead of fetching all and filtering in PHP
// $clients = Capsule::table('tblclients')->get();
// foreach ($clients as $client) { if (strpos($client->email, '@example.com')) ... }
```

#### Database Table Corruption

**Diagnosis:**
```bash
# Check tables
mysqlcheck -u root -p --check whmcs_database

# Check specific table
mysqlcheck -u root -p --check whmcs_database.tblclients
```

**Solutions:**
```sql
-- Repair corrupted tables
mysqlcheck -u root -p --repair whmcs_database

-- Repair specific table
REPAIR TABLE tblclients USE_FRM;

-- If InnoDB, try:
ALTER TABLE tblclients ENGINE=InnoDB;
```

### Performance Issues

#### High Server Load

**Diagnosis:**
```bash
# Check system resources
top -c
htop
free -m
df -h

# Check MySQL connections
mysql -u root -p -e "SHOW PROCESSLIST;"

# Check cron execution
# Check /storage/logs/cron/
```

**Solutions:**
```php
// Optimize cron
// See cron-events-reference.md

// Enable caching
// See caching-strategies.md

// Database optimization
// Run in MySQL:
// OPTIMIZE TABLE tblclients;
// OPTIMIZE TABLE tblhosting;
// OPTIMIZE TABLE tblorders;
// OPTIMIZE TABLE tblinvoices;
```

#### WHMCS Running Slow

**Diagnosis:**
```php
// Add to index.php for profiling
define('WHMCS_DEBUG_LOG', true);

// Check debug bar
// Append ?debug=1 to URL
```

**Solutions:**
1. Enable OPcache
2. Use Redis for caching
3. Optimize database queries
4. Clear templates_c
5. Check for infinite loops in hooks

```bash
# Clear all caches
rm -rf /path/to/whmcs/templates_c/*
rm -rf /path/to/whmcs/storage/cache/*
php /path/to/whmcs/crons/cron.php?debug=1

# Check for problematic hooks
# Temporarily disable custom hooks by renaming directory
mv /path/to/whmcs/includes/hooks /path/to/whmcs/includes/hooks_disabled
```

### SSL/TLS Issues

#### Mixed Content Errors

**Diagnosis:**
```javascript
// Browser console will show mixed content warnings
// Look for:
// - http:// resources on https:// page
// - Insecure form submissions
```

**Solutions:**
```php
// Force HTTPS
// configuration.php
$_SERVER['HTTPS'] = 'on';
$_SERVER['SERVER_PORT'] = 443;

// .htaccess
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]

// Update WHMCS base URL
// Admin > Configuration > General Settings > Security
// Ensure "Force SSL" is enabled
```

### Troubleshooting Workflow

1. **Identify the Issue**
   - Error message or symptoms
   - When it started
   - Recent changes made

2. **Check Logs**
   - WHMCS logs in /storage/logs/
   - Server error logs
   - PHP error logs
   - Browser console

3. **Reproduce the Issue**
   - Test in different browsers
   - Test with debug mode
   - Check with fresh install

4. **Research**
   - Check error-codes-reference.md
   - Search WHMCS community forums
   - Check module documentation

5. **Fix and Test**
   - Make minimal changes
   - Test thoroughly
   - Monitor for recurrence

## Related Documentation

- [Error Codes Reference](error-codes-reference.md)
- [Performance Optimization](performance-optimization.md)
- [Database Indexing Guide](database-indexing-guide.md)
