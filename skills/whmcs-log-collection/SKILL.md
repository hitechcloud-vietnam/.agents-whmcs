---
name: whmcs-log-collection
description: Centralized logging for WHMCS services
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Centralized Logging Skill

## Overview
This skill provides patterns and implementations for setting up centralized logging in WHMCS, including log aggregation, parsing, retention policies, and search functionality.

## Implementation Patterns

### Log Collection Manager
```php
<?php
/**
 * WHMCS Centralized Log Collection
 * Manages log aggregation from hosted services
 */

namespace WHMCS\Module\Server\Logging;

class LogCollectionManager {
    private $db;
    private $parsers = [];
    private $storageBackends = [];

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->initializeParsers();
        $this->initializeStorageBackends();
    }

    private function initializeParsers(): void {
        $this->parsers = [
            'syslog' => new SyslogParser(),
            'json' => new JSONLogParser(),
            'apache' => new ApacheLogParser(),
            'nginx' => new NginxLogParser(),
            'application' => new ApplicationLogParser()
        ];
    }

    private function initializeStorageBackends(): void {
        $this->storageBackends = [
            'elasticsearch' => new ElasticsearchStorage(),
            'splunk' => new SplunkStorage(),
            's3' => new S3LogStorage(),
            'local' => new LocalLogStorage()
        ];
    }

    /**
     * Configure log collection for service
     */
    public function configureCollection(array $params): array {
        $serviceId = $params['service_id'];
        $collectionId = 'lc_' . bin2hex(random_bytes(12));

        $collection = [
            'id' => $collectionId,
            'service_id' => $serviceId,
            'log_sources' => json_encode($params['sources'] ?? []),
            'parser_type' => $params['parser'] ?? 'json',
            'retention_days' => $params['retention_days'] ?? 30,
            'forward_to' => json_encode($params['forward_to'] ?? []),
            'enabled' => true,
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_log_collections', $collection);

        // Configure log shipping on VM
        $this->configureLogShipper($serviceId, $collection);

        return [
            'success' => true,
            'collection_id' => $collectionId
        ];
    }

    /**
     * Search logs
     */
    public function search(array $params): array {
        $query = $params['query'] ?? '';
        $serviceId = $params['service_id'] ?? null;
        $from = $params['from'] ?? date('Y-m-d H:i:s', strtotime('-24 hours'));
        $to = $params['to'] ?? date('Y-m-d H:i:s');
        $limit = $params['limit'] ?? 100;

        $sql = "SELECT * FROM mod_logs WHERE 1=1";
        $bindings = [];

        if ($serviceId) {
            $sql .= " AND service_id = ?";
            $bindings[] = $serviceId;
        }

        if ($query) {
            $sql .= " AND (message LIKE ? OR source LIKE ?)";
            $bindings[] = '%' . $query . '%';
            $bindings[] = '%' . $query . '%';
        }

        $sql .= " AND created_at BETWEEN ? AND ?";
        $bindings[] = $from;
        $bindings[] = $to;

        $sql .= " ORDER BY created_at DESC LIMIT ?";
        $bindings[] = $limit;

        $logs = $this->db->select($sql, $bindings);

        return array_map(function($log) {
            return [
                'id' => $log->id,
                'timestamp' => $log->created_at,
                'service_id' => $log->service_id,
                'level' => $log->level,
                'source' => $log->source,
                'message' => $log->message,
                'metadata' => json_decode($log->metadata, true) ?? []
            ];
        }, $logs);
    }

    /**
     * Get log statistics
     */
    public function getStats(int $serviceId, string $period = '24h'): array {
        $from = match($period) {
            '1h' => date('Y-m-d H:i:s', strtotime('-1 hour')),
            '24h' => date('Y-m-d H:i:s', strtotime('-24 hours')),
            '7d' => date('Y-m-d H:i:s', strtotime('-7 days')),
            '30d' => date('Y-m-d H:i:s', strtotime('-30 days')),
            default => date('Y-m-d H:i:s', strtotime('-24 hours'))
        };

        $stats = $this->db->select(
            "SELECT level, COUNT(*) as count
             FROM mod_logs
             WHERE service_id = ? AND created_at >= ?
             GROUP BY level",
            [$serviceId, $from]
        );

        $total = 0;
        $byLevel = [];

        foreach ($stats as $stat) {
            $byLevel[$stat->level] = $stat->count;
            $total += $stat->count;
        }

        return [
            'service_id' => $serviceId,
            'period' => $period,
            'total_logs' => $total,
            'by_level' => $byLevel,
            'from' => $from,
            'to' => date('Y-m-d H:i:s')
        ];
    }

    /**
     * Create log archive
     */
    public function createArchive(int $serviceId, array $params): array {
        $from = $params['from'];
        $to = $params['to'];
        $archiveId = 'arch_' . bin2hex(random_bytes(12));

        // Query logs within range
        $logs = $this->db->select(
            "SELECT * FROM mod_logs
             WHERE service_id = ? AND created_at BETWEEN ? AND ?
             ORDER BY created_at ASC",
            [$serviceId, $from, $to]
        );

        // Create archive file
        $archivePath = $this->createArchiveFile($archiveId, $logs);

        // Store archive metadata
        $this->db->insert('mod_log_archives', [
            'id' => $archiveId,
            'service_id' => $serviceId,
            'archive_path' => $archivePath,
            'size_bytes' => filesize($archivePath),
            'log_count' => count($logs),
            'from_date' => $from,
            'to_date' => $to,
            'created_at' => date('Y-m-d H:i:s')
        ]);

        // Optionally delete original logs
        if ($params['delete_original'] ?? false) {
            $this->db->delete('mod_logs', [
                'service_id' => $serviceId,
                'created_at >= ?' => $from,
                'created_at <= ?' => $to
            ]);
        }

        return [
            'success' => true,
            'archive_id' => $archiveId,
            'path' => $archivePath,
            'size_bytes' => filesize($archivePath),
            'log_count' => count($logs)
        ];
    }

    /**
     * Configure log rotation
     */
    public function configureRotation(int $serviceId, array $config): array {
        $rotation = [
            'service_id' => $serviceId,
            'max_size_mb' => $config['max_size_mb'] ?? 100,
            'max_files' => $config['max_files'] ?? 10,
            'compression' => $config['compression'] ?? true,
            'enabled' => true
        ];

        $this->db->update('mod_log_collections', [
            'rotation_config' => json_encode($rotation)
        ], ['service_id' => $serviceId]);

        return [
            'success' => true,
            'rotation' => $rotation
        ];
    }

    /**
     * Forward logs to external system
     */
    public function forwardLogs(array $params): bool {
        $logs = $params['logs'];
        $destination = $params['destination'];
        $format = $params['format'] ?? 'json';

        if (!isset($this->storageBackends[$destination['type']])) {
            throw new \Exception("Unknown destination type: {$destination['type']}");
        }

        $backend = $this->storageBackends[$destination['type']];
        $backend->send($logs, $destination);

        return true;
    }

    // Private helper methods

    private function configureLogShipper(int $serviceId, array $collection): void {
        $vmId = $this->getVmId($serviceId);
        $vmIp = $this->getVmIP($serviceId);

        $shipperConfig = $this->generateShipperConfig($collection);
        $this->deployLogShipper($vmId, $vmIp, $shipperConfig);
    }

    private function generateShipperConfig(array $collection): string {
        $sources = json_decode($collection['log_sources'], true);
        $forwardTo = json_decode($collection['forward_to'], true);

        return json_encode([
            'inputs' => array_map(function($source) {
                return [
                    'type' => $source['type'],
                    'paths' => $source['paths'],
                    'parser' => $collection['parser_type']
                ];
            }, $sources),
            'outputs' => array_map(function($dest) {
                return [
                    'type' => $dest['type'],
                    'host' => $dest['host'],
                    'port' => $dest['port']
                ];
            }, $forwardTo)
        ]);
    }
}

/**
 * Log Parsers
 */
interface LogParserInterface {
    public function parse(string $line): ?array;
    public function getFields(): array;
}

class JSONLogParser implements LogParserInterface {
    public function parse(string $line): ?array {
        $data = json_decode($line, true);
        if (!$data) return null;

        return [
            'timestamp' => $data['timestamp'] ?? $data['time'] ?? null,
            'level' => $data['level'] ?? $data['severity'] ?? 'info',
            'message' => $data['message'] ?? $data['msg'] ?? '',
            'source' => $data['source'] ?? $data['logger'] ?? 'unknown',
            'metadata' => $data
        ];
    }

    public function getFields(): array {
        return ['timestamp', 'level', 'message', 'source', 'metadata'];
    }
}

class ApacheLogParser implements LogParserInterface {
    private $pattern = '/^(\S+) \S+ \S+ \[([^\]]+)\] "([^"]*)" (\d+) (\d+) "([^"]*)" "([^"]*)"/';

    public function parse(string $line): ?array {
        if (!preg_match($this->pattern, $line, $matches)) {
            return null;
        }

        return [
            'timestamp' => $matches[2],
            'ip' => $matches[1],
            'method' => explode(' ', $matches[3])[0] ?? 'GET',
            'path' => explode(' ', $matches[3])[1] ?? '/',
            'status' => (int) $matches[4],
            'bytes' => (int) $matches[5],
            'referrer' => $matches[6],
            'user_agent' => $matches[7]
        ];
    }

    public function getFields(): array {
        return ['timestamp', 'ip', 'method', 'path', 'status', 'bytes'];
    }
}

/**
 * Storage Backends
 */
interface LogStorageInterface {
    public function send(array $logs, array $config): void;
    public function query(array $params): array;
}

class ElasticsearchStorage implements LogStorageInterface {
    private $hosts = [];

    public function configure(array $config): void {
        $this->hosts = $config['hosts'];
    }

    public function send(array $logs, array $config): void {
        $index = $config['index'] ?? 'whmcs-logs-' . date('Y.m.d');

        $params = ['body' => []];
        foreach ($logs as $log) {
            $params['body'][] = [
                'index' => [
                    '_index' => $index
                ]
            ];
            $params['body'][] = $log;
        }

        // Use Elasticsearch client
        $client = new \Elasticsearch\Client([
            'hosts' => $this->hosts
        ]);

        $client->bulk($params);
    }

    public function query(array $params): array {
        $client = new \Elasticsearch\Client(['hosts' => $this->hosts]);

        $query = [
            'index' => $params['index'] ?? 'whmcs-logs-*',
            'body' => [
                'query' => [
                    'match' => ['message' => $params['query']]
                ]
            ]
        ];

        return $client->search($query)['hits']['hits'];
    }
}

class S3LogStorage implements LogStorageInterface {
    private $bucket;
    private $prefix;

    public function send(array $logs, array $config): void {
        $this->bucket = $config['bucket'];
        $this->prefix = $config['prefix'] ?? 'logs/';

        $filename = date('Y/m/d/H/') . uniqid() . '.jsonl';
        $content = implode("\n", array_map('json_encode', $logs));

        $s3 = new \Aws\S3\S3Client(['region' => $config['region']]);
        $s3->putObject([
            'Bucket' => $this->bucket,
            'Key' => $this->prefix . $filename,
            'Body' => $content,
            'ContentType' => 'application/x-ndjson'
        ]);
    }

    public function query(array $params): array {
        // Query using Athena or similar
        return [];
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_log_collections` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `log_sources` TEXT NOT NULL,
  `parser_type` VARCHAR(50) DEFAULT 'json',
  `retention_days` INT DEFAULT 30,
  `forward_to` TEXT,
  `rotation_config` TEXT,
  `enabled` TINYINT(1) DEFAULT 1,
  `created_at` DATETIME NOT NULL,
  UNIQUE KEY `unique_service` (`service_id`)
);

CREATE TABLE `mod_logs` (
  `id` BIGINT AUTO_INCREMENT PRIMARY KEY,
  `service_id` INT NOT NULL,
  `timestamp` DATETIME NOT NULL,
  `level` ENUM('debug', 'info', 'warning', 'error', 'critical') DEFAULT 'info',
  `source` VARCHAR(100),
  `message` TEXT NOT NULL,
  `metadata` TEXT,
  `created_at` DATETIME NOT NULL,
  INDEX `idx_service_time` (`service_id`, `created_at`),
  INDEX `idx_level` (`level`),
  INDEX `idx_source` (`source`)
);

CREATE TABLE `mod_log_archives` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `archive_path` VARCHAR(500) NOT NULL,
  `size_bytes` BIGINT DEFAULT 0,
  `log_count` INT DEFAULT 0,
  `from_date` DATETIME NOT NULL,
  `to_date` DATETIME NOT NULL,
  `created_at` DATETIME NOT NULL
);
```

## Log Shipper Configuration (Filebeat example)
```yaml
# filebeat.yml for WHMCS log collection
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/syslog
      - /var/log/nginx/*.log
      - /opt/app/logs/*.log
    json.keys_under_root: true
    json.add_error_key: true

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "whmcs-logs-%{+yyyy.MM.dd}"

processors:
  - add_host_metadata:
  - add_cloud_metadata:
```

## Best Practices

1. **Structured Logging**: Use JSON format for easy parsing
2. **Log Levels**: Properly categorize log severity
3. **Retention Policies**: Balance storage costs with compliance
4. **Indexing**: Create indexes for frequently searched fields
5. **Compression**: Compress archived logs to save space

## Related Skills

- whmcs-monitoring-agent
- whmcs-log-analysis
- whmcs-incident-response
- whmcs-debug-mode