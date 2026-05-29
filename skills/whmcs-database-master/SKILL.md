# WHMCS Database Master

## Overview
Master skill for WHMCS database operations. Covers database schema, query building, migrations, and best practices.

## Database Schema Reference

### Core Tables

```sql
-- Clients table
CREATE TABLE `tblclients` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `uuid` CHAR(36) NOT NULL,
    `firstname` VARCHAR(100) NOT NULL,
    `lastname` VARCHAR(100) NOT NULL,
    `companyname` VARCHAR(255) DEFAULT NULL,
    `email` VARCHAR(255) NOT NULL,
    `address1` VARCHAR(255) DEFAULT NULL,
    `address2` VARCHAR(255) DEFAULT NULL,
    `city` VARCHAR(255) DEFAULT NULL,
    `state` VARCHAR(255) DEFAULT NULL,
    `postcode` VARCHAR(10) DEFAULT NULL,
    `country` VARCHAR(2) DEFAULT NULL,
    `phonenumber` VARCHAR(20) DEFAULT NULL,
    `password` VARCHAR(255) DEFAULT NULL,
    `passwordtype` VARCHAR(50) DEFAULT NULL,
    `currency` INT(11) DEFAULT NULL,
    `defaultgateway` VARCHAR(50) DEFAULT NULL,
    `credit` DECIMAL(10,2) DEFAULT 0.00,
    `taxexempt` TINYINT(1) DEFAULT 0,
    `latefeeexempt` TINYINT(1) DEFAULT 0,
    `overduenotices` TINYINT(1) DEFAULT 1,
    `separateinvoices` TINYINT(1) DEFAULT 0,
    `disableautocc` TINYINT(1) DEFAULT 0,
    `emailverified` TINYINT(1) DEFAULT 0,
    `created_at` DATETIME DEFAULT NULL,
    `updated_at` DATETIME DEFAULT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `idx_uuid` (`uuid`),
    KEY `idx_email` (`email`),
    KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Hosting/Services table
CREATE TABLE `tblhosting` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `uuid` CHAR(36) NOT NULL,
    `userid` INT(11) NOT NULL,
    `packageid` INT(11) NOT NULL,
    `server` INT(11) DEFAULT NULL,
    `regdate` DATE NOT NULL,
    `domain` VARCHAR(255) DEFAULT NULL,
    `paymentmethod` VARCHAR(50) DEFAULT NULL,
    `firstpaymentamount` DECIMAL(10,2) NOT NULL,
    `recurringamount` DECIMAL(10,2) DEFAULT 0.00,
    `billingcycle` ENUM('Free','One Time','Monthly','Quarterly','Semi-Annually','Annually','Biennially','Triennially') DEFAULT 'Monthly',
    `nextduedate` DATE DEFAULT NULL,
    `nextinvoicedate` DATE DEFAULT NULL,
    `termination_date` DATE DEFAULT NULL,
    `completed_date` DATE DEFAULT NULL,
    `username` VARCHAR(255) DEFAULT NULL,
    `password` VARCHAR(255) DEFAULT NULL,
    `subscriptionid` VARCHAR(255) DEFAULT NULL,
    `status` ENUM('Pending','Active','Suspended','Terminated','Cancelled','Fraud') DEFAULT 'Pending',
    `username_left` VARCHAR(255) DEFAULT NULL,
    `bandwidth_used` BIGINT(20) DEFAULT 0,
    `bandwidth_limit` BIGINT(20) DEFAULT 0,
    `disk_used` BIGINT(20) DEFAULT 0,
    `disk_limit` BIGINT(20) DEFAULT 0,
    PRIMARY KEY (`id`),
    KEY `idx_userid` (`userid`),
    KEY `idx_domain` (`domain`),
    KEY `idx_server` (`server`),
    KEY `idx_status` (`status`),
    KEY `idx_nextduedate` (`nextduedate`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Products table
CREATE TABLE `tblproducts` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `type` ENUM('hostingaccount','reselleraccount','server','other','dns') DEFAULT 'hostingaccount',
    `gid` INT(11) DEFAULT NULL,
    `name` VARCHAR(255) NOT NULL,
    `description` TEXT,
    `stock` INT(11) DEFAULT NULL,
    `stockcontrol` TINYINT(1) DEFAULT 0,
    `paytype` ENUM('free','onoff','recurring') DEFAULT 'recurring',
    `pricing` TEXT,
    `features` TEXT,
    `tax` TINYINT(1) DEFAULT 1,
    `hidden` TINYINT(1) DEFAULT 0,
    `showdomainoptions` TINYINT(1) DEFAULT 0,
    `proratabilling` TINYINT(1) DEFAULT 0,
    `proratadate` INT(11) DEFAULT NULL,
    `proratachargenextmonth` TINYINT(1) DEFAULT 0,
    `autosetup` TINYINT(1) DEFAULT 0,
    `autosetupperiod` INT(11) DEFAULT NULL,
    `module` VARCHAR(50) DEFAULT NULL,
    `servergroup` INT(11) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `idx_gid` (`gid`),
    KEY `idx_type` (`type`),
    KEY `idx_hidden` (`hidden`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Invoices table
CREATE TABLE `tblinvoices` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `userid` INT(11) NOT NULL,
    `invoicenum` VARCHAR(50) DEFAULT NULL,
    `date` DATE NOT NULL,
    `duedate` DATE DEFAULT NULL,
    `datepaid` DATETIME DEFAULT NULL,
    `status` ENUM('Draft','Unpaid','Paid','Cancelled','Refunded','Collection') DEFAULT 'Unpaid',
    `subtotal` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    `tax` DECIMAL(10,2) DEFAULT 0.00,
    `tax2` DECIMAL(10,2) DEFAULT 0.00,
    `total` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    `taxrate` DECIMAL(10,2) DEFAULT 0.00,
    `taxrate2` DECIMAL(10,2) DEFAULT 0.00,
    `paymentmethod` VARCHAR(50) DEFAULT NULL,
    `paymentmethodname` VARCHAR(255) DEFAULT NULL,
    `notes` TEXT,
    PRIMARY KEY (`id`),
    KEY `idx_userid` (`userid`),
    KEY `idx_status` (`status`),
    KEY `idx_date` (`date`),
    KEY `idx_duedate` (`duedate`),
    KEY `idx_invoicenum` (`invoicenum`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Orders table
CREATE TABLE `tblorders` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `ordernum` VARCHAR(50) NOT NULL,
    `userid` INT(11) NOT NULL,
    `contactid` INT(11) DEFAULT NULL,
    `dates` DATETIME NOT NULL,
    ` fulfillmentDate` DATE DEFAULT NULL,
    `registerDate` DATE DEFAULT NULL,
    `domain` VARCHAR(255) DEFAULT NULL,
    `domainid` INT(11) DEFAULT NULL,
    `firstpaymentamount` DECIMAL(10,2) NOT NULL,
    `recurringamount` DECIMAL(10,2) DEFAULT 0.00,
    `paymentmethod` VARCHAR(50) DEFAULT NULL,
    `paymentmethodname` VARCHAR(255) DEFAULT NULL,
    `invoiceid` INT(11) DEFAULT NULL,
    `status` ENUM('Pending','Active','Fraud','Cancelled') DEFAULT 'Pending',
    `ipaddress` VARCHAR(45) DEFAULT NULL,
    `hostname` VARCHAR(255) DEFAULT NULL,
    `notes` TEXT,
    `paymentlogic` VARCHAR(50) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `idx_userid` (`userid`),
    KEY `idx_status` (`status`),
    KEY `idx_dates` (`dates`),
    KEY `idx_ordernum` (`ordernum`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Tickets table
CREATE TABLE `tbltickets` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `tid` VARCHAR(10) NOT NULL,
    `did` INT(11) DEFAULT NULL,
    `userid` INT(11) DEFAULT NULL,
    `contactid` INT(11) DEFAULT NULL,
    `adminid` INT(11) DEFAULT NULL,
    `user_id` INT(11) DEFAULT NULL,
    `cc` VARCHAR(500) DEFAULT NULL,
    `name` VARCHAR(255) DEFAULT NULL,
    `email` VARCHAR(255) DEFAULT NULL,
    `date` DATETIME NOT NULL,
    `lastreply` DATETIME DEFAULT NULL,
    `subject` VARCHAR(255) NOT NULL,
    `message` TEXT,
    `status` VARCHAR(50) DEFAULT NULL,
    `priority` ENUM('Low','Medium','High') DEFAULT 'Medium',
    `flag` INT(11) DEFAULT NULL,
    `answered` TINYINT(1) DEFAULT 0,
    PRIMARY KEY (`id`),
    UNIQUE KEY `idx_tid` (`tid`),
    KEY `idx_userid` (`userid`),
    KEY `idx_did` (`did`),
    KEY `idx_status` (`status`),
    KEY `idx_priority` (`priority`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Query Builder Examples

```php
<?php
// Database query examples using Laravel's query builder

use Illuminate\Database\Capsule\Manager as DB;

// Basic select
$clients = DB::table('tblclients')->get();

// Select specific columns
$clients = DB::table('tblclients')
    ->select('id', 'firstname', 'lastname', 'email')
    ->get();

// Where conditions
$activeClients = DB::table('tblclients')
    ->where('status', 'Active')
    ->get();

// Multiple conditions
$clients = DB::table('tblclients')
    ->where('country', 'US')
    ->where('currency', 1)
    ->get();

// OR conditions
$clients = DB::table('tblclients')
    ->where('country', 'US')
    ->orWhere('country', 'CA')
    ->get();

// WhereIn
$clients = DB::table('tblclients')
    ->whereIn('id', [1, 2, 3, 4, 5])
    ->get();

// Like search
$clients = DB::table('tblclients')
    ->where('email', 'like', '%@gmail.com')
    ->get();

// Join tables
$orders = DB::table('tblorders')
    ->join('tblclients', 'tblorders.userid', '=', 'tblclients.id')
    ->select('tblorders.*', 'tblclients.firstname', 'tblclients.lastname')
    ->where('tblorders.status', 'Pending')
    ->get();

// Left join
$invoices = DB::table('tblinvoices')
    ->leftJoin('tblclients', 'tblinvoices.userid', '=', 'tblclients.id')
    ->select('tblinvoices.*', 'tblclients.email')
    ->where('tblinvoices.status', 'Unpaid')
    ->get();

// Aggregates
$count = DB::table('tblclients')->count();
$total = DB::table('tblinvoices')->where('status', 'Paid')->sum('total');
$avg = DB::table('tblorders')->avg('total');
$max = DB::table('tblorders')->max('total');

// Subquery
$recentOrders = DB::table('tblorders')
    ->whereIn('userid', function($query) {
        $query->select('id')
            ->from('tblclients')
            ->where('country', 'US');
    })
    ->get();

// Order and limit
$clients = DB::table('tblclients')
    ->orderBy('created_at', 'desc')
    ->limit(10)
    ->get();

// Pagination
$clients = DB::table('tblclients')
    ->paginate(15);

// Insert
$id = DB::table('tblclients')
    ->insertGetId([
        'firstname' => 'John',
        'lastname' => 'Doe',
        'email' => 'john@example.com',
        'created_at' => date('Y-m-d H:i:s'),
    ]);

// Update
DB::table('tblclients')
    ->where('id', $id)
    ->update([
        'companyname' => 'Acme Inc',
        'updated_at' => date('Y-m-d H:i:s'),
    ]);

// Delete
DB::table('tblclients')
    ->where('id', $id)
    ->delete();

// Increment/Decrement
DB::table('tblclients')
    ->where('id', $id)
    ->increment('login_count');

DB::table('tblclients')
    ->where('id', $id)
    ->decrement('credit', 10.00);

// Raw queries
$results = DB::select(
    'SELECT * FROM tblclients WHERE country = ? AND status = ?',
    ['US', 'Active']
);

// Raw update
DB::table('tblclients')
    ->where('id', $id)
    ->updateRaw([
        'login_count' => DB::raw('login_count + 1'),
    ]);
```

## Custom Module Database Tables

```php
<?php
// Migration: Create custom module tables

// In your module's activation function
function YourModule_activate()
{
    // Create the main data table
    \Illuminate\Database\Capsule\Manager::statement("
        CREATE TABLE IF NOT EXISTS `mod_yourmodule_data` (
            `id` INT AUTO_INCREMENT PRIMARY KEY,
            `client_id` INT NOT NULL,
            `service_id` INT DEFAULT NULL,
            `name` VARCHAR(255) NOT NULL,
            `value` TEXT,
            `status` VARCHAR(50) DEFAULT 'active',
            `metadata` JSON DEFAULT NULL,
            `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
            `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
            INDEX `idx_client_id` (`client_id`),
            INDEX `idx_service_id` (`service_id`),
            INDEX `idx_status` (`status`),
            FOREIGN KEY (`client_id`) REFERENCES `tblclients`(`id`) ON DELETE CASCADE,
            FOREIGN KEY (`service_id`) REFERENCES `tblhosting`(`id`) ON DELETE SET NULL
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
    ");

    // Create a settings table
    \Illuminate\Database\Capsule\Manager::statement("
        CREATE TABLE IF NOT EXISTS `mod_yourmodule_settings` (
            `id` INT AUTO_INCREMENT PRIMARY KEY,
            `setting_key` VARCHAR(255) NOT NULL UNIQUE,
            `setting_value` TEXT,
            `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
    ");

    // Create an activity log table
    \Illuminate\Database\Capsule\Manager::statement("
        CREATE TABLE IF NOT EXISTS `mod_yourmodule_activity` (
            `id` INT AUTO_INCREMENT PRIMARY KEY,
            `client_id` INT NOT NULL,
            `action` VARCHAR(100) NOT NULL,
            `details` TEXT,
            `ip_address` VARCHAR(45) DEFAULT NULL,
            `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
            INDEX `idx_client_id` (`client_id`),
            INDEX `idx_action` (`action`),
            INDEX `idx_created_at` (`created_at`)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
    ");

    return [
        'status' => 'success',
        'description' => 'Module activated successfully',
    ];
}

function YourModule_deactivate()
{
    // Drop tables (be careful with this in production)
    \Illuminate\Database\Capsule\Manager::statement("DROP TABLE IF EXISTS `mod_yourmodule_activity`");
    \Illuminate\Database\Capsule\Manager::statement("DROP TABLE IF EXISTS `mod_yourmodule_settings`");
    \Illuminate\Database\Capsule\Manager::statement("DROP TABLE IF EXISTS `mod_yourmodule_data`");

    return [
        'status' => 'success',
        'description' => 'Module deactivated successfully',
    ];
}

// Upgrade function
function YourModule_upgrade($vars)
{
    $version = $vars['version'];

    if (version_compare($version, '1.1.0', '<')) {
        // Add new column
        \Illuminate\Database\Capsule\Manager::statement("
            ALTER TABLE `mod_yourmodule_data`
            ADD COLUMN `new_field` VARCHAR(255) DEFAULT NULL AFTER `value`
        ");
    }

    if (version_compare($version, '1.2.0', '<')) {
        // Add index
        \Illuminate\Database\Capsule\Manager::statement("
            ALTER TABLE `mod_yourmodule_data`
            ADD INDEX `idx_new_field` (`new_field`)
        ");
    }

    if (version_compare($version, '1.3.0', '<')) {
        // Create new table
        \Illuminate\Database\Capsule\Manager::statement("
            CREATE TABLE IF NOT EXISTS `mod_yourmodule_cache` (
                `id` INT AUTO_INCREMENT PRIMARY KEY,
                `cache_key` VARCHAR(255) NOT NULL,
                `cache_value` LONGTEXT,
                `expires_at` DATETIME DEFAULT NULL,
                UNIQUE KEY `idx_cache_key` (`cache_key`),
                INDEX `idx_expires_at` (`expires_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
}
```

## Eloquent Models

```php
<?php
// Custom Eloquent models for WHMCS

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class YourModuleData extends Model
{
    protected $table = 'mod_yourmodule_data';

    protected $fillable = [
        'client_id',
        'service_id',
        'name',
        'value',
        'status',
        'metadata',
    ];

    protected $casts = [
        'metadata' => 'array',
        'created_at' => 'datetime',
        'updated_at' => 'datetime',
    ];

    public function client(): BelongsTo
    {
        return $this->belongsTo(\WHMCS\User\Client::class, 'client_id');
    }

    public function service(): BelongsTo
    {
        return $this->belongsTo(\WHMCS\Service\Service::class, 'service_id');
    }

    public function scopeActive($query)
    {
        return $query->where('status', 'active');
    }

    public function scopeForClient($query, int $clientId)
    {
        return $query->where('client_id', $clientId);
    }
}

// Usage examples
$data = YourModuleData::create([
    'client_id' => 1,
    'service_id' => 10,
    'name' => 'api_key',
    'value' => encrypt('secret_key'),
    'status' => 'active',
]);

$clientData = YourModuleData::active()
    ->forClient(1)
    ->get();

foreach ($clientData as $item) {
    echo $item->name . ': ' . decrypt($item->value);
}
```

## Best Practices

1. **Prepared Statements**: Always use prepared statements to prevent SQL injection
2. **Indexes**: Add indexes for frequently queried columns
3. **Foreign Keys**: Use foreign keys for data integrity
4. **Migrations**: Use upgrade functions for schema changes
5. **Backups**: Always backup before making schema changes
6. **Transactions**: Use transactions for multi-table operations
7. **Eager Loading**: Use eager loading to avoid N+1 queries
8. **Soft Deletes**: Consider soft deletes for important data
9. **JSON Fields**: Use JSON fields for flexible metadata
10. **Logging**: Log significant database operations
