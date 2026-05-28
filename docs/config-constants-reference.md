# WHMCS Configuration Constants Reference

**Version:** 8.x
**Updated:** 2026-05-28
**Related Skills:** `security-best-practices`, `performance-optimization`

## Overview

This reference documents all WHMCS configuration constants, their purposes, and recommended values. Constants are set in `configuration.php` and various configuration files.

## Core Configuration

### Database Connection

```php
// configuration.php

// Database Host
// Default: 'localhost'
// Use socket for improved security
$db_host = 'localhost';
// Alternative: $db_host = '127.0.0.1';
// Alternative with socket: $db_host = '/var/run/mysqld/mysqld.sock';

// Database Port
// Default: 3306
$db_port = 3306;

// Database Name
$db_name = 'whmcs_database';

// Database Username
$db_username = 'whmcs_user';

// Database Password
$db_password = 'secure_password_here';

// Database Type
// Supported: 'mysql' (default)
$db_type = 'mysql';

// Database Charset
// Default: 'utf8mb4'
$db_charset = 'utf8mb4';
```

### Encryption

```php
// Encryption Hash
// CRITICAL: Generate a new secure key for each installation
// Use: openssl_random_pseudo_bytes(32) or similar
$cc_encryption_hash = 'your-32-character-hex-key-here';
// Example: bin2hex(random_bytes(32))
```

### WHMCS Paths

```php
// Define custom paths if needed
// Usually auto-detected, only override if necessary

// $whmcspath = '/';
// $templatespath = ROOTDIR . '/templates';
// $attachmentspath = ROOTDIR . '/attachments';
// $downloadsspath = ROOTDIR . '/downloads';
```

### General Settings

```php
// Disable WHMCS Branding
// When true, removes WHMCS branding from client area
$disable_whmcs_brand = false;

// Use Custom Client Area Title
$display_errors = false; // Show errors (never in production)
$log_errors = true; // Log errors to storage/logs/

// PHP Error Display (deprecated in favor of WHMCS error handling)
$debug = false;
$display_errors = false;
```

## Security Constants

```php
// configuration.php

/**
 * Security Keys
 */

// CSRF Token Key
// Used for form security
define('CSRF_TOKEN_KEY', 'your-csrf-key');

/**
 * CAPTCHA Settings
 */

// reCAPTCHA Keys
$recaptcha_public_key = 'your-public-key';
$recaptcha_private_key = 'your-private-key';

// hCaptcha Keys
$hcaptcha_site_key = 'your-site-key';
$hcaptcha_secret_key = 'your-secret-key';

/**
 * Session Security
 */

// Session prefix (useful for multiple WHMCS installations)
$session_prefix = 'whmcs_';

// Session directory (for file-based sessions)
$session_save_path = '/path/to/sessions';

/**
 * API Security
 */

// Allowed API IPs (comma-separated)
$api_allowed_ips = '192.168.1.0/24, 10.0.0.0/8';

// API access log retention (days)
$api_log_days = 90;

/**
 * Admin Security
 */

// Require 2FA for admin access
$admin_2fa_required = true;

// Admin IP whitelist (comma-separated)
$admin_ip_whitelist = '192.168.1.0/24';

// Auto-logout inactive admins (minutes)
$admin_auto_logout = 60;
```

## Performance Constants

```php
// configuration.php

/**
 * Caching
 */

// Redis Configuration
$redis_cache = [
    'host' => '127.0.0.1',
    'port' => 6379,
    'database' => 0,
    'password' => null,
];

// Memcached Configuration
$memcached = [
    'host' => '127.0.0.1',
    'port' => 11211,
];

/**
 * PHP Settings
 */

// Memory limit
ini_set('memory_limit', '256M');

// Max execution time
ini_set('max_execution_time', 30);

// Display errors (never in production)
ini_set('display_errors', 0);
error_reporting(0);

/**
 * Database Performance
 */

// MySQL connect timeout (seconds)
$mysql_connect_timeout = 10;

// Query timeout (seconds)
$mysql_query_timeout = 30;

// Persistent connections
$pdo_attributes = [
    'ATTR_PERSISTENT' => false,
];
```

## Template Constants

```php
// configuration.php

/**
 * Template Settings
 */

// Default Template
$template = 'six';

// Admin Template
$administrator_theme = 'admin';

// Smarty Settings
$smarty_cache = true; // Enable template caching
$smarty_compile = true; // Enable template compilation
$smarty_debug = false; // Enable Smarty debug console

/**
 * Template Cache Directory
 */
$templates_compiledir = ROOTDIR . '/templates_c';

/**
 * Custom Template Variables
 */
$custom_template_vars = [
    'company_name' => 'Your Company',
    'custom_field' => 'value',
];
```

## Mail Configuration

```php
// configuration.php

/**
 * Mail Method
 * Options: 'mail', 'smtp', 'sendmail', 'pickup'
 */
$mail_method = 'smtp';

/**
 * SMTP Settings
 */
$smtp_host = 'smtp.example.com';
$smtp_port = 587;
$smtp_username = 'noreply@example.com';
$smtp_password = 'smtp-password';
$smtp_ssl = 'tls'; // 'tls', 'ssl', or ''

/**
 * From Address
 */
$email_general_from = 'noreply@example.com';
$email_general_from_name = 'Company Name';

/**
 * Email Branding
 */
$email_logo_url = 'https://example.com/logo.png';
$email_pdf_logo_url = 'https://example.com/pdf-logo.png';
```

## Payment Configuration

```php
// configuration.php

/**
 * Currency Settings
 */

// Default Currency
$currency = 1; // Currency ID

// Currency Format
// 1 = Before, 2 = After
$currency_format = 1;

// Tax Configuration
$taxenabled = true;
$taxrate = 20; // Default tax rate percentage
$taxrate2 = 0; // Secondary tax rate
$taxtype = 'exclusive'; // 'exclusive' or 'inclusive'
$taxcountries = 'all'; // 'all' or comma-separated country codes

/**
 * Invoice Settings
 */
$invoiceincrement = 1; // Starting invoice number
$invoiceincrements = true; // Auto-increment
$invoiceitems = true; // Show line items
$prevatbox = true; // Show VAT box
$auto_close_tickets = false;

/**
 * Payment Gateway
 */

// Default Gateway
$gateway_config = [
    'default_gateway' => 'paypal',
    'allow_balance' => true,
    'overdue_fees' => true,
];

/**
 * Invoice Auto-Creation
 */

// Days before due date to generate invoices
$invoiceGenerationDays = 7;

// Auto-suspend overdue services
$autoSuspension = true;
$suspensionOverdueDays = 14;

// Auto-terminate overdue services
$autoTermination = true;
$terminationOverdueDays = 45;
```

## Domain Configuration

```php
// configuration.php

/**
 * Domain Settings
 */

// Default nameservers
$defaultnameservers = [
    'ns1' => 'ns1.example.com',
    'ns2' => 'ns2.example.com',
    'ns3' => 'ns3.example.com',
    'ns4' => 'ns4.example.com',
    'ns5' => 'ns5.example.com',
];

// Domain pricing
$domainpricing = [
    'register' => true,
    'transfer' => true,
    'renew' => true,
];

// Auto-registration TLDs
$autoregistrar = 'enom';

/**
 * Domain Sync Settings
 */

// Sync domain status (hours)
$domainSyncInterval = 6;

// Days before expiry to warn
$expiryWarningDays = [30, 14, 7, 1];

/**
 * Domain Transfer
 */

// Require EPP code
$domainTransferEppRequired = true;

// Auto-renew on transfer complete
$domainAutoRenew = false;
```

## Support/Ticket Configuration

```php
// configuration.php

/**
 * Support Tickets
 */

// Default ticket status
$ticketstatus = [
    'Open' => '#ff6600',
    'Answered' => '#99cc00',
    'Customer-Reply' => '#0099cc',
    'Closed' => '#cccccc',
];

// Default department
$supportDepartment = 1;

// Ticket merge
$ticketMergingEnabled = true;

// Ticket attachments
$attachmentSizeLimit = 5120; // KB
$allowedFileTypes = '.jpg,.jpeg,.gif,.png,.pdf,.doc,.docx,.txt';

/**
 * Auto-Close Tickets
 */
$autoCloseTicketDays = 7;
$autoCloseTicketStatus = 'Closed';

/**
 * Escalation Rules
 */
$ticketEscalationEnabled = true;
$ticketEscalationMinutes = 60;
```

## Affiliate Configuration

```php
// configuration.php

/**
 * Affiliate Settings
 */

// Enable affiliates
$affiliatesenabled = true;

// Commission percentage
$affiliatecommission = 10; // Percentage

// Commission duration (months)
$affiliatecommissionslast = 3;

// Referral period (days)
$affiliatewindow = 365;

// Minimum payout
$affiliatepayout = 50;

// Payout methods
$affiliatepayoutmethods = ['paypal', 'bank_transfer'];

/**
 * Affiliate Tracking
 */
$affiliatetrackcoupone = true;
$affiliatejoincommission = false;
```

## Logging Constants

```php
// configuration.php

/**
 * Activity Logging
 */

// Log admin actions
$adminlog = true;

// Log client actions
$clientlog = true;

// Log module actions
$modulelog = true;

// Log gateway transactions
$gatewaylog = true;

// Log API calls
$apilog = true;

/**
 * Log Retention
 */
$log_days = 30; // Activity log retention
$modulelog_days = 90; // Module log retention

/**
 * Debug Logging
 */
$debug_log = false;
$debug_type = 'file'; // 'file' or 'email'
$debug_email = 'admin@example.com';
```

## Cron Configuration

```php
// configuration.php

/**
 * Cron Security
 */

// Cron security key
$cron_key = 'your-secure-cron-key';

/**
 * Cron Run Frequency
 */
$cron_schedule = [
    'auto' => true,
    'frequency' => '*/5 * * * *', // Every 5 minutes
];

/**
 * Module Queue
 */
$module_queue_enabled = true;
$module_queue_timeout = 300; // Seconds
```

## Maintenance Mode

```php
// configuration.php

/**
 * Maintenance Mode
 */

// Enable maintenance mode
$maintenance_mode = false;

// Allowed IPs in maintenance mode
$maintenance_allowed_ips = [
    '192.168.1.0/24',
    '10.0.0.0/8',
];

// Maintenance message
$maintenance_message = 'System under maintenance';

/**
 * Update Blocking
 */
$disable_auto_upgrade_check = false;
$allow_incompatible_mods = false;
```

## Third-Party Integrations

```php
// configuration.php

/**
 * Addons
 */

// License key for premium addons
$license_key = 'your-license-key';

/**
 * Remote Systems
 */

// API keys for external services
$api_keys = [
    'crazy_domains' => 'your-api-key',
    'domain_register' => 'your-api-key',
    // Add more as needed
];

/**
 * Webhooks
 */

// Webhook secret for validation
$webhook_secret = 'your-webhook-secret';

/**
 * Single Sign-On
 */
$sso_enabled = true;
$sso_secret_key = 'your-sso-secret';
```

## Environment Configuration

```php
// configuration.php

/**
 * Environment
 */
$environment = 'production'; // 'production', 'staging', 'development'

/**
 * Application Mode
 */
$app_mode = 'site'; // 'site', 'api', 'admin', 'cron'

/**
 * Base URL
 */
$whmcs_url = 'https://yourwhmcs.com/';
$system_url = 'https://yourwhmcs.com';

/**
 * SSL Settings
 */
$ssl = true; // Force SSL
$ssl_dir = 'https://'; // SSL URL prefix

/**
 * Proxy Settings
 */
$proxy_enabled = false;
$proxy_host = '';
$proxy_port = '';
$proxy_username = '';
$proxy_password = '';
```

## Constants Reference Table

| Constant | Type | Default | Description |
|----------|------|---------|-------------|
| ROOTDIR | Path | Auto | WHMCS root directory |
| WHMCS | Boolean | true | WHMCS indicator |
| $db_host | String | localhost | Database host |
| $db_name | String | whmcs | Database name |
| $cc_encryption_hash | String | - | Encryption key |
| $template | String | six | Default template |
| $smarty_cache | Boolean | true | Template caching |
| $cron_key | String | - | Cron security key |
| $affiliatesenabled | Boolean | false | Affiliate system |
| $taxenabled | Boolean | true | Tax system |
| $ssl | Boolean | false | Force SSL |

## Related Documentation

- [Security Best Practices](security-best-practices.md)
- [Performance Optimization](performance-optimization.md)
- [Troubleshooting Guide](troubleshooting-guide.md)
