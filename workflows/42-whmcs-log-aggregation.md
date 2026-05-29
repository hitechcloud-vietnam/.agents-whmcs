# WHMCS Log Aggregation Workflow

## Overview
This workflow covers centralized log aggregation for WHMCS.

## Step 1: Log Aggregation Service

```php
<?php
// src/Service/LogAggregationService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class LogAggregationService
{
    private $logSources = [
        'activity' => '/var/www/whmcs/storage/logs/activity.log',
        'module' => '/var/www/whmcs/storage/logs/module.log',
        'api' => '/var/www/whmcs/storage/logs/api.log',
        'error' => '/var/www/whmcs/storage/logs/error.log',
        'access' => '/var/www/whmcs/storage/logs/access.log'
    ];

    public function aggregateLogs(string $startTime = null, string $endTime = null): array
    {
        $logs = [];

        foreach ($this->logSources as $name => $path) {
            if (file_exists($path)) {
                $logs[$name] = $this->parseLogFile($path, $startTime, $endTime);
            }
        }

        $this->storeAggregatedLogs($logs);

        return [
            'aggregated_at' => date('Y-m-d H:i:s'),
            'sources' => array_keys($logs),
            'total_entries' => array_sum(array_map('count', $logs))
        ];
    }

    private function parseLogFile(string $path, ?string $startTime, ?string $endTime): array
    {
        $entries = [];
        $lines = file($path, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);

        foreach ($lines as $line) {
            $parsed = $this->parseLogLine($line);

            if ($parsed && $this->isInTimeRange($parsed['timestamp'], $startTime, $endTime)) {
                $entries[] = $parsed;
            }
        }

        return $entries;
    }

    private function parseLogLine(string $line): ?array
    {
        // Parse common log format: [timestamp] LEVEL: message
        if (preg_match('/\[([^\]]+)\]\s+(\w+):\s+(.+)/', $line, $matches)) {
            return [
                'timestamp' => $matches[1],
                'level' => $matches[2],
                'message' => $matches[3],
                'raw' => $line
            ];
        }

        return null;
    }

    private function isInTimeRange(?string $timestamp, ?string $startTime, ?string $endTime): bool
    {
        if (!$timestamp) return true;

        $time = strtotime($timestamp);

        if ($startTime && $time < strtotime($startTime)) {
            return false;
        }

        if ($endTime && $time > strtotime($endTime)) {
            return false;
        }

        return true;
    }

    private function storeAggregatedLogs(array $logs): void
    {
        foreach ($logs as $source => $entries) {
            foreach ($entries as $entry) {
                Capsule::table('mod_aggregated_logs')->insert([
                    'source' => $source,
                    'level' => $entry['level'] ?? 'INFO',
                    'message' => $entry['message'] ?? $entry['raw'],
                    'timestamp' => $entry['timestamp'] ?? date('Y-m-d H:i:s'),
                    'created_at' => date('Y-m-d H:i:s')
                ]);
            }
        }
    }

    public function queryLogs(array $filters = []): array
    {
        $query = Capsule::table('mod_aggregated_logs');

        if (!empty($filters['source'])) {
            $query->where('source', $filters['source']);
        }

        if (!empty($filters['level'])) {
            $query->where('level', $filters['level']);
        }

        if (!empty($filters['message_contains'])) {
            $query->where('message', 'like', '%' . $filters['message_contains'] . '%');
        }

        if (!empty($filters['start_time'])) {
            $query->where('timestamp', '>=', $filters['start_time']);
        }

        if (!empty($filters['end_time'])) {
            $query->where('timestamp', '<=', $filters['end_time']);
        }

        return $query
            ->orderBy('timestamp', 'desc')
            ->limit($filters['limit'] ?? 1000)
            ->get()
            ->toArray();
    }

    public function getErrorSummary(string $timeRange = '24 hours'): array
    {
        $since = date('Y-m-d H:i:s', strtotime("-{$timeRange}"));

        return Capsule::table('mod_aggregated_logs')
            ->where('level', 'ERROR')
            ->where('created_at', '>=', $since)
            ->selectRaw('source, COUNT(*) as count')
            ->groupBy('source')
            ->get()
            ->toArray();
    }
}
```

## Verification Checklist

- [ ] Log aggregation service implemented
- [ ] Log parsing working
- [ ] Storage configured
- [ ] Query functionality working
- [ ] Error summary generating
