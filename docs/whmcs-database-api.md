# WHMCS Database API Reference

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-database-design`, `module-database-patterns`, `whmcs-factory-pattern`

---

## Overview

WHMCS uses Laravel's Eloquent ORM through the `Capsule` class for database operations. This provides a powerful query builder and schema builder for module development.

---

## Capsule Setup

### Basic Initialization

```php
<?php
use WHMCS\Database\Capsule;

// Capsule is already loaded in WHMCS environment
// No initialization needed in modules/hooks
```

### Table Prefixes

All WHMCS tables use the prefix defined in `configuration.php` (`$db_prefix`). Use backticks for table names:

```php
// Correct - use backticks
Capsule::table('`tblhosting`');

// Or use the prefix-aware method
$tableName = 'tblhosting'; // WHMCS handles prefix automatically
```

---

## Query Builder

### Select Queries

```php
// Get all records
$clients = Capsule::table('tblclients')->get();

// Get single record
$client = Capsule::table('tblclients')->where('id', 123)->first();

// Get specific column
$email = Capsule::table('tblclients')->where('id', 123)->value('email');

// Get column as array
$names = Capsule::table('tblclients')->pluck('firstname')->toArray();

// Count records
$count = Capsule::table('tblclients')->where('status', 'Active')->count();

// Check existence
$exists = Capsule::table('tblclients')->where('email', $email)->exists();
```

### Where Clauses

```php
// Simple where
$active = Capsule::table('tblclients')->where('status', 'Active')->get();

// Multiple conditions
$clients = Capsule::table('tblclients')
    ->where('status', 'Active')
    ->where('country', 'US')
    ->get();

// OR conditions
$clients = Capsule::table('tblclients')
    ->where('status', 'Active')
    ->orWhere('status', 'Inactive')
    ->get();

// Where in
$clients = Capsule::table('tblclients')
    ->whereIn('id', [1, 2, 3, 4, 5])
    ->get();

// Where between
$invoices = Capsule::table('tblinvoices')
    ->whereBetween('total', [100, 500])
    ->get();

// Where null/not null
$clients = Capsule::table('tblclients')
    ->whereNull('companyname')
    ->get();

// Raw where
$clients = Capsule::table('tblclients')
    ->whereRaw("DATE(datecreated) = DATE(NOW())")
    ->get();
```

### Ordering and Limiting

```php
// Order by
$clients = Capsule::table('tblclients')
    ->orderBy('datecreated', 'desc')
    ->get();

// Multiple order
$clients = Capsule::table('tblclients')
    ->orderBy('lastname')
    ->orderBy('firstname')
    ->get();

// Limit
$recentClients = Capsule::table('tblclients')
    ->orderBy('datecreated', 'desc')
    ->limit(10)
    ->get();

// Offset
$pagedClients = Capsule::table('tblclients')
    ->limit(10)
    ->offset(20)
    ->get();
```

### Aggregates

```php
// Count
$count = Capsule::table('tblclients')->count();

// Sum
$totalRevenue = Capsule::table('tblinvoices')
    ->where('status', 'Paid')
    ->sum('total');

// Average
$avgOrderValue = Capsule::table('tblorders')
    ->where('status', 'Completed')
    ->avg('total');

// Min/Max
$minAmount = Capsule::table('tblinvoices')->min('total');
$maxAmount = Capsule::table('tblinvoices')->max('total');
```

### Joins

```php
$invoices = Capsule::table('tblinvoices as i')
    ->join('tblclients as c', 'c.id', '=', 'i.userid')
    ->select('i.*', 'c.firstname', 'c.lastname', 'c.email')
    ->where('i.status', 'Unpaid')
    ->get();

// Left join
$services = Capsule::table('tblhosting as h')
    ->leftJoin('tblproducts as p', 'p.id', '=', 'h.packageid')
    ->select('h.*', 'p.name as product_name')
    ->get();
```

### Group By and Having

```php
$stats = Capsule::table('tblinvoices')
    ->selectRaw('status, COUNT(*) as count, SUM(total) as total')
    ->groupBy('status')
    ->having('count', '>', 10)
    ->get();

// With raw expressions
$monthlyStats = Capsule::table('tblorders')
    ->selectRaw('YEAR(date) as year, MONTH(date) as month, COUNT(*) as orders')
    ->groupByRaw('YEAR(date), MONTH(date)')
    ->orderByRaw('YEAR(date), MONTH(date)')
    ->get();
```

---

## Insert Operations

### Insert Single Record

```php
// Basic insert
Capsule::table('tblclients')->insert([
    'firstname' => 'John',
    'lastname' => 'Doe',
    'email' => 'john@example.com',
    'datecreated' => date('Y-m-d H:i:s'),
    'password' => password_hash('password', PASSWORD_DEFAULT),
]);

// Get inserted ID
$id = Capsule::table('tblclients')->insertGetId([
    'firstname' => 'John',
    'lastname' => 'Doe',
    'email' => 'john@example.com',
    'datecreated' => date('Y-m-d H:i:s'),
]);
```

### Insert Multiple Records

```php
Capsule::table('mod_email_queue')->insert([
    [
        'user_id' => 1,
        'template' => 'welcome',
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s'),
    ],
    [
        'user_id' => 2,
        'template' => 'welcome',
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s'),
    ],
    [
        'user_id' => 3,
        'template' => 'welcome',
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s'),
    ],
]);
```

---

## Update Operations

### Basic Update

```php
Capsule::table('tblclients')
    ->where('id', 123)
    ->update([
        'firstname' => 'Jane',
        'companyname' => 'Acme Corp',
    ]);
```

### Update or Insert (Upsert)

```php
Capsule::table('mod_settings')->updateOrInsert(
    ['setting_key' => 'api_key'],  // Where clause
    [  // Data to update/insert
        'setting_value' => 'new_value',
        'updated_at' => date('Y-m-d H:i:s'),
    ]
);
```

### Increment/Decrement

```php
// Increment
Capsule::table('mod_api_usage')
    ->where('client_id', $clientId)
    ->increment('request_count');

// Increment with value
Capsule::table('tblcredit')
    ->where('clientid', $clientId)
    ->increment('amount', 50.00);

// Decrement
Capsule::table('tblhosting')
    ->where('id', $serviceId)
    ->decrement('disk_usage', $bytesUsed);
```

---

## Delete Operations

```php
// Delete single record
Capsule::table('mod_temp_data')
    ->where('id', $id)
    ->delete();

// Delete multiple records
Capsule::table('mod_temp_data')
    ->where('created_at', '<', $threshold)
    ->delete();

// Truncate table (admin only, with caution)
Capsule::table('mod_logs')->truncate();
```

---

## Schema Builder

### Create Table

```php
use WHMCS\Database\Capsule;

Capsule::schema()->create('mod_my_module_data', function($t) {
    // Primary key
    $t->increments('id');

    // String types
    $t->string('title', 255);
    $t->string('slug', 100)->unique();

    // Text types
    $t->text('description');
    $t->longText('metadata')->nullable();

    // Number types
    $t->integer('user_id')->unsigned();
    $t->bigInteger('views')->default(0);

    // Decimal types
    $t->decimal('price', 10, 2)->default(0.00);

    // Date/Time types
    $t->date('expires_at')->nullable();
    $t->dateTime('created_at')->nullable();
    $t->timestamp('updated_at')->useCurrent();

    // Boolean
    $t->boolean('is_active')->default(true);

    // JSON
    $t->json('settings')->nullable();

    // Foreign key
    $t->foreign('user_id')->references('id')->on('tblclients')->onDelete('cascade');

    // Indexes
    $t->index('user_id');
    $t->index(['user_id', 'is_active']);
});
```

### Modify Existing Table

```php
// Add column
Capsule::schema()->table('mod_my_module_data', function($t) {
    $t->string('new_column', 100)->after('existing_column')->nullable();
    $t->unsignedBigInteger('relation_id')->nullable();
});

// Modify column
Capsule::schema()->table('mod_my_module_data', function($t) {
    $t->string('title', 500)->change();
});

// Drop index
Capsule::schema()->table('mod_my_module_data', function($t) {
    $t->dropIndex('user_id');
});

// Drop column
Capsule::schema()->table('mod_my_module_data', function($t) {
    $t->dropColumn('old_column');
});
```

### Drop Table

```php
Capsule::schema()->dropIfExists('mod_my_module_data');
```

### Check Table Existence

```php
if (Capsule::schema()->hasTable('mod_my_module_data')) {
    // Table exists
}
```

### Check Column Existence

```php
if (Capsule::schema()->hasColumn('mod_my_module_data', 'new_column')) {
    // Column exists
}
```

---

## Eloquent Models

### Defining Models

```php
<?php
namespace WHMCS\Module\MyModule\Models;

use WHMCS\Database\Capsule;

class ServiceUsage extends \Illuminate\Database\Eloquent\Model
{
    protected $table = 'mod_service_usage';

    protected $fillable = [
        'service_id',
        'user_id',
        'usage_type',
        'usage_value',
        'usage_date',
    ];

    protected $casts = [
        'usage_value' => 'float',
        'usage_date' => 'date',
    ];

    // Relationship
    public function service()
    {
        return $this->belongsTo(\WHMCS\Service\Migration::class, 'service_id');
    }

    // Scopes
    public function scopeActive($query)
    {
        return $query->where('status', 'active');
    }

    public function scopeForDate($query, $date)
    {
        return $query->whereDate('usage_date', $date);
    }
}
```

### Using Models

```php
// Create
$usage = ServiceUsage::create([
    'service_id' => 123,
    'user_id' => 456,
    'usage_type' => 'bandwidth',
    'usage_value' => 1024.5,
    'usage_date' => date('Y-m-d'),
]);

// Find
$usage = ServiceUsage::find($id);

// Update
$usage->usage_value = 2048;
$usage->save();

// Delete
$usage->delete();

// Query with scopes
$stats = ServiceUsage::forDate($date)->get();
```

---

## Transactions

```php
use WHMCS\Database\Capsule;

Capsule::connection()->transaction(function() {
    // Create order
    $orderId = Capsule::table('tblorders')->insertGetId([
        'userid' => $userId,
        'date' => date('Y-m-d H:i:s'),
        'status' => 'Pending',
    ]);

    // Create order items
    Capsule::table('tblorderitems')->insert([
        'orderid' => $orderId,
        'type' => 'hosting',
        'relid' => $productId,
    ]);

    // Create invoice
    Capsule::table('tblinvoices')->insert([
        'userid' => $userId,
        'invoicenum' => generateInvoiceNumber(),
        'date' => date('Y-m-d H:i:s'),
        'duedate' => date('Y-m-d H:i:s', strtotime('+7 days')),
        'total' => $amount,
    ]);

    // If any fails, all rollback
});
```

---

## Raw Queries

### Raw Select

```php
$results = Capsule::select("
    SELECT c.*, COUNT(o.id) as order_count
    FROM tblclients c
    LEFT JOIN tblorders o ON o.userid
    WHERE c.status = ?
    GROUP BY c.id
", ['Active']);
```

### Raw Statements

```php
// For INSERT, UPDATE, DELETE without results
Capsule::statement("
    DELETE FROM mod_temp_data
    WHERE created_at < DATE_SUB(NOW(), INTERVAL 30 DAY)
");
```

---

## Logging Queries

```php
// Enable query logging for debugging
Capsule::connection()->enableQueryLog();

// Execute queries
$results = Capsule::table('tblclients')->get();

// Get logged queries
$queries = Capsule::getQueryLog();

// Log to file
foreach ($queries as $query) {
    logActivity("Query: " . $query['query'] . " - Bindings: " . json_encode($query['bindings']));
}
```

---

## Common Table Reference

| Table | Description |
|-------|-------------|
| `tblclients` | Client accounts |
| `tblhosting` | Hosting/services |
| `tblproducts` | Products |
| `tblorders` | Orders |
| `tblinvoices` | Invoices |
| `tblinvoiceitems` | Invoice line items |
| `tbldomains` | Domain registrations |
| `tbltickets` | Support tickets |
| `tblticketlog` | Ticket history |
| `tblcustomfields` | Custom fields |
| `tblcustomfieldsvalues` | Custom field values |
| `tbladmins` | Admin users |
| `tblactivitylog` | Activity log |

---

## Related Documentation

- [Database Schema Design](database-schema-design.md)
- [Module Database Patterns](module-database-patterns.md)
- [Database Migrations](database-migrations.md)
