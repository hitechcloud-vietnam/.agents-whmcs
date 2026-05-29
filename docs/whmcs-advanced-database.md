# WHMCS Advanced Database

Complete guide to database optimization and advanced techniques.

## Overview

Master database operations for WHMCS.

## Schema Optimization

### Index Management

```php
<?php
/**
 * Database index manager
 */
class IndexManager
{
    /**
     * Create optimized indexes
     */
    public function createOptimizedIndexes(): void
    {
        // Composite index for common queries
        Capsule::statement('
            CREATE INDEX idx_hosting_status_due 
            ON tblhosting(domainstatus, nextduedate)
        ');
        
        // Index for client lookups
        Capsule::statement('
            CREATE INDEX idx_clients_email_status 
            ON tblclients(email, status)
        ');
        
        // Index for invoice queries
        Capsule::statement('
            CREATE INDEX idx_invoices_status_date 
            ON tblinvoices(status, date)
        ');
        
        // Fulltext index for search
        Capsule::statement('
            ALTER TABLE tbltickets 
            ADD FULLTEXT INDEX idx_ticket_search(subject, message)
        ');
    }
    
    /**
     * Analyze table for optimization
     */
    public function analyzeTable(string $table): array
    {
        Capsule::statement("ANALYZE TABLE {$table}");
        
        $result = Capsule::select("SHOW TABLE STATUS LIKE '{$table}'");
        
        return [
            'name' => $result[0]->Name ?? $table,
            'engine' => $result[0]->Engine ?? 'unknown',
            'rows' => $result[0]->Rows ?? 0,
            'data_length' => $result[0]->Data_length ?? 0,
            'index_length' => $result[0]->Index_length ?? 0,
            'auto_increment' => $result[0]->Auto_increment ?? 0,
        ];
    }
    
    /**
     * Check index usage
     */
    public function checkIndexUsage(string $table): array
    {
        $indexes = Capsule::select("SHOW INDEX FROM {$table}");
        
        $usage = [];
        foreach ($indexes as $index) {
            $usage[] = [
                'key_name' => $index->Key_name,
                'column' => $index->Column_name,
                'unique' => !$index->Non_unique,
            ];
        }
        
        return $usage;
    }
}
```

## Query Optimization

### Advanced Query Patterns

```php
<?php
/**
 * Advanced query patterns
 */
class AdvancedQueries
{
    /**
     * Window functions for ranking
     */
    public function getTopClientsByRevenue(int $limit = 10): array
    {
        return Capsule::select("
            SELECT 
                c.id,
                c.email,
                c.firstname,
                c.lastname,
                SUM(i.total) as total_revenue,
                RANK() OVER (ORDER BY SUM(i.total) DESC) as revenue_rank
            FROM tblclients c
            JOIN tblinvoices i ON c.id = i.userid
            WHERE i.status = 'Paid'
            GROUP BY c.id, c.email, c.firstname, c.lastname
            ORDER BY total_revenue DESC
            LIMIT ?
        ", [$limit]);
    }
    
    /**
     * Running total
     */
    public function getRevenueRunningTotal(): array
    {
        return Capsule::select("
            SELECT 
                date,
                total,
                SUM(total) OVER (ORDER BY date) as running_total
            FROM (
                SELECT 
                    date,
                    SUM(total) as total
                FROM tblinvoices
                WHERE status = 'Paid'
                GROUP BY date
            ) daily
        ");
    }
    
    /**
     * Partition by status
     */
    public function getServicesByStatus(): array
    {
        return Capsule::select("
            SELECT 
                domainstatus,
                COUNT(*) as count,
                AVG(amount) as avg_price
            FROM tblhosting
            GROUP BY domainstatus WITH ROLLUP
        ");
    }
    
    /**
     * Recursive CTE for hierarchy
     */
    public function getDepartmentHierarchy(): array
    {
        return Capsule::select("
            WITH RECURSIVE dept_tree AS (
                SELECT id, name, parent_id, 0 as level
                FROM departments
                WHERE parent_id IS NULL
                
                UNION ALL
                
                SELECT d.id, d.name, d.parent_id, dt.level + 1
                FROM departments d
                JOIN dept_tree dt ON d.parent_id = dt.id
            )
            SELECT * FROM dept_tree ORDER BY level, name
        ");
    }
}
```

## Transactions

### Transaction Management

```php
<?php
/**
 * Transaction helper
 */
class TransactionHelper
{
    /**
     * Execute within transaction
     */
    public function withTransaction(callable $callback)
    {
        Capsule::connection()->transaction(function() use ($callback) {
            return $callback();
        });
    }
    
    /**
     * Create service with related records
     */
    public function createServiceWithBilling(int $clientId, array $serviceData, array $invoiceData): int
    {
        return Capsule::connection()->transaction(function() use ($clientId, $serviceData, $invoiceData) {
            // Create service
            $serviceId = Capsule::table('tblhosting')->insertGetId([
                'userid' => $clientId,
                'packageid' => $serviceData['package_id'],
                'domain' => $serviceData['domain'],
                'regdate' => date('Y-m-d'),
                'domainstatus' => 'Pending',
            ]);
            
            // Create invoice
            $invoiceId = Capsule::table('tblinvoices')->insertGetId([
                'userid' => $clientId,
                'date' => date('Y-m-d'),
                'duedate' => $invoiceData['due_date'] ?? date('Y-m-d', strtotime('+7 days')),
                'status' => 'Unpaid',
            ]);
            
            // Create invoice item
            Capsule::table('tblinvoiceitems')->insert([
                'invoiceid' => $invoiceId,
                'userid' => $clientId,
                'type' => 'Hosting',
                'relid' => $serviceId,
                'description' => $serviceData['description'],
                'amount' => $serviceData['amount'],
                'taxed' => 1,
            ]);
            
            // Update service with invoice
            Capsule::table('tblhosting')
                ->where('id', $serviceId)
                ->update(['invoice_id' => $invoiceId]);
            
            return $serviceId;
        });
    }
    
    /**
     * Bulk update with transaction
     */
    public function bulkUpdateServices(array $serviceIds, string $status): int
    {
        return Capsule::connection()->transaction(function() use ($serviceIds, $status) {
            return Capsule::table('tblhosting')
                ->whereIn('id', $serviceIds)
                ->update([
                    'domainstatus' => $status,
                    'updated_at' => date('Y-m-d H:i:s'),
                ]);
        });
    }
}
```

## Data Migration

### Safe Data Operations

```php
<?php
/**
 * Safe migration operations
 */
class SafeMigration
{
    /**
     * Copy data safely with validation
     */
    public function safeCopy(string $sourceTable, string $destTable, array $fieldMapping): array
    {
        $results = [
            'copied' => 0,
            'skipped' => 0,
            'errors' => [],
        ];
        
        $sourceRecords = Capsule::table($sourceTable)->get();
        
        foreach ($sourceRecords as $record) {
            try {
                $data = [];
                
                foreach ($fieldMapping as $sourceField => $destField) {
                    $data[$destField] = $record->$sourceField ?? null;
                }
                
                // Validate before insert
                if ($this->validateRecord($data)) {
                    Capsule::table($destTable)->insert($data);
                    $results['copied']++;
                } else {
                    $results['skipped']++;
                }
                
            } catch (Exception $e) {
                $results['errors'][] = [
                    'id' => $record->id ?? 'unknown',
                    'error' => $e->getMessage(),
                ];
            }
        }
        
        return $results;
    }
    
    /**
     * Validate record before insert
     */
    private function validateRecord(array $data): bool
    {
        // Check required fields
        if (empty($data['email'])) {
            return false;
        }
        
        // Check email format
        if (!filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
            return false;
        }
        
        // Check for duplicates
        $exists = Capsule::table('dest_table')
            ->where('email', $data['email'])
            ->exists();
        
        return !$exists;
    }
    
    /**
     * Batch update with progress
     */
    public function batchUpdate(string $table, array $updates, int $batchSize = 1000): array
    {
        $total = Capsule::table($table)->count();
        $processed = 0;
        
        Capsule::table($table)
            ->chunk($batchSize, function($records) use ($updates, &$processed) {
                foreach ($records as $record) {
                    Capsule::table($table)
                        ->where('id', $record->id)
                        ->update($updates);
                    $processed++;
                }
            });
        
        return [
            'total' => $total,
            'processed' => $processed,
        ];
    }
}
```

## Partitioning

### Table Partitioning

```php
<?php
/**
 * Table partitioning utilities
 */
class PartitionManager
{
    /**
     * Partition invoices by month
     */
    public function partitionInvoicesByMonth(): void
    {
        $pdo = Capsule::connection()->getPdo();
        
        // Check if partitioning is supported
        $engine = $pdo->query("SELECT ENGINE FROM information_schema.TABLES 
            WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = 'tblinvoices'")
            ->fetchColumn();
        
        if ($engine !== 'InnoDB') {
            throw new Exception('InnoDB required for partitioning');
        }
        
        // Create partitioned table
        Capsule::statement("
            ALTER TABLE tblinvoices
            PARTITION BY RANGE (YEAR(date) * 100 + MONTH(date)) (
                PARTITION p202401 VALUES LESS THAN (202402),
                PARTITION p202402 VALUES LESS THAN (202403),
                PARTITION p202403 VALUES LESS THAN (202404),
                PARTITION p202404 VALUES LESS THAN (202405),
                PARTITION p202405 VALUES LESS THAN (202406),
                PARTITION p_future VALUES LESS THAN MAXVALUE
            )
        ");
    }
    
    /**
     * Add partition for new month
     */
    public function addMonthlyPartition(string $table, int $year, int $month): void
    {
        $nextMonth = $month === 12 ? 1 : $month + 1;
        $nextYear = $month === 12 ? $year + 1 : $year;
        
        $partitionName = "p{$year}" . str_pad($month, 2, '0', STR_PAD_LEFT);
        $lessThanValue = $nextYear * 100 + $nextMonth;
        
        Capsule::statement("
            ALTER TABLE {$table}
            ADD PARTITION (
                PARTITION {$partitionName} VALUES LESS THAN ({$lessThanValue})
            )
        ");
    }
    
    /**
     * Get partition info
     */
    public function getPartitionInfo(string $table): array
    {
        $result = Capsule::select("
            SELECT PARTITION_NAME, TABLE_ROWS, DATA_LENGTH, INDEX_LENGTH
            FROM information_schema.PARTITIONS
            WHERE TABLE_SCHEMA = DATABASE()
            AND TABLE_NAME = ?
            AND PARTITION_NAME IS NOT NULL
            ORDER BY PARTITION_ORDINAL_POSITION
        ", [$table]);
        
        return $result;
    }
}
```

## Best Practices

1. **Use transactions** - Wrap related operations
2. **Index wisely** - Don't over-index
3. **Partition large tables** - Improve query performance
4. **Use EXPLAIN** - Analyze query plans
5. **Batch operations** - Process in chunks
6. **Validate data** - Check before insert

## Related Documentation

- [whmcs-advanced-performance.md](whmcs-advanced-performance.md)
- [whmcs-schema-tables.md](whmcs-schema-tables.md)
