# WHMCS Database Optimization

## Overview

Database optimization improves WHMCS performance through query optimization, indexing, and configuration tuning.

## Query Optimization

### Avoid SELECT *

```php
<?php
// BAD - Selects all columns
$clients = Capsule::table('tblclients')->get();

// GOOD - Select only needed columns
$clients = Capsule::table('tblclients')
    ->select('id', 'email', 'firstname', 'lastname', 'status')
    ->get();
```

### Use Indexes

```php
<?php
// Create index on frequently queried columns
Capsule::schema()->table('tblhosting', function ($t) {
    $t->index(['userid', 'domainstatus'], 'idx_user_status');
    $t->index(['nextduedate', 'domainstatus'], 'idx_due_status');
});

// Create composite index for complex queries
Capsule::schema()->table('tblinvoices', function ($t) {
    $t->index(['userid', 'status', 'duedate'], 'idx_client_status_due');
});
```

### Limit Results

```php
<?php
// Always limit results
$recentInvoices = Capsule::table('tblinvoices')
    ->where('userid', $userId)
    ->orderBy('id', 'desc')
    ->limit(100)  // Always set a limit
    ->get();

// Use pagination
$page = 1;
$perPage = 25;
$offset = ($page - 1) * $perPage;

$clients = Capsule::table('tblclients')
    ->offset($offset)
    ->limit($perPage)
    ->get();
```

### Optimize Joins

```php
<?php
// BAD - Unnecessary join
$invoices = Capsule::table('tblinvoices')
    ->join('tblclients', 'tblinvoices.userid', '=', 'tblclients.id')
    ->where('tblinvoices.id', $invoiceId)
    ->select('tblinvoices.*')  // Only need invoice data
    ->first();

// GOOD - No join needed if client data not used
$invoice = Capsule::table('tblinvoices')
    ->where('id', $invoiceId)
    ->first();
```

## Caching Strategies

### Cache Expensive Queries

```php
<?php
function getDashboardStats(): array
{
    return Cache::remember('dashboard_stats', 300, function () {
        return [
            'total_clients' => Capsule::table('tblclients')->count(),
            'active_services' => Capsule::table('tblhosting')
                ->where('domainstatus', 'Active')
                ->count(),
            'pending_orders' => Capsule::table('tblorders')
                ->where('status', 'Pending')
                ->count(),
            'monthly_revenue' => Capsule::table('tblinvoices')
                ->where('status', 'Paid')
                ->whereMonth('datepaid', date('m'))
                ->whereYear('datepaid', date('Y'))
                ->sum('total'),
        ];
    });
}
```

### Cache Invalidation

```php
<?php
function invalidateDashboardCache(): void
{
    Cache::forget('dashboard_stats');
}

// Invalidate on data changes
add_hook('InvoicePaid', 1, function($vars) {
    invalidateDashboardCache();
});

add_hook('ClientCreated', 1, function($vars) {
    invalidateDashboardCache();
});
```

## Connection Pooling

```php
<?php
// Database configuration in configuration.php
$db_host = 'localhost';
$db_username = 'whmcs_user';
$db_password = 'password';
$db_name = 'whmcs_db';

// Connection options for pooling
Capsule::connection()->setFetchMode(PDO::FETCH_OBJ);

// Enable persistent connections
Capsule::connection()->getPdo()->setAttribute(PDO::ATTR_PERSISTENT, true);
```

## Index Maintenance

```php
<?php
// Check table indexes
function showIndexes(string $table): array
{
    return Capsule::select("SHOW INDEXES FROM {$table}");
}

// Analyze table
function analyzeTable(string $table): void
{
    Capsule::statement("ANALYZE TABLE {$table}");
}

// Optimize table
function optimizeTable(string $table): void
{
    Capsule::statement("OPTIMIZE TABLE {$table}");
}

// Repair table (if needed)
function repairTable(string $table): void
{
    Capsule::statement("REPAIR TABLE {$table}");
}
```

## Query Analysis

```php
<?php
class QueryAnalyzer
{
    public static function explain(string $sql, array $bindings = []): array
    {
        $query = "EXPLAIN " . $sql;
        return Capsule::select($query, $bindings);
    }
    
    public static function analyzeSlowQueries(int $minTime = 1): array
    {
        // Enable slow query log (run once in database)
        // SET GLOBAL slow_query_log = 'ON';
        // SET GLOBAL slow_query_log_file = '/path/to/slow.log';
        // SET GLOBAL long_query_time = 1;
        
        return Capsule::select(
            "SELECT * FROM mysql.slow_log ORDER BY start_time DESC LIMIT 100"
        );
    }
    
    public static function checkUnusedIndexes(string $table): array
    {
        return Capsule::select(
            "SELECT * FROM sys.schema_unused_indexes WHERE schema = DATABASE() AND table_name = ?",
            [$table]
        );
    }
}
```

## Batch Operations

```php
<?php
// Process large datasets in batches
function processLargeDataset(callable $processor, int $batchSize = 1000): void
{
    $lastId = 0;
    
    while (true) {
        $records = Capsule::table('tblrecords')
            ->where('id', '>', $lastId)
            ->orderBy('id')
            ->limit($batchSize)
            ->get();
        
        if ($records->isEmpty()) {
            break;
        }
        
        foreach ($records as $record) {
            $processor($record);
            $lastId = $record->id;
        }
        
        // Free memory
        gc_collect_cycles();
    }
}

// Usage
processLargeDataset(function ($record) {
    processRecord($record);
}, 500);
```

## Best Practices

1. **Index wisely** - Add indexes for WHERE, JOIN, ORDER BY columns
2. **Avoid functions on columns** - `WHERE YEAR(date) = 2024` prevents index use
3. **Use appropriate data types** - Use INT over VARCHAR for IDs
4. **Partition large tables** - Consider partitioning by date
5. **Monitor slow queries** - Enable query logging
6. **Use EXPLAIN** - Analyze query execution plans

## Related Documentation

- [WHMCS Capsule Queries](/docs/whmcs-capsule-queries.md)
- [WHMCS Cache Layer](/docs/whmcs-cache-layer.md)