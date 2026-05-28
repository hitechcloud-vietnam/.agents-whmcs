# Failover Strategies for WHMCS

High availability failover strategies ensure continuous operation when database servers or other critical components experience failures.

## High Availability Architecture

### HA Configuration

```php
<?php
/**
 * High availability configuration
 */
class HighAvailabilityConfig
{
    public const FAILOVER_MODES = [
        'active_passive' => 'One primary, one or more standby servers',
        'active_active' => 'Multiple servers handling traffic simultaneously',
        'cluster' => 'Coordinated cluster with shared storage',
    ];

    /**
     * Get HA configuration
     */
    public static function getConfig(): array
    {
        return [
            'mode' => 'active_passive',
            'health_check' => [
                'enabled' => true,
                'interval' => 5, // seconds
                'timeout' => 3,
                'retries' => 3,
                'endpoint' => '/health.php',
            ],
            'servers' => [
                'primary' => [
                    'id' => 'server-1',
                    'host' => '10.0.0.1',
                    'port' => 3306,
                    'priority' => 100,
                    'read_only' => false,
                ],
                'standby' => [
                    [
                        'id' => 'server-2',
                        'host' => '10.0.0.2',
                        'port' => 3306,
                        'priority' => 80,
                        'read_only' => true,
                    ],
                    [
                        'id' => 'server-3',
                        'host' => '10.0.0.3',
                        'port' => 3306,
                        'priority' => 60,
                        'read_only' => true,
                    ],
                ],
            ],
            'virtual_ip' => [
                'enabled' => true,
                'address' => '10.0.0.100',
                'interface' => 'eth0',
            ],
            'failover' => [
                'grace_period' => 10,
                'pre_failover_script' => '/usr/local/bin/pre-failover.sh',
                'post_failover_script' => '/usr/local/bin/post-failover.sh',
            ],
        ];
    }
}
```

## Failover Manager

### Database Failover Controller

```php
<?php
/**
 * Database failover manager
 */
class DatabaseFailoverManager
{
    private array $config;
    private ?PDO $activeConnection = null;
    private ?string $activeServerId = null;
    private bool $isFailingOver = false;

    public function __construct()
    {
        $this->config = HighAvailabilityConfig::getConfig();
        $this->initializeConnection();
    }

    /**
     * Get current active connection
     */
    public function getConnection(): PDO
    {
        if ($this->activeConnection === null) {
            $this->initializeConnection();
        }

        return $this->activeConnection;
    }

    /**
     * Check if connection is healthy
     */
    public function isHealthy(): bool
    {
        try {
            $stmt = $this->activeConnection->query('SELECT 1');
            return $stmt->fetch() !== false;
        } catch (Exception $e) {
            return false;
        }
    }

    /**
     * Perform failover to next available server
     */
    public function failover(): bool
    {
        if ($this->isFailingOver) {
            return false;
        }

        $this->isFailingOver = true;

        try {
            logActivity('WHMCS Database: Starting failover');

            // Execute pre-failover script
            $this->executeScript($this->config['failover']['pre_failover_script']);

            // Find best standby server
            $standby = $this->selectStandbyServer();

            if ($standby === null) {
                logActivity('WHMCS Database: No available standby server');
                return false;
            }

            // Promote standby
            $this->promoteServer($standby);

            // Update connection
            $this->activeServerId = $standby['id'];
            $this->activeConnection = $this->createConnection($standby);

            // Execute post-failover script
            $this->executeScript($this->config['failover']['post_failover_script']);

            logActivity('WHMCS Database: Failover completed to ' . $standby['id']);

            // Trigger application notification
            $this->notifyApplication();

            return true;
        } catch (Exception $e) {
            logActivity('WHMCS Database: Failover failed - ' . $e->getMessage());
            return false;
        } finally {
            $this->isFailingOver = false;
        }
    }

    /**
     * Check all servers and failover if primary fails
     */
    public function healthCheck(): array
    {
        $results = [];
        $allHealthy = true;

        // Check primary
        $primary = $this->config['servers']['primary'];
        $results[$primary['id']] = $this->checkServer($primary);

        if (!$results[$primary['id']]['healthy']) {
            $allHealthy = false;
        }

        // Check standbys
        foreach ($this->config['servers']['standby'] as $standby) {
            $results[$standby['id']] = $this->checkServer($standby);
        }

        // Trigger failover if primary unhealthy
        if (!$results[$primary['id']]['healthy'] && !$this->isFailingOver) {
            $this->failover();
        }

        return [
            'all_healthy' => $allHealthy,
            'active_server' => $this->activeServerId,
            'servers' => $results,
            'checked_at' => date('Y-m-d H:i:s'),
        ];
    }

    /**
     * Get virtual IP for application
     */
    public function getVirtualIp(): ?string
    {
        if (!$this->config['virtual_ip']['enabled']) {
            return null;
        }

        return $this->config['virtual_ip']['address'];
    }

    private function initializeConnection(): void
    {
        $primary = $this->config['servers']['primary'];
        $this->activeServerId = $primary['id'];

        try {
            $this->activeConnection = $this->createConnection($primary);
        } catch (Exception $e) {
            // Fall back to standby
            $standby = $this->selectStandbyServer();
            if ($standby) {
                $this->activeServerId = $standby['id'];
                $this->activeConnection = $this->createConnection($standby);
            } else {
                throw new RuntimeException('No database servers available');
            }
        }
    }

    private function createConnection(array $config): PDO
    {
        $dsn = sprintf(
            'mysql:host=%s;port=%d;dbname=%s;charset=utf8mb4',
            $config['host'],
            $config['port'],
            'whmcs'
        );

        return new PDO($dsn, 'whmcs_user', 'password', [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_TIMEOUT => 5,
        ]);
    }

    private function checkServer(array $config): array
    {
        $startTime = microtime(true);

        try {
            $connection = $this->createConnection($config);
            $stmt = $connection->query('SELECT 1');
            $latency = (microtime(true) - $startTime) * 1000;

            return [
                'healthy' => true,
                'latency_ms' => round($latency, 2),
                'error' => null,
            ];
        } catch (Exception $e) {
            return [
                'healthy' => false,
                'latency_ms' => null,
                'error' => $e->getMessage(),
            ];
        }
    }

    private function selectStandbyServer(): ?array
    {
        $candidates = [];

        foreach ($this->config['servers']['standby'] as $standby) {
            $health = $this->checkServer($standby);

            if ($health['healthy']) {
                $candidates[] = [
                    'server' => $standby,
                    'health' => $health,
                ];
            }
        }

        if (empty($candidates)) {
            return null;
        }

        // Sort by priority
        usort($candidates, fn($a, $b) =>
            $b['server']['priority'] <=> $a['server']['priority']
        );

        return $candidates[0]['server'];
    }

    private function promoteServer(array $server): void
    {
        $connection = $this->createConnection($server);

        // Stop replication if slave
        $connection->exec('STOP SLAVE');
        $connection->exec('RESET SLAVE ALL');

        // Enable writes
        $connection->exec('SET GLOBAL read_only = 0');
        $connection->exec('SET GLOBAL super_read_only = 0');
    }

    private function executeScript(string $script): void
    {
        if (file_exists($script)) {
            exec($script . ' 2>&1', $output, $returnCode);

            if ($returnCode !== 0) {
                logActivity("WHMCS: Failover script {$script} returned {$returnCode}");
            }
        }
    }

    private function notifyApplication(): void
    {
        // Write failover notification for application to pick up
        $notification = [
            'type' => 'database_failover',
            'new_primary' => $this->activeServerId,
            'timestamp' => date('Y-m-d H:i:s'),
        ];

        file_put_contents(
            '/tmp/whmcs_failover_notification.json',
            json_encode($notification)
        );
    }
}
```

## Connection Pooling

### PDO Connection Pool

```php
<?php
/**
 * Connection pool for database connections
 */
class ConnectionPool
{
    private int $minConnections = 5;
    private int $maxConnections = 50;
    private int $currentConnections = 0;
    private array $availableConnections = [];
    private array $inUseConnections = [];
    private float $connectionTimeout = 5.0;
    private float $idleTimeout = 300.0; // 5 minutes
    private array $config;
    private static ?self $instance = null;

    private function __construct()
    {
        $this->config = HighAvailabilityConfig::getConfig();
        $this->initializePool();
    }

    public static function getInstance(): self
    {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    /**
     * Get connection from pool
     */
    public function getConnection(): PDO
    {
        // Try to get from available pool
        if (!empty($this->availableConnections)) {
            $connection = array_pop($this->availableConnections);
            $this->inUseConnections[spl_object_id($connection)] = $connection;
            return $connection;
        }

        // Create new connection if under limit
        if ($this->currentConnections < $this->maxConnections) {
            $connection = $this->createConnection();
            $this->currentConnections++;
            $this->inUseConnections[spl_object_id($connection)] = $connection;
            return $connection;
        }

        // Wait for available connection
        return $this->waitForConnection();
    }

    /**
     * Return connection to pool
     */
    public function releaseConnection(PDO $connection): void
    {
        $id = spl_object_id($connection);

        if (isset($this->inUseConnections[$id])) {
            unset($this->inUseConnections[$id]);

            // Check if connection is still valid
            if ($this->isConnectionValid($connection)) {
                $this->availableConnections[] = $connection;
            } else {
                $this->currentConnections--;
            }
        }
    }

    /**
     * Execute with automatic connection management
     */
    public function execute(callable $callback): mixed
    {
        $connection = $this->getConnection();

        try {
            return $callback($connection);
        } finally {
            $this->releaseConnection($connection);
        }
    }

    /**
     * Health check and cleanup
     */
    public function maintenance(): array
    {
        $stats = [
            'total' => $this->currentConnections,
            'available' => count($this->availableConnections),
            'in_use' => count($this->inUseConnections),
            'closed' => 0,
        ];

        // Clean up idle connections
        $validConnections = [];
        foreach ($this->availableConnections as $connection) {
            if ($this->isConnectionValid($connection)) {
                $validConnections[] = $connection;
            } else {
                $this->currentConnections--;
                $stats['closed']++;
            }
        }

        $this->availableConnections = $validConnections;

        // Ensure minimum connections
        while ($this->currentConnections < $this->minConnections) {
            $connection = $this->createConnection();
            $this->availableConnections[] = $connection;
            $this->currentConnections++;
        }

        return $stats;
    }

    private function initializePool(): void
    {
        for ($i = 0; $i < $this->minConnections; $i++) {
            $connection = $this->createConnection();
            $this->availableConnections[] = $connection;
            $this->currentConnections++;
        }
    }

    private function createConnection(): PDO
    {
        $primary = $this->config['servers']['primary'];

        $dsn = sprintf(
            'mysql:host=%s;port=%d;dbname=%s;charset=utf8mb4',
            $primary['host'],
            $primary['port'],
            'whmcs'
        );

        return new PDO($dsn, 'whmcs_user', 'password', [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_OBJ,
            PDO::ATTR_TIMEOUT => 5,
        ]);
    }

    private function isConnectionValid(PDO $connection): bool
    {
        try {
            $stmt = $connection->query('SELECT 1');
            return $stmt->fetch() !== false;
        } catch (Exception $e) {
            return false;
        }
    }

    private function waitForConnection(): PDO
    {
        $startTime = microtime(true);

        while (microtime(true) - $startTime < $this->connectionTimeout) {
            if (!empty($this->availableConnections)) {
                return $this->getConnection();
            }
            usleep(100000); // 100ms
        }

        throw new RuntimeException('Connection pool timeout');
    }
}
```

## Health Check Endpoint

### Health Check Implementation

```php
<?php
<?php
/**
 * WHMCS health check endpoint
 * Location: public_html/health.php
 */

header('Content-Type: application/json');
header('Cache-Control: no-store');

$health = [
    'status' => 'healthy',
    'timestamp' => date('Y-m-d H:i:s'),
    'checks' => [],
];

// Database check
try {
    $startTime = microtime(true);

    $pdo = new PDO(
        'mysql:host=127.0.0.1;dbname=whmcs;charset=utf8mb4',
        'whmcs_user',
        'password',
        [PDO::ATTR_TIMEOUT => 3]
    );

    $pdo->query('SELECT 1');
    $latency = (microtime(true) - $startTime) * 1000;

    $health['checks']['database'] = [
        'status' => 'healthy',
        'latency_ms' => round($latency, 2),
    ];
} catch (PDOException $e) {
    $health['checks']['database'] = [
        'status' => 'unhealthy',
        'error' => $e->getMessage(),
    ];
    $health['status'] = 'unhealthy';
}

// Redis check
try {
    $redis = new Redis();
    $redis->connect('127.0.0.1', 6379);
    $redis->ping();

    $health['checks']['redis'] = [
        'status' => 'healthy',
    ];
} catch (Exception $e) {
    $health['checks']['redis'] = [
        'status' => 'degraded',
        'error' => 'Redis unavailable - caching disabled',
    ];
}

// Storage check
try {
    $cacheDir = __DIR__ . '/../data/cache';
    $testFile = $cacheDir . '/health_check_' . getmypid();

    file_put_contents($testFile, time());
    $readBack = file_get_contents($testFile);
    unlink($testFile);

    $health['checks']['storage'] = [
        'status' => 'healthy',
        'writable' => ($readBack !== false),
    ];
} catch (Exception $e) {
    $health['checks']['storage'] = [
        'status' => 'unhealthy',
        'error' => $e->getMessage(),
    ];
    $health['status'] = 'unhealthy';
}

// Disk space check
$diskFree = disk_free_space(__DIR__);
$diskTotal = disk_total_space(__DIR__);
$diskPercent = ($diskTotal - $diskFree) / $diskTotal * 100;

$health['checks']['disk'] = [
    'status' => $diskPercent > 90 ? 'unhealthy' : ($diskPercent > 80 ? 'warning' : 'healthy'),
    'used_percent' => round($diskPercent, 1),
    'free_bytes' => $diskFree,
];

if ($diskPercent > 90) {
    $health['status'] = 'unhealthy';
}

// Memory check
$memoryLimit = ini_get('memory_limit');
$memoryUsage = memory_get_usage(true);

$health['checks']['memory'] = [
    'limit' => $memoryLimit,
    'usage_bytes' => $memoryUsage,
    'usage_percent' => round($memoryUsage / (int) $memoryLimit * 100, 1),
];

http_response_code($health['status'] === 'healthy' ? 200 : 503);
echo json_encode($health, JSON_PRETTY_PRINT);
```

## Load Balancer Integration

### HAProxy Configuration Helper

```php
<?php
/**
 * HAProxy configuration generator for WHMCS
 */
class HAProxyConfigGenerator
{
    /**
     * Generate HAProxy configuration
     */
    public static function generate(array $servers): string
    {
        $config = <<<'EOCFG'
global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn 4096
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    option redispatch
    retries 3
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

# Health check configuration
backend whmcs_mysql
    mode tcp
    option tcp-check
    tcp-check expect string ^200\ OK
    balance roundrobin

EOCFG;

        // Add server entries
        $serverNum = 1;
        foreach ($servers as $server) {
            $isPrimary = ($server['role'] ?? '') === 'primary';
            $backup = $isPrimary ? '' : 'backup';
            $checkInter = $isPrimary ? '3s' : '5s';

            $config .= "    server mysql{$serverNum} {$server['host']}:{$server['port']} ";
            $config .= "check inter {$checkInter} rise 2 fall 3 {$backup}\n";
            $serverNum++;
        }

        return $config;
    }

    /**
     * Parse HAProxy stats
     */
    public static function parseStats(string $statsOutput): array
    {
        $lines = explode("\n", trim($statsOutput));
        $servers = [];

        foreach ($lines as $line) {
            if (strpos($line, '#') === 0) continue;

            $fields = explode(',', $line);
            if (count($fields) < 17) continue;

            $servers[] = [
                'name' => $fields[0],
                'status' => $fields[1],
                'server_ip' => $fields[16],
                'server_port' => $fields[17] ?? null,
            ];
        }

        return $servers;
    }
}
```

## Application-Level Failover

### WHMCS Database Wrapper with Failover

```php
<?php
/**
 * WHMCS database wrapper with automatic failover
 */
class WHMCSDatabaseWrapper
{
    private ?DatabaseFailoverManager $failoverManager = null;
    private ConnectionPool $connectionPool;

    public function __construct()
    {
        $this->failoverManager = new DatabaseFailoverManager();
        $this->connectionPool = ConnectionPool::getInstance();
    }

    /**
     * Execute query with failover support
     */
    public function query(string $sql, array $params = []): PDOStatement
    {
        $maxRetries = 3;
        $lastException = null;

        for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
            try {
                return $this->connectionPool->execute(function ($connection) use ($sql, $params) {
                    $stmt = $connection->prepare($sql);
                    $stmt->execute($params);
                    return $stmt;
                });
            } catch (PDOException $e) {
                $lastException = $e;

                // Check if it's a connection error
                if ($this->isConnectionError($e) && $attempt < $maxRetries) {
                    logActivity("WHMCS: Database connection error, retrying ({$attempt}/{$maxRetries})");

                    // Trigger failover check
                    $this->failoverManager->healthCheck();

                    // Wait before retry
                    usleep(500000 * $attempt); // 500ms, 1s, 1.5s
                    continue;
                }

                throw $e;
            }
        }

        throw $lastException;
    }

    /**
     * Execute transaction
     */
    public function transaction(callable $callback): mixed
    {
        return $this->connectionPool->execute(function ($connection) use ($callback) {
            try {
                $connection->beginTransaction();
                $result = $callback($connection);
                $connection->commit();
                return $result;
            } catch (Exception $e) {
                $connection->rollBack();
                throw $e;
            }
        });
    }

    /**
     * Get last insert ID
     */
    public function lastInsertId(): string
    {
        return $this->connectionPool->execute(function ($connection) {
            return $connection->lastInsertId();
        });
    }

    private function isConnectionError(PDOException $e): bool
    {
        $connectionErrors = [
            'SQLSTATE[HY000]',
            'Lost connection',
            'Connection refused',
            'Connection timed out',
            'Gone away',
        ];

        foreach ($connectionErrors as $error) {
            if (strpos($e->getMessage(), $error) !== false) {
                return true;
            }
        }

        return false;
    }
}
```

## Use Cases

1. **Planned Maintenance**: Zero-downtime database upgrades
2. **Hardware Failure**: Automatic recovery from server crashes
3. **Network Issues**: Transparent reconnection during outages
4. **Geographic Redundancy**: Disaster recovery across data centers
5. **Load Spikes**: Connection pooling to handle traffic bursts

## Best Practices

| Practice | Description |
|----------|-------------|
| Active-passive setup | Simplest HA configuration |
| Regular health checks | Monitor every 5 seconds |
| Connection pooling | Reuse connections efficiently |
| Graceful failover | Minimum disruption during failover |
| Testing | Regular failover drills |
| Monitoring | Alert on failover events |

## Related Patterns

- [Database Replication](./replication-patterns.md) - Replication strategies
- [Backup Strategies](./backup-strategies.md) - Backup approaches
- [Disaster Recovery](./disaster-recovery.md) - Recovery planning
- [Monitoring Techniques](./monitoring-techniques.md) - System monitoring