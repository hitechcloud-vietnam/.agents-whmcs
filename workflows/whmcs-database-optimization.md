# WHMCS Database Optimization Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Optimize WHMCS database performance and ensure efficient queries.

## Database Analysis

### 1. Identify Slow Queries
```sql
-- Enable slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow-queries.log';
SET GLOBAL long_query_time = 2;

-- Analyze query patterns
SELECT
    DIGEST_TEXT AS query,
    COUNT_STAR AS executions,
    SUM_TIMER_WAIT / 1000000000000 AS total_time,
    AVG_TIMER_WAIT / 1000000000000 AS avg_time
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

### 2. Table Analysis
```php
<?php
// Analyze WHMCS tables
function analyzeTables(): void {
    $importantTables = [
        'tblhosting', 'tblhostingaddons', 'tblorders',
        'tblinvoices', 'tblinvoiceitems', 'tblclients',
        'tbltickets', 'tbldomains', 'tblactivitylog'
    ];

    foreach ($importantTables as $table) {
        Capsule::statement("ANALYZE TABLE {$table}");
    }
}

// Run during maintenance window
add_hook('AdminAreaPage', 1, function() {
    if (isMaintenanceMode()) {
        analyzeTables();
    }
});
```

## Index Optimization

### Common Missing Indexes
```sql
-- Add indexes for common queries
ALTER TABLE tblhosting ADD INDEX idx_userid (userid);
ALTER TABLE tblhosting ADD INDEX idx_domainstatus (domainstatus);
ALTER TABLE tblorders ADD INDEX idx_userid_date (userid, date);
ALTER TABLE tblinvoices ADD INDEX idx_userid_status (userid, status);
ALTER TABLE tblactivitylog ADD INDEX idx_date (date);
```

### Module Table Indexes
```php
// In module activate()
Capsule::schema()->create('mod_mymodule_data', function($t) {
    $t->increments('id');
    $t->integer('service_id')->unsigned();
    $t->string('status', 20);
    $t->timestamp('created_at');

    // Always add indexes for foreign keys and query fields
    $t->index('service_id');
    $t->index('status');
    $t->index(['service_id', 'status']); // Composite index
});

// For existing tables
Capsule::statement('CREATE INDEX idx_service_status ON mod_mymodule_data(service_id, status)');
```

## Query Optimization

### 1. Use Eloquent Instead of Raw SQL
```php
// Bad - Raw SQL
$results = Capsule::select("SELECT * FROM tblhosting WHERE userid = ? AND domainstatus = ?", [$userId, 'Active']);

// Good - Eloquent
$results = Capsule::table('tblhosting')
    ->where('userid', $userId)
    ->where('domainstatus', 'Active')
    ->get();

// Best - Select only needed columns
$results = Capsule::table('tblhosting')
    ->where('userid', $userId)
    ->where('domainstatus', 'Active')
    ->select('id', 'domain', 'nextduedate', 'billingcycle')
    ->get();
```

### 2. Pagination
```php
// Instead of getting all records
$page = (int)($_GET['page'] ?? 1);
$perPage = 50;

$services = Capsule::table('tblhosting')
    ->where('userid', $userId)
    ->orderBy('id', 'desc')
    ->paginate($perPage);

// Or using limit/offset
$services = Capsule::table('tblhosting')
    ->where('userid', $userId)
    ->limit($perPage)
    ->offset(($page - 1) * $perPage)
    ->get();
```

### 3. Avoid N+1 Queries
```php
// Bad
foreach ($services as $service) {
    $client = Capsule::table('tblclients')
        ->where('id', $service->userid)
        ->first();
    echo $client->email;
}

// Good - Join or eager load
$services = Capsule::table('tblhosting')
    ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
    ->where('tblhosting.userid', $userId)
    ->select('tblhosting.*', 'tblclients.email')
    ->get();

foreach ($services as $service) {
    echo $service->email; // Already available
}
```

## Caching Strategy

### Query Results Caching
```php
<?php
class CacheService {
    private static function getCache(string $key): mixed {
        return Capsule::table('mod_cache')
            ->where('cache_key', $key)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->value('cache_value');
    }

    private static function setCache(string $key, $value, int $ttl = 3600): void {
        Capsule::table('mod_cache')->updateOrInsert(
            ['cache_key' => $key],
            [
                'cache_value' => serialize($value),
                'expires_at' => date('Y-m-d H:i:s', time() + $ttl),
            ]
        );
    }

    public static function getUserServices(int $userId, bool $useCache = true): array {
        $cacheKey = "user_services_{$userId}";

        if ($useCache && ($cached = self::getCache($cacheKey))) {
            return unserialize($cached);
        }

        $services = Capsule::table('tblhosting')
            ->where('userid', $userId)
            ->get()
            ->toArray();

        self::setCache($cacheKey, $services, 1800); // 30 min cache

        return $services;
    }
}
```

## Table Maintenance

### Regular Optimization
```bash
#!/bin/bash
# optimize-db.sh
MYSQL_USER="whmcs_user"
MYSQL_PASS="password"
DATABASE="whmcs_db"

# Optimize tables
mysql -u${MYSQL_USER} -p${MYQL_PASS} ${DATABASE} -e "
    OPTIMIZE TABLE tblhosting;
    OPTIMIZE TABLE tblorders;
    OPTIMIZE TABLE tblactivitylog;
    OPTIMIZE TABLE tblticketlog;
"

# Check table status
mysql -u${MYSQL_USER} -p${MYSQL_PASS} ${DATABASE} -e "
    SELECT TABLE_NAME, Data_free, Index_length
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA = '${DATABASE}'
    AND Data_free > 1000000
    ORDER BY Data_free DESC;
"
```

### Cron Job for Cleanup
```php
add_hook('DailyCronJob', 1, function() {
    // Clean old cache entries
    Capsule::table('mod_cache')
        ->where('expires_at', '<', date('Y-m-d H:i:s'))
        ->delete();

    // Clean old logs (keep 90 days)
    Capsule::table('mod_mymodule_logs')
        ->where('created_at', '<', date('Y-m-d H:i:s', strtotime('-90 days')))
        ->delete();

    // Vacuum old activity (keep 30 days)
    Capsule::table('tblactivitylog')
        ->where('date', '<', date('Y-m-d H:i:s', strtotime('-30 days')))
        ->delete();
});
```

## Monitoring

### Performance Dashboard Query
```php
<?php
function getDatabaseStats(): array {
    $tables = [
        'tblhosting' => 'Hosting Services',
        'tblorders' => 'Orders',
        'tblinvoices' => 'Invoices',
        'tblclients' => 'Clients',
        'tbltickets' => 'Tickets',
    ];

    $stats = [];
    foreach ($tables as $table => $label) {
        $stats[$table] = Capsule::select("
            SELECT
                TABLE_ROWS as rows,
                ROUND(DATA_LENGTH / 1024 / 1024, 2) as data_mb,
                ROUND(INDEX_LENGTH / 1024 / 1024, 2) as index_mb
            FROM information_schema.TABLES
            WHERE TABLE_SCHEMA = DATABASE()
            AND TABLE_NAME = ?
        ", [$table])[0] ?? null;
    }

    return $stats;
}
```

## Checklist

```
Analysis:
□ Identify slow queries
□ Analyze table sizes
□ Check index usage
□ Review query execution plans

Optimization:
□ Add missing indexes
□ Optimize existing indexes
□ Implement query caching
□ Add pagination to large result sets
□ Fix N+1 query problems

Maintenance:
□ Schedule regular OPTIMIZE TABLE
□ Set up log rotation
□ Configure slow query log
□ Automate cleanup jobs

Monitoring:
□ Set up query monitoring
□ Track slow query count
□ Monitor table sizes
□ Alert on performance degradation
```

---

**Related Skills:**
- whmcs-performance-optimization
- whmcs-database-design
- whmcs-cron-automation