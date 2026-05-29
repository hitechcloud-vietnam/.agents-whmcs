# WHMCS Database Maintenance Workflow

## Overview
This workflow covers regular database maintenance tasks for WHMCS.

## Step 1: Database Maintenance Service

```php
<?php
// src/Service/DatabaseMaintenanceService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class DatabaseMaintenanceService
{
    public function runMaintenance(): array
    {
        $results = [
            'started_at' => date('Y-m-d H:i:s'),
            'operations' => []
        ];

        $results['operations']['optimize_tables'] = $this->optimizeTables();
        $results['operations']['analyze_tables'] = $this->analyzeTables();
        $results['operations']['clean_logs'] = $this->cleanOldLogs();
        $results['operations']['repair_tables'] = $this->checkAndRepairTables();
        $results['operations']['index_maintenance'] = $this->maintainIndexes();

        $results['completed_at'] = date('Y-m-d H:i:s');

        return $results;
    }

    public function optimizeTables(): array
    {
        $optimized = 0;
        $errors = [];

        $tables = Capsule::connection()->select('SHOW TABLES');

        foreach ($tables as $table) {
            $tableName = array_values((array)$table)[0];

            try {
                Capsule::connection()->statement("OPTIMIZE TABLE `{$tableName}`");
                $optimized++;
            } catch (\Exception $e) {
                $errors[] = [
                    'table' => $tableName,
                    'error' => $e->getMessage()
                ];
            }
        }

        return [
            'optimized' => $optimized,
            'errors' => $errors
        ];
    }

    public function analyzeTables(): array
    {
        $analyzed = 0;

        $tables = Capsule::connection()->select('SHOW TABLES');

        foreach ($tables as $table) {
            $tableName = array_values((array)$table)[0];

            try {
                Capsule::connection()->statement("ANALYZE TABLE `{$tableName}`");
                $analyzed++;
            } catch (\Exception $e) {
                // Ignore errors for analyze
            }
        }

        return ['analyzed' => $analyzed];
    }

    public function cleanOldLogs(int $daysToKeep = 90): array
    {
        $cutoffDate = date('Y-m-d H:i:s', strtotime("-{$daysToKeep} days"));

        $deleted = [];

        // Clean activity logs
        $activityDeleted = Capsule::table('tblactivitylog')
            ->where('date', '<', $cutoffDate)
            ->delete();
        $deleted['activity_log'] = $activityDeleted;

        // Clean admin logs
        $adminDeleted = Capsule::table('tbladminlog')
            ->where('date', '<', $cutoffDate)
            ->delete();
        $deleted['admin_log'] = $adminDeleted;

        // Clean module logs
        $moduleDeleted = Capsule::table('mod_logs')
            ->where('created_at', '<', $cutoffDate)
            ->delete();
        $deleted['module_log'] = $moduleDeleted;

        return $deleted;
    }

    public function checkAndRepairTables(): array
    {
        $tables = Capsule::connection()->select('SHOW TABLES');
        $issues = [];

        foreach ($tables as $table) {
            $tableName = array_values((array)$table)[0];

            try {
                $result = Capsule::connection()->select("CHECK TABLE `{$tableName}`");

                foreach ($result as $row) {
                    if ($row->Msg_type !== 'status' || $row->Msg_text !== 'OK') {
                        // Table has issues, try to repair
                        Capsule::connection()->statement("REPAIR TABLE `{$tableName}`");
                        $issues[] = [
                            'table' => $tableName,
                            'status' => 'repaired',
                            'original_status' => $row->Msg_text
                        ];
                    }
                }
            } catch (\Exception $e) {
                $issues[] = [
                    'table' => $tableName,
                    'status' => 'error',
                    'error' => $e->getMessage()
                ];
            }
        }

        return ['issues_found' => count($issues), 'details' => $issues];
    }

    public function maintainIndexes(): array
    {
        $recommendations = [];

        // Check for missing indexes on foreign keys
        $foreignKeys = [
            'tblhosting' => ['userid', 'packageid', 'serverid'],
            'tblinvoices' => ['userid'],
            'tblorders' => ['userid'],
            'tblticket' => ['userid', 'adminid']
        ];

        foreach ($foreignKeys as $table => $columns) {
            foreach ($columns as $column) {
                if (!$this->indexExists($table, $column)) {
                    $recommendations[] = [
                        'type' => 'missing_index',
                        'table' => $table,
                        'column' => $column,
                        'recommendation' => "ALTER TABLE `{$table}` ADD INDEX `idx_{$column}` (`{$column}`)"
                    ];
                }
            }
        }

        return ['recommendations' => $recommendations];
    }

    private function indexExists(string $table, string $column): bool
    {
        $indexes = Capsule::connection()->select("SHOW INDEX FROM `{$table}` WHERE Column_name = '{$column}'");
        return count($indexes) > 0;
    }

    public function getDatabaseStats(): array
    {
        $tables = Capsule::connection()->select(
            "SELECT
                table_name,
                ROUND(data_length / 1024 / 1024, 2) AS data_mb,
                ROUND(index_length / 1024 / 1024, 2) AS index_mb,
                ROUND((data_length + index_length) / 1024 / 1024, 2) AS total_mb,
                table_rows
            FROM information_schema.tables
            WHERE table_schema = ?
            ORDER BY (data_length + index_length) DESC",
            [Capsule::config('db_name')]
        );

        $totalSize = array_sum(array_column($tables, 'total_mb'));

        return [
            'tables' => $tables,
            'total_size_mb' => round($totalSize, 2),
            'table_count' => count($tables)
        ];
    }

    public function getSlowQueries(int $limit = 10): array
    {
        return Capsule::connection()->select(
            "SELECT
                query,
                exec_count,
                total_latency,
                avg_latency,
                rows_sent,
                rows_examined
            FROM performance_schema.events_statements_summary_by_digest
            WHERE avg_latency > 1000000
            ORDER BY avg_latency DESC
            LIMIT ?",
            [$limit]
        );
    }
}
```

## Step 2: Maintenance Cron

```php
<?php
// includes/cron/database_maintenance_cron.php

require_once __DIR__ . '/../../init.php';

use WHMCS\Module\Addon\YourModule\Service\DatabaseMaintenanceService;

$maintenance = new DatabaseMaintenanceService();

// Run weekly maintenance on Sunday at 3 AM
if (date('w') === 0 && date('H') === '03') {
    $results = $maintenance->runMaintenance();

    logActivity("Database maintenance completed: " . json_encode($results));

    // Alert if issues found
    if (!empty($results['operations']['repair_tables']['details'])) {
        // Send alert to admin
    }
}
```

## Verification Checklist

- [ ] Database maintenance service implemented
- [ ] Table optimization working
- [ ] Log cleanup working
- [ ] Table repair working
- [ ] Index recommendations working
- [ ] Cron job configured
