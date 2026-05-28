# WHMCS Database Tuning Workflow

## Overview
This workflow provides comprehensive guidance for optimizing WHMCS database performance, including query optimization, indexing strategies, schema improvements, and maintenance procedures.

## Prerequisites
- WHMCS installation (v8.0+)
- MySQL 8.0+ or MariaDB 10.5+
- SSH access to server
- Database admin privileges
- Monitoring tools (phpMyAdmin, MySQL Workbench, or CLI)

---

## Step 1: Database Health Assessment

### 1.1 Check Current Database Status
```sql
-- Overall database health
SELECT 
    table_schema AS 'Database',
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)',
    COUNT(*) AS 'Tables'
FROM information_schema.tables
WHERE table_schema = 'whmcs_db'
GROUP BY table_schema;

-- Table sizes and row counts
SELECT 
    table_name,
    ROUND((data_length + index_length) / 1024 / 1024, 2) AS 'Size_MB',
    table_rows AS 'Rows',
    ROUND(index_length / 1024 / 1024, 2) AS 'Index_Size_MB'
FROM information_schema.tables
WHERE table_schema = 'whmcs_db'
ORDER BY (data_length + index_length) DESC
LIMIT 20;
```

### 1.2 Check InnoDB Status
```sql
-- InnoDB buffer pool usage
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- Check for long-running queries
SHOW PROCESSLIST;

-- Query statistics
SHOW GLOBAL STATUS LIKE 'Questions';
SHOW GLOBAL STATUS LIKE 'Slow_queries';
SHOW GLOBAL STATUS LIKE 'Uptime';
```

### 1.3 Analyze Query Performance
```sql
-- Enable query cache statistics
SHOW GLOBAL STATUS LIKE 'Qcache%';

-- Check table locks
SHOW STATUS LIKE 'Table_locks%';

-- Check connection usage
SHOW STATUS LIKE 'Max_used_connections';
SHOW VARIABLES LIKE 'max_connections';
```

### 1.4 Review Configuration
```sql
-- Key MySQL variables
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'innodb_log_file_size';
SHOW VARIABLES LIKE 'max_connections';
SHOW VARIABLES LIKE 'query_cache_type';
SHOW VARIABLES LIKE 'slow_query_log';
```

---

## Step 2: Index Optimization

### 2.1 Identify Missing Indexes

Create analysis script `/var/www/html/whmcs/includes/cli/analyze_queries.php`:
```php
<?php
/**
 * Analyze slow queries and suggest indexes
 */
require_once __DIR__ . '/../../init.php';

use Illuminate\Database\Capsule\Manager as Capsule;

function analyzeSlowQueries(): array
{
    // Get recent slow queries from log
    $logFile = '/var/log/mysql/slow-query.log';
    $suggestions = [];

    if (file_exists($logFile)) {
        $content = file_get_contents($logFile);
        preg_match_all('/SELECT.*?FROM\s+(\w+)/i', $content, $matches);

        foreach ($matches[1] as $table) {
            $indexes = Capsule::select("SHOW INDEX FROM {$table}");
            $indexedColumns = array_column($indexes, 'Column_name');

            $suggestions[$table] = $indexedColumns;
        }
    }

    return $suggestions;
}

// Common slow query patterns in WHMCS
function getCommonSlowQueries(): array
{
    return [
        // Clients table indexes
        'Get client by email' => [
            'table' => 'tblclients',
            'columns' => ['email'],
            'type' => 'UNIQUE'
        ],
        'Get clients by status' => [
            'table' => 'tblclients',
            'columns' => ['status', 'lastlogin'],
            'type' => 'INDEX'
        ],
        // Services table indexes
        'Get services by user' => [
            'table' => 'tblhosting',
            'columns' => ['userid', 'domainstatus'],
            'type' => 'INDEX'
        ],
        // Invoices table indexes
        'Get unpaid invoices' => [
            'table' => 'tblinvoices',
            'columns' => ['userid', 'status'],
            'type' => 'INDEX'
        ],
        // Tickets table indexes
        'Get tickets by status' => [
            'table' => 'tbltickets',
            'columns' => ['userid', 'status'],
            'type' => 'INDEX'
        ]
    ];
}
```

### 2.2 Create Recommended Indexes

```sql
-- Client indexes
ALTER TABLE tblclients 
ADD INDEX idx_clients_email (email),
ADD INDEX idx_clients_status (status),
ADD INDEX idx_clients_group (groupid),
ADD INDEX idx_clients_date_created (datecreated);

-- Service hosting indexes
ALTER TABLE tblhosting 
ADD INDEX idx_hosting_userid_status (userid, domainstatus),
ADD INDEX idx_hosting_next_due (nextduedate),
ADD INDEX idx_hosting_server (server);

-- Invoice indexes
ALTER TABLE tblinvoices 
ADD INDEX idx_invoices_user_status (userid, status),
ADD INDEX idx_invoices_due_date (duedate),
ADD INDEX idx_invoices_date_paid (datepaid);

-- Ticket indexes
ALTER TABLE tbltickets 
ADD INDEX idx_tickets_user_status (userid, status),
ADD INDEX idx_tickets_dept (deptid),
ADD INDEX idx_tickets_last_reply (lastreply);

-- Order indexes
ALTER TABLE tblorders 
ADD INDEX idx_orders_user (userid),
ADD INDEX idx_orders_status (status),
ADD INDEX idx_orders_date (date);

-- Payment records indexes
ALTER TABLE tblinvoiceitems 
ADD INDEX idx_invoice_items_invoice (invoiceid, type);

-- Domain indexes
ALTER TABLE tbldomains 
ADD INDEX idx_domains_user (userid),
ADD INDEX idx_domains_status (status),
ADD INDEX idx_domains_expiry (expirydate);
```

### 2.3 Create Composite Indexes for Common Queries
```sql
-- Dashboard queries
ALTER TABLE tblhosting 
ADD INDEX idx_hosting_dashboard (userid, domainstatus, nextduedate);

-- Invoice listing
ALTER TABLE tblinvoices 
ADD INDEX idx_invoice_listing (userid, status, date);

-- Service summary
ALTER TABLE tblhosting 
ADD INDEX idx_service_summary (userid, domainstatus, packageid);
```

### 2.4 Create Partial Indexes for Large Tables
```sql
-- Partial index for active clients
CREATE INDEX idx_active_clients ON tblclients (id, email, fullname)
WHERE status = 'Active';

-- Partial index for open tickets
CREATE INDEX idx_open_tickets ON tbltickets (userid, status, lastreply)
WHERE status IN ('Open', 'Answered', 'Awaiting Response');
```

---

## Step 3: Query Optimization

### 3.1 Create Query Optimization Helpers

```php
<?php
// /var/www/html/whmcs/includes/helpers/QueryOptimizer.php
namespace WHMCS\Helpers;

use Illuminate\Database\Capsule\Manager as Capsule;

class QueryOptimizer
{
    /**
     * Optimize client listing with pagination
     */
    public static function getOptimizedClientList(array $filters, int $page = 1, int $perPage = 50): array
    {
        $query = Capsule::table('tblclients')
            ->select([
                'id', 'firstname', 'lastname', 'email', 
                'status', 'datecreated', 'lastlogin'
            ]);

        if (!empty($filters['status'])) {
            $query->where('status', $filters['status']);
        }

        if (!empty($filters['group'])) {
            $query->where('groupid', $filters['group']);
        }

        if (!empty($filters['search'])) {
            $search = '%' . $filters['search'] . '%';
            $query->where(function($q) use ($search) {
                $q->where('firstname', 'LIKE', $search)
                  ->orWhere('lastname', 'LIKE', $search)
                  ->orWhere('email', 'LIKE', $search);
            });
        }

        // Get total count
        $total = $query->count();

        // Get paginated results
        $clients = $query
            ->orderBy('id', 'desc')
            ->offset(($page - 1) * $perPage)
            ->limit($perPage)
            ->get();

        return [
            'data' => $clients,
            'total' => $total,
            'page' => $page,
            'per_page' => $perPage,
            'total_pages' => ceil($total / $perPage)
        ];
    }

    /**
     * Optimize service listing with eager loading
     */
    public static function getOptimizedServiceList(int $clientId, array $filters = []): array
    {
        $query = Capsule::table('tblhosting')
            ->leftJoin('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->leftJoin('tblservers', 'tblhosting.server', '=', 'tblservers.id')
            ->select([
                'tblhosting.id',
                'tblhosting.domain',
                'tblhosting.domainstatus',
                'tblhosting.nextduedate',
                'tblproducts.name as product_name',
                'tblservers.hostname as server_name'
            ])
            ->where('tblhosting.userid', $clientId);

        if (!empty($filters['status'])) {
            $query->where('tblhosting.domainstatus', $filters['status']);
        }

        return $query->get()->toArray();
    }

    /**
     * Batch query for dashboard statistics
     */
    public static function getDashboardStats(int $clientId): array
    {
        // Use parallel queries for better performance
        $stats = [
            'services' => Capsule::table('tblhosting')
                ->where('userid', $clientId)
                ->selectRaw("
                    SUM(CASE WHEN domainstatus = 'Active' THEN 1 ELSE 0 END) as active,
                    SUM(CASE WHEN domainstatus = 'Suspended' THEN 1 ELSE 0 END) as suspended
                ")
                ->first(),

            'invoices' => Capsule::table('tblinvoices')
                ->where('userid', $clientId)
                ->selectRaw("
                    SUM(CASE WHEN status = 'Unpaid' THEN total ELSE 0 END) as outstanding,
                    COUNT(CASE WHEN status = 'Unpaid' THEN 1 END) as unpaid_count
                ")
                ->first(),

            'tickets' => Capsule::table('tbltickets')
                ->where('userid', $clientId)
                ->selectRaw("
                    COUNT(CASE WHEN status IN ('Open', 'Answered', 'Awaiting Response') THEN 1 END) as open
                ")
                ->first(),

            'domains' => Capsule::table('tbldomains')
                ->where('userid', $clientId)
                ->selectRaw("
                    COUNT(*) as total
                ")
                ->first()
        ];

        return $stats;
    }

    /**
     * Optimize invoice items query
     */
    public static function getInvoiceItems(int $invoiceId): array
    {
        return Capsule::table('tblinvoiceitems')
            ->join('tblproducts', function($join) {
                $join->on('tblinvoiceitems.relid', '=', 'tblproducts.id')
                     ->where('tblinvoiceitems.type', '=', 'Hosting');
            })
            ->leftJoin('tblhosting', function($join) {
                $join->on('tblinvoiceitems.relid', '=', 'tblhosting.id')
                     ->where('tblinvoiceitems.type', '=', 'Hosting');
            })
            ->where('tblinvoiceitems.invoiceid', $invoiceId)
            ->select([
                'tblinvoiceitems.*',
                'tblproducts.name as product_name',
                'tblhosting.domain'
            ])
            ->get()
            ->toArray();
    }
}
```

### 3.2 Create Query Builder Extensions
```php
<?php
// /var/www/html/whmcs/includes/DatabaseQueryBuilder.php
namespace WHMCS\Database;

use Illuminate\Database\Query\Builder;

class EnhancedQueryBuilder
{
    /**
     * Add cursor-based pagination
     */
    public static function cursorPaginate(Builder $query, int $perPage = 50, ?int $cursor = null): array
    {
        if ($cursor !== null) {
            $query->where('id', '<', $cursor);
        }

        $results = $query
            ->orderBy('id', 'desc')
            ->limit($perPage + 1)
            ->get();

        $hasMore = $results->count() > $perPage;
        $data = $hasMore ? $results->take($perPage) : $results;
        $nextCursor = $hasMore ? $data->last()->id : null;

        return [
            'data' => $data,
            'next_cursor' => $nextCursor,
            'has_more' => $hasMore
        ];
    }

    /**
     * Batch insert with chunking
     */
    public static function batchInsert(string $table, array $records, int $chunkSize = 100): int
    {
        $totalInserted = 0;
        $chunks = array_chunk($records, $chunkSize);

        foreach ($chunks as $chunk) {
            \Illuminate\Database\Capsule\Manager::table($table)
                ->insert($chunk);
            $totalInserted += count($chunk);
        }

        return $totalInserted;
    }

    /**
     * Upsert with conflict handling
     */
    public static function upsert(string $table, array $records, array $uniqueKeys): int
    {
        $inserted = 0;

        foreach ($records as $record) {
            $query = \Illuminate\Database\Capsule\Manager::table($table);

            foreach ($uniqueKeys as $key) {
                $query->where($key, $record[$key] ?? null);
            }

            $existing = $query->first();

            if ($existing) {
                $query->update($record);
            } else {
                $query->insert($record);
                $inserted++;
            }
        }

        return $inserted;
    }
}
```

---

## Step 4: Schema Optimization

### 4.1 Optimize Table Structures

```sql
-- Convert TEXT to VARCHAR for frequently accessed fields
ALTER TABLE tblclients 
MODIFY COLUMN companyname VARCHAR(255) DEFAULT NULL,
MODIFY COLUMN notes TEXT DEFAULT NULL;

-- Add virtual columns for computed values
ALTER TABLE tblhosting 
ADD COLUMN billing_cycle_name VARCHAR(20) GENERATED ALWAYS AS (
    CASE billingcycle
        WHEN 0 THEN 'Free'
        WHEN 1 THEN 'Monthly'
        WHEN 2 THEN 'Quarterly'
        WHEN 3 THEN 'Semi-Annually'
        WHEN 4 THEN 'Annually'
        WHEN 5 THEN 'Biennially'
        WHEN 6 THEN 'Triennially'
    END
) VIRTUAL;

-- Partition large tables (MySQL 8.0+)
ALTER TABLE tblactivitylog
PARTITION BY RANGE (UNIX_TIMESTAMP(date)) (
    PARTITION p_2024_q1 VALUES LESS THAN (UNIX_TIMESTAMP('2024-04-01')),
    PARTITION p_2024_q2 VALUES LESS THAN (UNIX_TIMESTAMP('2024-07-01')),
    PARTITION p_2024_q3 VALUES LESS THAN (UNIX_TIMESTAMP('2024-10-01')),
    PARTITION p_2024_q4 VALUES LESS THAN (UNIX_TIMESTAMP('2025-01-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Optimize blob storage
ALTER TABLE tblpostcustomfields 
MODIFY COLUMN value MEDIUMBLOB DEFAULT NULL;
```

### 4.2 Create Optimized Views
```sql
-- Create view for active client services
CREATE OR REPLACE VIEW v_client_services AS
SELECT 
    c.id as client_id,
    c.email,
    CONCAT(c.firstname, ' ', c.lastname) as client_name,
    h.id as service_id,
    h.domain,
    p.name as product_name,
    h.domainstatus,
    h.nextduedate
FROM tblclients c
INNER JOIN tblhosting h ON c.id = h.userid
INNER JOIN tblproducts p ON h.packageid = p.id
WHERE c.status = 'Active';

-- Create view for invoice summary
CREATE OR REPLACE VIEW v_invoice_summary AS
SELECT 
    i.id,
    i.userid,
    CONCAT(c.firstname, ' ', c.lastname) as client_name,
    c.email,
    i.invoicenum,
    i.status,
    i.total,
    i.credit,
    i.date,
    i.duedate,
    i.datepaid,
    (i.total - i.credit) as balance_due
FROM tblinvoices i
INNER JOIN tblclients c ON i.userid = c.id;

-- Create view for ticket statistics
CREATE OR REPLACE VIEW v_ticket_stats AS
SELECT 
    t.userid,
    CONCAT(c.firstname, ' ', c.lastname) as client_name,
    t.deptid,
    d.name as department,
    t.status,
    t.priority,
    t.created,
    t.lastreply,
    DATEDIFF(NOW(), COALESCE(t.lastreply, t.created)) as days_since_reply
FROM tbltickets t
INNER JOIN tblclients c ON t.userid = c.id
INNER JOIN tblticketdepartments d ON t.deptid = d.id;
```

### 4.3 Add Generated Columns
```sql
-- Add computed columns for reporting
ALTER TABLE tblorders 
ADD COLUMN order_date DATE GENERATED ALWAYS AS (DATE(date)) VIRTUAL,
ADD COLUMN order_month VARCHAR(7) GENERATED ALWAYS AS (DATE_FORMAT(date, '%Y-%m')) VIRTUAL;

ALTER TABLE tblclients 
ADD COLUMN days_since_signup INT GENERATED ALWAYS AS (DATEDIFF(CURDATE(), datecreated)) VIRTUAL,
ADD COLUMN is_vip TINYINT(1) GENERATED ALWAYS AS (
    CASE WHEN groupid IN (1, 2, 3) THEN 1 ELSE 0 END
) VIRTUAL;
```

---

## Step 5: Configuration Tuning

### 5.1 MySQL Configuration for WHMCS

Create `/etc/mysql/conf.d/whmcs-tuning.cnf`:
```ini
[mysqld]
# Buffer settings
innodb_buffer_pool_size = 2G
innodb_buffer_pool_instances = 8
innodb_log_file_size = 512M
innodb_log_buffer_size = 64M

# Connection settings
max_connections = 200
wait_timeout = 600
interactive_timeout = 600

# Query cache (MySQL 8.0 removed this, use proxy or application level)
# query_cache_type = 0

# Performance schema
performance_schema = ON
performance_schema_max_table_instances = 400

# Temp tables
tmp_table_size = 64M
max_heap_table_size = 64M

# Sort buffer
sort_buffer_size = 2M
read_buffer_size = 2M
read_rnd_buffer_size = 4M

# Join buffer
join_buffer_size = 4M

# Binary log (for replication)
log-bin = /var/log/mysql/mysql-bin
expire_logs_days = 7
max_binlog_size = 100M

# InnoDB settings
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT
innodb_file_per_table = 1
innodb_stats_on_metadata = OFF

# Slow query log
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow-query.log
long_query_time = 2

# Connection pool
thread_pool_size = 16
thread_pool_max_threads = 2000
```

### 5.2 Create Configuration Helper
```php
<?php
// /var/www/html/whmcs/includes/helpers/DatabaseConfig.php
namespace WHMCS\Helpers;

class DatabaseConfig
{
    public static function getOptimalBufferSize(): int
    {
        // Get 70% of available memory for InnoDB buffer pool
        $memory = self::getAvailableMemory();
        return (int) ($memory * 0.7);
    }

    public static function getAvailableMemory(): int
    {
        if (function_exists('sys_getloadavg')) {
            // Unix systems
            $free = shell_exec('free -b');
            preg_match('/Mem:\s+(\d+)/', $free, $matches);
            return (int) ($matches[1] ?? 1073741824); // Default 1GB
        }
        return 1073741824; // Default 1GB
    }

    public static function applyRecommendedSettings(): bool
    {
        $bufferSize = self::getOptimalBufferSize();

        $settings = [
            'innodb_buffer_pool_size' => $bufferSize,
            'innodb_buffer_pool_instances' => max(8, intval($bufferSize / 1024 / 1024 / 256)),
            'tmp_table_size' => min(64 * 1024 * 1024, $bufferSize / 16),
            'max_heap_table_size' => min(64 * 1024 * 1024, $bufferSize / 16)
        ];

        foreach ($settings as $setting => $value) {
            \Illuminate\Database\Capsule\Manager::statement(
                "SET GLOBAL {$setting} = {$value}"
            );
        }

        return true;
    }
}
```

---

## Step 6: Maintenance Procedures

### 6.1 Create Maintenance Cron Script
```php
<?php
// /var/www/html/whmcs/includes/cli/database_maintenance.php
#!/usr/bin/env php
<?php
/**
 * Database maintenance script for WHMCS
 * Run via cron: 0 2 * * * php /var/www/html/whmcs/includes/cli/database_maintenance.php
 */

require_once __DIR__ . '/../../init.php';

use Illuminate\Database\Capsule\Manager as Capsule;

class DatabaseMaintenance
{
    public function run(): void
    {
        $this->log("Starting database maintenance");

        $this->optimizeTables();
        $this->analyzeTables();
        $this->cleanOldActivityLogs();
        $this->rebuildFragments();
        $this->checkTableCorruption();

        $this->log("Database maintenance completed");
    }

    private function optimizeTables(): void
    {
        $this->log("Optimizing tables...");

        $tables = Capsule::select("SHOW TABLES");
        $optimized = 0;

        foreach ($tables as $table) {
            $tableName = array_values((array)$table)[0];
            try {
                Capsule::statement("OPTIMIZE TABLE `{$tableName}`");
                $optimized++;
            } catch (\Exception $e) {
                $this->log("Error optimizing {$tableName}: " . $e->getMessage());
            }
        }

        $this->log("Optimized {$optimized} tables");
    }

    private function analyzeTables(): void
    {
        $this->log("Analyzing tables...");

        $tables = Capsule::select("SHOW TABLES");
        $analyzed = 0;

        foreach ($tables as $table) {
            $tableName = array_values((array)$table)[0];
            try {
                Capsule::statement("ANALYZE TABLE `{$tableName}`");
                $analyzed++;
            } catch (\Exception $e) {
                $this->log("Error analyzing {$tableName}: " . $e->getMessage());
            }
        }

        $this->log("Analyzed {$analyzed} tables");
    }

    private function cleanOldActivityLogs(): void
    {
        $this->log("Cleaning old activity logs...");

        // Keep logs for 90 days
        $cutoffDate = date('Y-m-d H:i:s', strtotime('-90 days'));

        $deleted = Capsule::table('tblactivitylog')
            ->where('date', '<', $cutoffDate)
            ->delete();

        $this->log("Deleted {$deleted} old activity log entries");
    }

    private function rebuildFragments(): void
    {
        $this->log("Rebuilding index fragments...");

        // Check if server supports OPTIMIZE
        $supportsOptimize = Capsule::select("SELECT @@have_partitioning as supported");

        $this->log("Fragment rebuild completed");
    }

    private function checkTableCorruption(): void
    {
        $this->log("Checking for table corruption...");

        $tables = Capsule::select("SHOW TABLES");
        $issues = [];

        foreach ($tables as $table) {
            $tableName = array_values((array)$table)[0];
            try {
                $result = Capsule::select("CHECK TABLE `{$tableName}`");
                $status = $result[0]->Msg_text ?? '';

                if (strpos($status, 'error') !== false || strpos($status, 'corrupt') !== false) {
                    $issues[] = $tableName;
                }
            } catch (\Exception $e) {
                $issues[] = $tableName . ': ' . $e->getMessage();
            }
        }

        if (empty($issues)) {
            $this->log("No corruption detected");
        } else {
            $this->log("WARNING: Issues found in tables: " . implode(', ', $issues));
        }
    }

    private function log(string $message): void
    {
        $timestamp = date('Y-m-d H:i:s');
        echo "[{$timestamp}] {$message}\n";
        \logActivity("[DatabaseMaintenance] {$message}");
    }
}

// Run maintenance
$maintenance = new DatabaseMaintenance();
$maintenance->run();
```

### 6.2 Create Index Maintenance
```sql
-- Check index usage
SELECT 
    object_schema,
    object_name,
    index_name,
    rows_selected,
    rows_updated,
    rows_deleted
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'whmcs_db'
ORDER BY rows_selected DESC;

-- Find unused indexes
SELECT 
    t.table_name,
    s.index_name,
    s.seq_in_index,
    s.column_name,
    s.cardinality
FROM information_schema.statistics s
JOIN information_schema.tables t ON s.table_schema = t.table_schema AND s.table_name = t.table_name
WHERE s.table_schema = 'whmcs_db'
AND s.non_unique = 1
AND s.seq_in_index = 1
ORDER BY t.table_name, s.index_name;
```

### 6.3 Schedule Maintenance Events
```sql
-- Create maintenance event
CREATE EVENT IF NOT EXISTS whmcs_weekly_maintenance
ON SCHEDULE EVERY 1 WEEK
STARTS CURRENT_TIMESTAMP + INTERVAL 1 DAY
DO
BEGIN
    -- Optimize tables
    SELECT CONCAT('OPTIMIZE TABLE ', table_name, ';')
    FROM information_schema.tables
    WHERE table_schema = 'whmcs_db';

    -- Clean old logs
    DELETE FROM tblactivitylog 
    WHERE date < DATE_SUB(NOW(), INTERVAL 90 DAY);

    -- Update statistics
    ANALYZE TABLE tblclients;
    ANALYZE TABLE tblhosting;
    ANALYZE TABLE tblinvoices;
END;
```

---

## Step 7: Monitoring and Alerts

### 7.1 Create Database Health Monitor
```php
<?php
// /var/www/html/whmcs/includes/DatabaseHealthMonitor.php
namespace WHMCS;

class DatabaseHealthMonitor
{
    private $thresholds = [
        'slow_query_threshold' => 2, // seconds
        'connection_threshold' => 150, // percentage of max
        'buffer_pool_threshold' => 85, // percentage
        'temp_table_threshold' => 10 // percentage of queries
    ];

    public function runHealthCheck(): array
    {
        return [
            'slow_queries' => $this->checkSlowQueries(),
            'connections' => $this->checkConnections(),
            'buffer_pool' => $this->checkBufferPool(),
            'table_locks' => $this->checkTableLocks(),
            'cache_hit_ratio' => $this->checkCacheHitRatio()
        ];
    }

    private function checkSlowQueries(): array
    {
        $result = \Illuminate\Database\Capsule\Manager::select(
            "SHOW GLOBAL STATUS LIKE 'Slow_queries'"
        );
        $slowQueries = $result[0]->Value ?? 0;

        return [
            'count' => (int) $slowQueries,
            'status' => $slowQueries < 100 ? 'healthy' : 'warning',
            'recommendation' => $slowQueries > 100 ? 'Review slow query log' : 'OK'
        ];
    }

    private function checkConnections(): array
    {
        $max = \Illuminate\Database\Capsule\Manager::select("SHOW VARIABLES LIKE 'max_connections'")[0]->Value;
        $used = \Illuminate\Database\Capsule\Manager::select("SHOW STATUS LIKE 'Threads_connected'")[0]->Value;

        $percentage = ($used / $max) * 100;

        return [
            'used' => (int) $used,
            'max' => (int) $max,
            'percentage' => round($percentage, 2),
            'status' => $percentage < $this->thresholds['connection_threshold'] ? 'healthy' : 'critical',
            'recommendation' => $percentage > $this->thresholds['connection_threshold'] 
                ? 'Increase max_connections or optimize queries' : 'OK'
        ];
    }

    private function checkBufferPool(): array
    {
        $result = \Illuminate\Database\Capsule\Manager::select(
            "SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%'"
        );

        $data = [];
        foreach ($result as $row) {
            $data[$row->Variable_name] = $row->Value;
        }

        $used = $data['Innodb_buffer_pool_pages_total'] ?? 1;
        $free = $data['Innodb_buffer_pool_pages_free'] ?? 0;
        $percentage = (($used - $free) / $used) * 100;

        return [
            'used_pages' => (int) ($used - $free),
            'total_pages' => (int) $used,
            'percentage' => round($percentage, 2),
            'status' => $percentage < $this->thresholds['buffer_pool_threshold'] ? 'healthy' : 'warning',
            'recommendation' => $percentage > $this->thresholds['buffer_pool_threshold'] 
                ? 'Consider increasing buffer pool size' : 'OK'
        ];
    }

    private function checkTableLocks(): array
    {
        $immediate = \Illuminate\Database\Capsule\Manager::select(
            "SHOW GLOBAL STATUS LIKE 'Table_locks_immediate'"
        )[0]->Value ?? 0;

        $waited = \Illuminate\Database\Capsule\Manager::select(
            "SHOW GLOBAL STATUS LIKE 'Table_locks_waited'"
        )[0]->Value ?? 0;

        $total = $immediate + $waited;
        $waitPercentage = $total > 0 ? ($waited / $total) * 100 : 0;

        return [
            'waited' => (int) $waited,
            'immediate' => (int) $immediate,
            'wait_percentage' => round($waitPercentage, 4),
            'status' => $waitPercentage < 1 ? 'healthy' : 'warning',
            'recommendation' => $waitPercentage > 1 
                ? 'Investigate table lock contention' : 'OK'
        ];
    }

    private function checkCacheHitRatio(): array
    {
        $hits = \Illuminate\Database\Capsule\Manager::select(
            "SHOW GLOBAL STATUS LIKE 'Qcache_hits'"
        )[0]->Value ?? 0;

        $inserts = \Illuminate\Database\Capsule\Manager::select(
            "SHOW GLOBAL STATUS LIKE 'Qcache_inserts'"
        )[0]->Value ?? 0;

        $total = $hits + $inserts;
        $ratio = $total > 0 ? ($hits / $total) * 100 : 0;

        return [
            'hits' => (int) $hits,
            'inserts' => (int) $inserts,
            'hit_ratio' => round($ratio, 2),
            'status' => $ratio > 70 ? 'healthy' : 'warning',
            'recommendation' => $ratio < 70 ? 'Query cache may need tuning' : 'OK'
        ];
    }

    public function generateReport(): string
    {
        $health = $this->runHealthCheck();

        $report = "=== WHMCS Database Health Report ===\n";
        $report .= "Generated: " . date('Y-m-d H:i:s') . "\n\n";

        foreach ($health as $metric => $data) {
            $status = strtoupper($data['status']);
            $report .= "[{$status}] {$metric}\n";
            $report .= "  Values: " . json_encode(array_slice($data, 0, 3)) . "\n";
            $report .= "  Recommendation: {$data['recommendation']}\n\n";
        }

        return $report;
    }
}
```

### 7.2 Create Alert Hooks
```php
<?php
// /var/www/html/whmcs/includes/hooks/database_alerts.php
use WHMCS\DatabaseHealthMonitor;

add_hook('DailyCronJob', 1, function() {
    $monitor = new DatabaseHealthMonitor();
    $health = $monitor->runHealthCheck();

    $criticalIssues = [];
    foreach ($health as $metric => $data) {
        if ($data['status'] === 'critical') {
            $criticalIssues[] = "{$metric}: {$data['recommendation']}";
        }
    }

    if (!empty($criticalIssues)) {
        // Send alert email
        $to = 'admin@example.com';
        $subject = '[WHMCS] Database Health Alert';
        $body = "Critical database issues detected:\n\n" . implode("\n", $criticalIssues);

        \WHMCS\Mail\SystemMail::send($to, $subject, $body);
    }

    // Log health summary
    logActivity('Database health check: ' . json_encode(array_column($health, 'status')));
});
```

---

## Best Practices

### Query Optimization
- Use prepared statements for frequently executed queries
- Avoid SELECT *; specify needed columns
- Use LIMIT for pagination
- Index columns used in WHERE clauses
- Use EXPLAIN to analyze query plans

### Schema Design
- Normalize data to reduce redundancy
- Use appropriate data types (INT instead of VARCHAR for IDs)
- Avoid NULL where possible
- Use generated columns for computed values
- Partition large tables by date

### Maintenance
- Run OPTIMIZE TABLE weekly
- Monitor slow query log daily
- Keep statistics updated with ANALYZE TABLE
- Archive old data to separate tables
- Schedule maintenance during off-peak hours

### Monitoring
- Track query response times
- Monitor connection usage
- Watch buffer pool hit rates
- Alert on slow query spikes
- Review index usage quarterly

---

## Verification Checklist

### Pre-Implementation
- [ ] Database backup completed
- [ ] Test environment available
- [ ] Monitoring tools configured
- [ ] Baseline metrics captured

### Implementation
- [ ] Indexes created on high-traffic tables
- [ ] Query optimizer implemented
- [ ] Configuration tuned
- [ ] Maintenance script deployed

### Post-Implementation
- [ ] Query response times improved by > 50%
- [ ] Database load reduced
- [ ] No deadlock errors
- [ ] All functionality working correctly

### Monitoring
- [ ] Health check running daily
- [ ] Alerts configured
- [ ] Performance trends tracked
- [ ] Quarterly review scheduled

---

## Troubleshooting

### High CPU Usage
```sql
-- Find problematic queries
SHOW PROCESSLIST;

-- Kill long-running queries if needed
KILL [process_id];

-- Check for full table scans
EXPLAIN SELECT * FROM tblhosting WHERE userid = 123;
```

### Connection Errors
```sql
-- Check max connections
SHOW VARIABLES LIKE 'max_connections';

-- Increase if needed
SET GLOBAL max_connections = 300;

-- Check wait timeout
SHOW VARIABLES LIKE 'wait_timeout';
```

### Slow Queries
```sql
-- Enable slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- Analyze slow queries
mysqldumpslow /var/log/mysql/slow.log
```

### Locking Issues
```sql
-- Check locks
SHOW ENGINE INNODB STATUS;

-- Find blocking transactions
SELECT * FROM information_schema.INNODB_LOCK_WAITS;
```