# WHMCS Database Indexing Guide

**Version:** 8.x
**Updated:** 2026-05-28
**Related Skills:** `database-schema-design`, `performance-optimization`

## Overview

Proper database indexing is critical for WHMCS performance. This guide covers existing indexes, custom indexing strategies, and optimization techniques.

## Core WHMCS Tables Indexes

### tblclients (Client Accounts)

```sql
-- Existing indexes
SHOW INDEX FROM tblclients;

-- Primary Key: id
-- Indexes typically include:
-- - email (unique)
-- - status
-- - groupid
-- - datecreated

-- Custom index for domain-based lookups
ALTER TABLE tblclients
ADD INDEX idx_companyname (companyname);

-- Custom index for date-based queries
ALTER TABLE tblclients
ADD INDEX idx_created_status (datecreated, status);
```

### tblhosting (Hosting Services)

```sql
-- Existing indexes
SHOW INDEX FROM tblhosting;

-- Primary Key: id
-- Standard indexes include:
-- - userid
-- - domainstatus
-- - server
-- - packageid
-- - regdate

-- Critical: User + Status (frequently queried together)
ALTER TABLE tblhosting
ADD INDEX idx_userid_status (userid, domainstatus);

-- Critical: Next due date queries
ALTER TABLE tblhosting
ADD INDEX idx_nextduedate_status (nextduedate, domainstatus);

-- Custom: Server-based service queries
ALTER TABLE tblhosting
ADD INDEX idx_server_status (server, domainstatus);

-- Custom: Renewal processing
ALTER TABLE tblhosting
ADD INDEX idx_renewal_queue (domainstatus, nextduedate, billingcycle);
```

### tblorders (Orders)

```sql
-- Existing indexes
SHOW INDEX FROM tblorders;

-- Standard indexes:
-- - id
-- - userid
-- - status
-- - date

-- Critical: User + Status (common query)
ALTER TABLE tblorders
ADD INDEX idx_userid_status (userid, status);

-- Critical: Daily order reports
ALTER TABLE tblorders
ADD INDEX idx_date_status (date, status);

-- Custom: Affiliate reporting
ALTER TABLE tblorders
ADD INDEX idx_affiliate (affiliateid, status);
```

### tblinvoices (Invoices)

```sql
-- Existing indexes
SHOW INDEX FROM tblinvoices;

-- Standard indexes:
-- - id
-- - userid
-- - status
-- - duedate
-- - date

-- Critical: Overdue invoice queries
ALTER TABLE tblinvoices
ADD INDEX idx_overdue_status (userid, duedate, status);

-- Critical: Monthly revenue reports
ALTER TABLE tblinvoices
ADD INDEX idx_date_status_total (date, status, total);

-- Custom: Payment matching
ALTER TABLE tblinvoices
ADD INDEX idx_invoice_number (invoicenum);

-- Custom: Bulk status updates
ALTER TABLE tblinvoices
ADD INDEX idx_status_date (status, duedate);
```

### tblticketmessages (Support Tickets)

```sql
-- Existing indexes
SHOW INDEX FROM tblticketmessages;

-- Standard indexes:
-- - id
-- - tid (ticket ID)
-- - userid
-- - date

-- Critical: Ticket messages lookup
ALTER TABLE tblticketmessages
ADD INDEX idx_ticket_date (tid, date);

-- Critical: User ticket search
ALTER TABLE tblticketmessages
ADD INDEX idx_userid (userid);
```

### tbldomains (Domain Registrations)

```sql
-- Existing indexes
SHOW INDEX FROM tbldomains;

-- Standard indexes:
-- - id
-- - userid
-- - domain
-- - status
-- - nextduedate

-- Critical: Domain expiration queries
ALTER TABLE tbldomains
ADD INDEX idx_expiry_status (nextduedate, status);

-- Critical: Domain expiration reports
ALTER TABLE tbldomains
ADD INDEX idx_expiring_30_days (status, domaintype, nextduedate);

-- Custom: Registrar-based queries
ALTER TABLE tbldomains
ADD INDEX idx_registrar (registrar);
```

## Query Optimization Examples

### Common Expensive Queries

#### Find Overdue Services

```sql
-- BEFORE: Without proper index, scans entire table
SELECT h.*, c.email, c.firstname, c.lastname
FROM tblhosting h
JOIN tblclients c ON h.userid = c.id
WHERE h.domainstatus = 'Active'
AND h.nextduedate < CURDATE()
ORDER BY h.nextduedate ASC;

-- Optimization: Add composite index
ALTER TABLE tblhosting
ADD INDEX idx_overdue_services (domainstatus, nextduedate);
```

#### Monthly Revenue Report

```sql
-- BEFORE: Expensive query on large tables
SELECT
    YEAR(i.date) as year,
    MONTH(i.date) as month,
    SUM(i.total) as revenue
FROM tblinvoices i
WHERE i.status = 'Paid'
GROUP BY YEAR(i.date), MONTH(i.date);

-- Optimization: Add date index
ALTER TABLE tblinvoices
ADD INDEX idx_date_status_paid (date, status);
```

#### Client Service Summary

```sql
-- Expensive N+1 query pattern
SELECT c.id, c.email,
    (SELECT COUNT(*) FROM tblhosting WHERE userid = c.id AND domainstatus = 'Active') as active_services,
    (SELECT COUNT(*) FROM tblhosting WHERE userid = c.id AND domainstatus = 'Suspended') as suspended_services,
    (SELECT COUNT(*) FROM tblhosting WHERE userid = c.id AND domainstatus = 'Terminated') as terminated_services
FROM tblclients c;

-- Better: Use JOINs or subquery with proper indexes
SELECT
    c.id,
    c.email,
    COUNT(CASE WHEN h.domainstatus = 'Active' THEN 1 END) as active_services,
    COUNT(CASE WHEN h.domainstatus = 'Suspended' THEN 1 END) as suspended_services
FROM tblclients c
LEFT JOIN tblhosting h ON c.id = h.userid
GROUP BY c.id, c.email;
```

### Custom Indexes for Modules

#### Usage Tracking Table

```sql
-- Create usage tracking table
CREATE TABLE IF NOT EXISTS mod_yourmodule_usage (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    service_id INT UNSIGNED NOT NULL,
    record_date DATE NOT NULL,
    bandwidth_used BIGINT DEFAULT 0,
    disk_used BIGINT DEFAULT 0,
    requests_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_service_date (service_id, record_date),
    INDEX idx_record_date (record_date),
    FOREIGN KEY (service_id) REFERENCES tblhosting(id) ON DELETE CASCADE
);

-- Query for monthly usage
SELECT
    service_id,
    SUM(bandwidth_used) as total_bandwidth,
    AVG(disk_used) as avg_disk,
    SUM(requests_count) as total_requests
FROM mod_yourmodule_usage
WHERE record_date BETWEEN '2026-01-01' AND '2026-01-31'
GROUP BY service_id;
```

#### Custom Logging Table

```sql
-- Create module activity log
CREATE TABLE IF NOT EXISTS mod_yourmodule_activity (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    service_id INT UNSIGNED,
    action VARCHAR(50) NOT NULL,
    details JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_service_action (service_id, action),
    INDEX idx_created_at (created_at),
    INDEX idx_action_date (action, created_at)
);
```

## Index Maintenance

### Analyzing Index Usage

```sql
-- Find unused indexes (MySQL 8.0+)
SELECT
    object_schema,
    object_name,
    index_name,
    cardinality
FROM information_schema.statistics
WHERE object_schema = 'whmcs_database'
AND object_name LIKE 'tbl%'
AND index_name != 'PRIMARY'
ORDER BY cardinality DESC;

-- Check for duplicate indexes
SELECT
    s1.table_name,
    s1.index_name,
    s1.column_name,
    s2.index_name as duplicate_index,
    s2.column_name as duplicate_column
FROM information_schema.statistics s1
JOIN information_schema.statistics s2
    ON s1.table_name = s2.table_name
    AND s1.index_name != s2.index_name
    AND s1.column_name = s2.column_name
WHERE s1.object_schema = 'whmcs_database'
ORDER BY s1.table_name, s1.index_name;
```

### Index Rebuilding

```sql
-- Optimize table (rebuilds indexes)
OPTIMIZE TABLE tblclients;
OPTIMIZE TABLE tblhosting;
OPTIMIZE TABLE tblorders;
OPTIMIZE TABLE tblinvoices;
OPTIMIZE TABLE tbldomains;

-- For large tables, use pt-online-schema-change
-- (from Percona Toolkit)
pt-online-schema-change \
    --alter "OPTIMIZE TABLE" \
    --execute \
    D=mydb,t=tblhosting
```

### Monitoring Query Performance

```sql
-- Enable slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- View current queries
SHOW FULL PROCESSLIST;

-- Kill long-running query
KILL CONNECTION 12345;
```

## PHP-Based Index Management

```php
<?php
// includes/hooks/index_optimization.php

/**
 * Hook to log slow queries for indexing review
 */
add_hook('DatabaseQueryExecuted', 1, function($vars) {
    // Log queries taking longer than 100ms
    if ($vars['time'] > 0.1) {
        Capsule::table('tblslow_query_log')->insert([
            'query' => substr($vars['query'], 0, 1000),
            'execution_time' => $vars['time'],
            'created_at' => date('Y-m-d H:i:s'),
            'backtrace' => json_encode(debug_backtrace()),
        ]);
    }
});

/**
 * Helper to check if index exists
 */
function indexExists(string $table, string $indexName): bool
{
    $indexes = Capsule::connection()->select(
        "SHOW INDEX FROM `{$table}` WHERE Key_name = ?",
        [$indexName]
    );
    return count($indexes) > 0;
}

/**
 * Helper to create index safely (won't fail if exists)
 */
function safeCreateIndex(string $table, string $indexName, string $columns): bool
{
    if (indexExists($table, $indexName)) {
        return true; // Index already exists
    }

    try {
        Capsule::statement(
            "ALTER TABLE `{$table}` ADD INDEX `{$indexName}` ({$columns})"
        );
        return true;
    } catch (\Exception $e) {
        logActivity("Index creation failed: {$e->getMessage()}");
        return false;
    }
}

/**
 * Apply recommended indexes for WHMCS
 */
function applyWhmcsIndexes(): void
{
    $indexes = [
        'tblhosting' => [
            'idx_userid_status' => 'userid, domainstatus',
            'idx_nextduedate_status' => 'nextduedate, domainstatus',
        ],
        'tblorders' => [
            'idx_userid_status' => 'userid, status',
            'idx_date_status' => 'date, status',
        ],
        'tblinvoices' => [
            'idx_overdue_status' => 'userid, duedate, status',
            'idx_date_status_total' => 'date, status, total',
        ],
    ];

    foreach ($indexes as $table => $tableIndexes) {
        foreach ($tableIndexes as $indexName => $columns) {
            safeCreateIndex($table, $indexName, $columns);
        }
    }
}

// Run on admin login (can also run via cron)
// add_hook('AdminLogin', 1, function() { applyWhmcsIndexes(); });
```

## Performance Impact Matrix

| Table | Query Type | Recommended Index | Impact |
|-------|------------|-------------------|--------|
| tblclients | Email lookup | email (exists) | High |
| tblclients | Company search | companyname | Medium |
| tblhosting | User services | userid, status | High |
| tblhosting | Due date processing | nextduedate, status | High |
| tblhosting | Server services | server, status | High |
| tblorders | User history | userid, status | High |
| tblorders | Daily reports | date, status | High |
| tblinvoices | User invoices | userid, status | High |
| tblinvoices | Overdue queries | duedate, status | High |
| tbldomains | Expiring domains | nextduedate, status | High |
| tbldomains | User domains | userid, status | Medium |

## Best Practices

1. **Analyze Before Creating**: Use EXPLAIN to verify index helps
2. **Order Matters**: Columns in composite index should be ordered by selectivity
3. **Avoid Over-Indexing**: Each index slows INSERT/UPDATE
4. **Monitor Performance**: Use slow query log to identify issues
5. **Regular Maintenance**: OPTIMIZE tables periodically
6. **Test Impact**: Verify index improves query, not slows writes
7. **Document Changes**: Keep track of custom indexes for upgrades

## Related Documentation

- [Database Schema Design](database-schema-design.md)
- [Performance Optimization](performance-optimization.md)
- [Caching Strategies](caching-strategies.md)
