# WHMCS Capsule Queries

## Overview

WHMC's Capsule ORM provides a fluent query builder for database operations. This guide covers advanced query patterns.

## Basic Queries

### Select

```php
<?php
use WHMCS\Database\Capsule;

// Get all results
$clients = Capsule::table('tblclients')->get();

// Get first result
$client = Capsule::table('tblclients')
    ->where('id', 1)
    ->first();

// Get specific columns
$emails = Capsule::table('tblclients')
    ->select('email', 'firstname', 'lastname')
    ->get();

// Get distinct values
$countries = Capsule::table('tblclients')
    ->distinct()
    ->select('country')
    ->get();
```

### Where Clauses

```php
<?php
// Basic where
$activeClients = Capsule::table('tblclients')
    ->where('status', 'Active')
    ->get();

// Multiple conditions
$clients = Capsule::table('tblclients')
    ->where('status', 'Active')
    ->where('country', 'US')
    ->where('created_at', '>=', '2024-01-01')
    ->get();

// Or where
$clients = Capsule::table('tblclients')
    ->where('status', 'Active')
    ->orWhere('status', 'Inactive')
    ->get();

// Where in
$clients = Capsule::table('tblclients')
    ->whereIn('id', [1, 2, 3, 4, 5])
    ->get();

// Where between
$clients = Capsule::table('tblclients')
    ->whereBetween('created_at', ['2024-01-01', '2024-12-31'])
    ->get();

// Where null/not null
$clients = Capsule::table('tblclients')
    ->whereNull('companyname')
    ->get();

// Like/pattern matching
$clients = Capsule::table('tblclients')
    ->where('email', 'like', '%@example.com')
    ->get();
```

## Advanced Querying

### Joins

```php
<?php
// Inner join
$invoices = Capsule::table('tblinvoices')
    ->join('tblclients', 'tblinvoices.userid', '=', 'tblclients.id')
    ->select('tblinvoices.*', 'tblclients.email', 'tblclients.firstname')
    ->get();

// Left join
$services = Capsule::table('tblhosting')
    ->leftJoin('tblhostingaddons', 'tblhosting.id', '=', 'tblhostingaddons.hostingid')
    ->select('tblhosting.*', 'tblhostingaddons.name as addon_name')
    ->get();

// Multiple joins
$orders = Capsule::table('tblorders')
    ->join('tblclients', 'tblorders.userid', '=', 'tblclients.id')
    ->join('tblproducts', 'tblorders.products', '=', 'tblproducts.id')
    ->where('tblorders.status', 'Active')
    ->get();
```

### Aggregates

```php
<?php
// Count
$clientCount = Capsule::table('tblclients')->count();

// Sum
$totalRevenue = Capsule::table('tblinvoices')
    ->where('status', 'Paid')
    ->sum('total');

// Average
$avgOrderValue = Capsule::table('tblorders')
    ->where('status', 'Completed')
    ->avg('totaldue');

// Max/Min
$highestInvoice = Capsule::table('tblinvoices')->max('total');
$lowestInvoice = Capsule::table('tblinvoices')->min('total');

// Group by with aggregates
$revenueByMonth = Capsule::table('tblinvoices')
    ->selectRaw('MONTH(datepaid) as month, SUM(total) as total')
    ->where('status', 'Paid')
    ->groupByRaw('MONTH(datepaid)')
    ->get();
```

### Subqueries

```php
<?php
// Subquery in select
$clients = Capsule::table('tblclients')
    ->select('*')
    ->selectSub(function ($query) {
        $query->from('tblinvoices')
            ->whereColumn('userid', 'tblclients.id')
            ->where('status', 'Paid')
            ->selectRaw('SUM(total)');
    }, 'total_spent')
    ->get();

// Subquery in where
$highValueClients = Capsule::table('tblclients')
    ->whereIn('id', function ($query) {
        $query->from('tblinvoices')
            ->select('userid')
            ->where('status', 'Paid')
            ->groupBy('userid')
            ->havingRaw('SUM(total) > 1000');
    })
    ->get();

// Exists subquery
$clientsWithOrders = Capsule::table('tblclients')
    ->whereExists(function ($query) {
        $query->from('tblorders')
            ->whereColumn('userid', 'tblclients.id');
    })
    ->get();
```

## Eloquent Models

```php
<?php
use Illuminate\Database\Eloquent\Model;

class Client extends Model
{
    protected $table = 'tblclients';
    protected $primaryKey = 'id';
    public $timestamps = false;
    
    protected $fillable = [
        'firstname',
        'lastname',
        'email',
        'companyname',
    ];
    
    // Relationships
    public function invoices()
    {
        return $this->hasMany(Invoice::class, 'userid');
    }
    
    public function services()
    {
        return $this->hasMany(Service::class, 'userid');
    }
    
    // Scopes
    public function scopeActive($query)
    {
        return $query->where('status', 'Active');
    }
    
    public function scopeByCountry($query, string $country)
    {
        return $query->where('country', $country);
    }
}

// Usage
$activeClients = Client::active()->get();
$usClients = Client::active()->byCountry('US')->get();
```

## Raw Expressions

```php
<?php
// Raw select
$clients = Capsule::table('tblclients')
    ->selectRaw("CONCAT(firstname, ' ', lastname) as full_name")
    ->selectRaw("CASE WHEN status = 'Active' THEN 1 ELSE 0 END as is_active")
    ->get();

// Raw where
$clients = Capsule::table('tblclients')
    ->whereRaw("YEAR(created_at) = YEAR(CURRENT_DATE)")
    ->whereRaw("MONTH(created_at) = MONTH(CURRENT_DATE)")
    ->get();

// Raw order by
$clients = Capsule::table('tblclients')
    ->orderByRaw("FIELD(status, 'Active', 'Suspended', 'Inactive')")
    ->get();

// Raw updates
Capsule::table('tblclients')
    ->where('id', $id)
    ->update([
        'status' => 'Active',
        'updated_at' => Capsule::raw('NOW()'),
    ]);
```

## Pagination

```php
<?php
// Simple pagination
$clients = Capsule::table('tblclients')
    ->paginate(25);

// With query builder
$clients = Capsule::table('tblclients')
    ->where('status', 'Active')
    ->orderBy('created_at', 'desc')
    ->paginate(50);

// Get specific page
$clients = Capsule::table('tblclients')
    ->paginate(25, ['*'], 'page', 3);

// Manual pagination
$offset = ($page - 1) * $perPage;
$clients = Capsule::table('tblclients')
    ->offset($offset)
    ->limit($perPage)
    ->get();
```

## Transactions

```php
<?php
use WHMCS\Database\Capsule;

// Transaction
Capsule::transaction(function () {
    $clientId = Capsule::table('tblclients')->insertGetId([
        'firstname' => 'John',
        'lastname' => 'Doe',
        'email' => 'john@example.com',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    Capsule::table('mod_custom_data')->insert([
        'client_id' => $clientId,
        'extra_field' => 'value',
    ]);
    
    return $clientId;
});

// Manual transaction
$pdo = Capsule::connection()->getPdo();
$pdo->beginTransaction();

try {
    // Operations
    $pdo->commit();
} catch (Exception $e) {
    $pdo->rollBack();
    throw $e;
}
```

## Related Documentation

- [WHMCS Capsule Schema](/docs/whmcs-capsule-schema.md)
- [WHMCS Eloquent Models](/docs/whmcs-eloquent-models.md)