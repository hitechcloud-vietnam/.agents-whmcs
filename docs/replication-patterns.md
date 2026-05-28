# Database Replication Patterns for WHMCS

Database replication ensures data redundancy, improves read performance, and provides high availability for WHMCS installations.

## Replication Architecture

### Replication Types Overview

```php
<?php
/**
 * Database replication configuration
 */
class ReplicationConfig
{
    public const REPLICATION_MODES = [
        'async' => 'Asynchronous - writes confirmed immediately, replication eventually consistent',
        'sync' => 'Synchronous - writes confirmed after all replicas acknowledge',
        'semi_sync' => 'Semi-synchronous - confirmed after at least one replica acknowledges',
    ];

    /**
     * Get replication configuration
     */
    public static function getConfig(): array
    {
        return [
            'mode' => 'semi_sync',
            'master' => [
                'host' => '10.0.0.1',
                'port' => 3306,
                'user' => 'whmcs_repl',
                'password' => 'replication_password',
                'tls' => [
                    'enabled' => true,
                    'ca' => '/etc/mysql/certs/ca.pem',
                    'verify' => true,
                ],
            ],
            'replicas' => [
                [
                    'name' => 'replica-1',
                    'host' => '10.0.0.2',
                    'port' => 3306,
                    'priority' => 1,
                    'role' => 'read_replica',
                ],
                [
                    'name' => 'replica-2',
                    'host' => '10.0.0.3',
                    'port' => 3306,
                    'priority' => 2,
                    'role' => 'read_replica',
                ],
                [
                    'name' => 'replica-3',
                    'host' => '10.0.0.4',
                    'port' => 3306,
                    'priority' => 3,
                    'role' => 'hot_standby',
                ],
            ],
            'failover' => [
                'automatic' => true,
                'health_check_interval' => 5,
                'max_retry_attempts' => 3,
                'grace_period' => 30,
            ],
        ];
    }
}
```

## Read Replica Router

### Connection Manager with Replication Support

```php
<?php
/**
 * Replication-aware connection manager
 */
class ReplicationConnectionManager
{
    private ?PDO $masterConnection = null;
    private array $replicaConnections = [];
    private array $config;
    private int $currentReplicaIndex = 0;
    private int $replicaCount = 0;

    public function __construct()
    {
        $this->config = ReplicationConfig::getConfig();
        $this->replicaCount = count($this->config['replicas']);
    }

    /**
     * Get master connection for writes
     */
    public function getMasterConnection(): PDO
    {
        if ($this->masterConnection === null) {
            $this->masterConnection = $this->createConnection(
                $this->config['master'],
                'master'
            );
        }

        return $this->masterConnection;
    }

    /**
     * Get next available read replica (round-robin)
     */
    public function getReadReplica(): PDO
    {
        $replicaIndex = $this->currentReplicaIndex % $this->replicaCount;
        $this->currentReplicaIndex++;

        $replicaConfig = $this->config['replicas'][$replicaIndex];
        $replicaName = $replicaConfig['name'];

        if (!isset($this->replicaConnections[$replicaName])) {
            $this->replicaConnections[$replicaName] = $this->createConnection(
                $replicaConfig,
                'replica'
            );
        }

        return $this->replicaConnections[$replicaName];
    }

    /**
     * Get least loaded replica
     */
    public function getLeastLoadedReplica(): PDO
    {
        $minLoad = PHP_INT_MAX;
        $selectedReplica = null;
        $selectedConfig = null;

        foreach ($this->config['replicas'] as $replicaConfig) {
            if ($replicaConfig['role'] === 'hot_standby') {
                continue; // Skip hot standby for regular reads
            }

            $load = $this->getReplicaLoad($replicaConfig);

            if ($load < $minLoad) {
                $minLoad = $load;
                $selectedReplica = $replicaConfig['name'];
                $selectedConfig = $replicaConfig;
            }
        }

        if ($selectedReplica === null) {
            return $this->getReadReplica();
        }

        if (!isset($this->replicaConnections[$selectedReplica])) {
            $this->replicaConnections[$selectedReplica] = $this->createConnection(
                $selectedConfig,
                'replica'
            );
        }

        return $this->replicaConnections[$selectedReplica];
    }

    /**
     * Get replica by name
     */
    public function getReplicaByName(string $name): ?PDO
    {
        foreach ($this->config['replicas'] as $replicaConfig) {
            if ($replicaConfig['name'] === $name) {
                if (!isset($this->replicaConnections[$name])) {
                    $this->replicaConnections[$name] = $this->createConnection(
                        $replicaConfig,
                        'replica'
                    );
                }
                return $this->replicaConnections[$name];
            }
        }

        return null;
    }

    /**
     * Execute write operation (always master)
     */
    public function write(callable $operation): mixed
    {
        $connection = $this->getMasterConnection();

        try {
            $connection->beginTransaction();
            $result = $operation($connection);
            $connection->commit();
            return $result;
        } catch (Exception $e) {
            $connection->rollBack();
            throw $e;
        }
    }

    /**
     * Execute read from replica
     */
    public function read(callable $operation, bool $consistent = false): mixed
    {
        // For consistent reads, use master
        if ($consistent) {
            return $operation($this->getMasterConnection());
        }

        return $operation($this->getReadReplica());
    }

    /**
     * Health check for all replicas
     */
    public function checkReplicaHealth(): array
    {
        $health = [];

        foreach ($this->config['replicas'] as $replicaConfig) {
            $status = $this->checkConnection($replicaConfig);

            if ($status['connected']) {
                $status['lag'] = $this->getReplicationLag($replicaConfig);
                $status['load'] = $this->getReplicaLoad($replicaConfig);
            }

            $health[$replicaConfig['name']] = $status;
        }

        return $health;
    }

    private function createConnection(array $config, string $type): PDO
    {
        $dsn = sprintf(
            'mysql:host=%s;port=%d;dbname=%s;charset=utf8mb4',
            $config['host'],
            $config['port'],
            $config['database'] ?? 'whmcs'
        );

        $options = [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_OBJ,
            PDO::ATTR_EMULATE_PREPARES => false,
            PDO::ATTR_TIMEOUT => 10,
        ];

        // Add TLS for production
        if (!empty($config['tls']['enabled'])) {
            $options[PDO::MYSQL_ATTR_SSL_CA] = $config['tls']['ca'];
        }

        return new PDO($dsn, $config['user'], $config['password'], $options);
    }

    private function checkConnection(array $config): array
    {
        try {
            $connection = $this->createConnection($config, 'health_check');
            return ['connected' => true, 'latency_ms' => 0];
        } catch (Exception $e) {
            return ['connected' => false, 'error' => $e->getMessage()];
        }
    }

    private function getReplicationLag(array $replicaConfig): ?int
    {
        try {
            $connection = $this->createConnection($replicaConfig, 'lag_check');
            $stmt = $connection->query('SHOW SLAVE STATUS');
            $status = $stmt->fetch();
            return $status ? (int) $status['Seconds_Behind_Master'] : null;
        } catch (Exception $e) {
            return null;
        }
    }

    private function getReplicaLoad(array $config): int
    {
        try {
            $connection = $this->createConnection($config, 'load_check');
            $stmt = $connection->query('SHOW STATUS LIKE "Threads_connected"');
            $status = $stmt->fetch();
            return (int) $status['Value'];
        } catch (Exception $e) {
            return PHP_INT_MAX;
        }
    }
}
```

## Automatic Failover

### Master Failover Manager

```php
<?php
/**
 * Automatic master failover manager
 */
class MasterFailoverManager
{
    private ReplicationConnectionManager $connectionManager;
    private array $config;
    private bool $isFailingOver = false;

    public function __construct(ReplicationConnectionManager $connectionManager)
    {
        $this->connectionManager = $connectionManager;
        $this->config = ReplicationConfig::getConfig();
    }

    /**
     * Start monitoring for failover
     */
    public function startMonitoring(): void
    {
        $interval = $this->config['failover']['health_check_interval'];

        while (true) {
            $health = $this->connectionManager->checkReplicaHealth();

            if (!$this->isMasterHealthy()) {
                $this->initiateFailover();
            }

            sleep($interval);
        }
    }

    /**
     * Initiate failover to best replica
     */
    public function initiateFailover(): bool
    {
        if ($this->isFailingOver) {
            return false;
        }

        $this->isFailingOver = true;

        try {
            logActivity('WHMCS Database: Initiating master failover');

            // Find best replica
            $bestReplica = $this->selectBestReplica();

            if ($bestReplica === null) {
                logActivity('WHMCS Database: No suitable replica for failover');
                return false;
            }

            // Promote replica
            $this->promoteReplica($bestReplica);

            // Update configuration
            $this->updateMasterConfig($bestReplica);

            logActivity('WHMCS Database: Failover completed to ' . $bestReplica['name']);

            return true;
        } catch (Exception $e) {
            logActivity('WHMCS Database: Failover failed - ' . $e->getMessage());
            return false;
        } finally {
            $this->isFailingOver = false;
        }
    }

    /**
     * Select best replica for promotion
     */
    private function selectBestReplica(): ?array
    {
        $health = $this->connectionManager->checkReplicaHealth();
        $candidates = [];

        foreach ($health as $name => $status) {
            if (!$status['connected'] || $status['lag'] > 5) {
                continue; // Skip unhealthy or lagging replicas
            }

            $candidates[] = [
                'name' => $name,
                'lag' => $status['lag'] ?? 0,
                'load' => $status['load'] ?? PHP_INT_MAX,
                'priority' => $this->getReplicaPriority($name),
            ];
        }

        if (empty($candidates)) {
            return null;
        }

        // Sort by priority (desc), then lag (asc), then load (asc)
        usort($candidates, function ($a, $b) {
            if ($a['priority'] !== $b['priority']) {
                return $b['priority'] <=> $a['priority'];
            }
            if ($a['lag'] !== $b['lag']) {
                return $a['lag'] <=> $b['lag'];
            }
            return $a['load'] <=> $b['load'];
        });

        return $candidates[0];
    }

    private function promoteReplica(array $replicaConfig): void
    {
        $connection = $this->connectionManager->getReplicaByName($replicaConfig['name']);

        // Stop replication
        $connection->exec('STOP SLAVE');
        $connection->exec('RESET SLAVE ALL');

        // Configure as master
        $connection->exec("SET @@global.read_only = 0");
        $connection->exec("SET @@global.auto_increment_increment = 1");

        logActivity("WHMCS Database: Promoted replica {$replicaConfig['name']} to master");
    }

    private function updateMasterConfig(array $newMaster): void
    {
        // Update local config (in production, use configuration management)
        $configPath = __DIR__ . '/config.database.php';

        $config = include $configPath;
        $config['master'] = [
            'host' => $newMaster['host'],
            'port' => $newMaster['port'],
            'replaced_at' => date('Y-m-d H:i:s'),
        ];

        // In production, also update DNS/resolver and notify application
    }

    private function isMasterHealthy(): bool
    {
        try {
            $connection = $this->connectionManager->getMasterConnection();
            $stmt = $connection->query('SELECT 1');
            return $stmt->fetch() !== false;
        } catch (Exception $e) {
            return false;
        }
    }

    private function getReplicaPriority(string $name): int
    {
        foreach ($this->config['replicas'] as $replica) {
            if ($replica['name'] === $name) {
                return $replica['priority'];
            }
        }
        return 0;
    }
}
```

## Read-Write Splitting

### Query Router for Read-Write Splitting

```php
<?php
/**
 * Query router with automatic read-write splitting
 */
class QueryRouter
{
    private ReplicationConnectionManager $connectionManager;

    // Patterns that indicate write operations
    private const WRITE_PATTERNS = [
        '/^\s*INSERT/i',
        '/^\s*UPDATE/i',
        '/^\s*DELETE/i',
        '/^\s*REPLACE/i',
        '/^\s*ALTER/i',
        '/^\s*CREATE/i',
        '/^\s*DROP/i',
        '/^\s*TRUNCATE/i',
        '/^\s*GRANT/i',
        '/^\s*REVOKE/i',
    ];

    // Patterns that require consistent reads
    private const CONSISTENT_READ_PATTERNS = [
        '/^\s*SELECT.*FOR\s+UPDATE/i',
        '/^\s*SELECT.*LOCK\s+IN\s+SHARE/i',
        '/^\s*SELECT.*WHERE.*id\s*=\s*\?/i',
    ];

    public function __construct(ReplicationConnectionManager $connectionManager)
    {
        $this->connectionManager = $connectionManager;
    }

    /**
     * Route query to appropriate connection
     */
    public function query(string $sql, array $params = []): PDOStatement
    {
        if ($this->isWriteQuery($sql)) {
            return $this->executeOnMaster($sql, $params);
        }

        if ($this->requiresConsistentRead($sql)) {
            return $this->executeOnMaster($sql, $params);
        }

        return $this->executeOnReplica($sql, $params);
    }

    /**
     * Execute transaction (always on master)
     */
    public function transaction(callable $callback): mixed
    {
        return $this->connectionManager->write(function ($connection) use ($callback) {
            return $callback($connection);
        });
    }

    /**
     * Execute raw query
     */
    public function raw(string $sql, bool $forceMaster = false): PDOStatement
    {
        if ($forceMaster || $this->isWriteQuery($sql)) {
            return $this->getMaster()->query($sql);
        }

        return $this->getReplica()->query($sql);
    }

    private function isWriteQuery(string $sql): bool
    {
        foreach (self::WRITE_PATTERNS as $pattern) {
            if (preg_match($pattern, $sql)) {
                return true;
            }
        }
        return false;
    }

    private function requiresConsistentRead(string $sql): bool
    {
        foreach (self::CONSISTENT_READ_PATTERNS as $pattern) {
            if (preg_match($pattern, $sql)) {
                return true;
            }
        }
        return false;
    }

    private function executeOnMaster(string $sql, array $params): PDOStatement
    {
        $connection = $this->getMaster();
        $stmt = $connection->prepare($sql);
        $stmt->execute($params);
        return $stmt;
    }

    private function executeOnReplica(string $sql, array $params): PDOStatement
    {
        $connection = $this->getReplica();
        $stmt = $connection->prepare($sql);
        $stmt->execute($params);
        return $stmt;
    }

    private function getMaster(): PDO
    {
        return $this->connectionManager->getMasterConnection();
    }

    private function getReplica(): PDO
    {
        return $this->connectionManager->getLeastLoadedReplica();
    }
}
```

## Eloquent Integration

### Replication-Aware Model

```php
<?php
/**
 * Replication-aware Eloquent model
 */
class ReplicationAwareModel extends \Illuminate\Database\Eloquent\Model
{
    protected static ?QueryRouter $queryRouter = null;

    /**
     * Set query router
     */
    public static function setQueryRouter(QueryRouter $router): void
    {
        static::$queryRouter = $router;
    }

    /**
     * Override query building for replication
     */
    protected function newQuery()
    {
        if (static::$queryRouter === null) {
            return parent::newQuery();
        }

        $query = parent::newQuery();

        // Use read replica for SELECT queries
        // The query router will handle routing
        return $query;
    }

    /**
     * Ensure writes go to master
     */
    public function save(array $options = [])
    {
        if (static::$queryRouter) {
            return static::$queryRouter->transaction(function ($connection) {
                $this->setConnection('master');
                return parent::save();
            });
        }

        return parent::save($options);
    }

    /**
     * Consistent read for single record
     */
    public static function findConsistent($id, $columns = ['*'])
    {
        if (static::$queryRouter) {
            return static::$queryRouter->read(function ($connection) use ($id, $columns) {
                static::setConnection('master');
                return parent::find($id, $columns);
            }, true); // true for consistent read
        }

        return parent::find($id, $columns);
    }
}

/**
 * WHMCS Client model with replication support
 */
class Client extends ReplicationAwareModel
{
    protected $table = 'tblclients';
    protected $primaryKey = 'id';

    /**
     * Get client by ID using consistent read
     */
    public static function findConsistentById(int $id): ?self
    {
        return static::findConsistent($id);
    }
}
```

## Lag Monitoring

### Replication Lag Monitor

```php
<?php
/**
 * Monitor and alert on replication lag
 */
class ReplicationLagMonitor
{
    private ReplicationConnectionManager $connectionManager;
    private int $criticalLagThreshold = 10; // seconds
    private int $warningLagThreshold = 5;

    public function __construct(ReplicationConnectionManager $connectionManager)
    {
        $this->connectionManager = $connectionManager;
    }

    /**
     * Check all replicas for lag
     */
    public function checkAllReplicas(): array
    {
        $health = $this->connectionManager->checkReplicaHealth();
        $alerts = [];

        foreach ($health as $name => $status) {
            $alert = $this->evaluateLag($name, $status);
            if ($alert) {
                $alerts[] = $alert;
            }
        }

        return [
            'status' => empty($alerts) ? 'healthy' : 'warning',
            'replicas' => $health,
            'alerts' => $alerts,
            'checked_at' => date('Y-m-d H:i:s'),
        ];
    }

    /**
     * Get replication lag for a specific replica
     */
    public function getReplicaLag(string $replicaName): ?int
    {
        $replica = $this->connectionManager->getReplicaByName($replicaName);

        if ($replica === null) {
            return null;
        }

        $stmt = $replica->query('SHOW SLAVE STATUS');
        $status = $stmt->fetch();

        return $status ? (int) $status['Seconds_Behind_Master'] : null;
    }

    /**
     * Wait for replication to catch up
     */
    public function waitForCatchUp(string $replicaName, int $timeout = 60): bool
    {
        $startTime = time();

        while (time() - $startTime < $timeout) {
            $lag = $this->getReplicaLag($replicaName);

            if ($lag === null) {
                return false; // Replica not replicating
            }

            if ($lag <= 1) {
                return true;
            }

            sleep(1);
        }

        return false;
    }

    private function evaluateLag(string $name, array $status): ?array
    {
        if (!$status['connected']) {
            return [
                'severity' => 'critical',
                'message' => "Replica {$name} is disconnected",
                'replica' => $name,
            ];
        }

        $lag = $status['lag'] ?? null;

        if ($lag === null) {
            return null;
        }

        if ($lag >= $this->criticalLagThreshold) {
            return [
                'severity' => 'critical',
                'message' => "Replica {$name} lag is {$lag}s (critical threshold: {$this->criticalLagThreshold}s)",
                'replica' => $name,
                'lag' => $lag,
            ];
        }

        if ($lag >= $this->warningLagThreshold) {
            return [
                'severity' => 'warning',
                'message' => "Replica {$name} lag is {$lag}s (warning threshold: {$this->warningLagThreshold}s)",
                'replica' => $name,
                'lag' => $lag,
            ];
        }

        return null;
    }
}
```

## Use Cases

1. **Read-Heavy Workloads**: Distribute reads across replicas for faster page loads
2. **Reporting and Analytics**: Run complex queries on dedicated replica
3. **Geographic Distribution**: Place replicas in different regions
4. **Backup Strategy**: Point-in-time recovery from replica
5. **Maintenance without Downtime**: Upgrade master without service interruption

## Best Practices

| Practice | Description |
|----------|-------------|
| Semi-sync replication | Balance between performance and consistency |
| Read-write splitting | Direct writes to master, distribute reads |
| Lag monitoring | Alert when replication falls behind |
| Connection pooling | Reuse connections to reduce overhead |
| Health checks | Regular monitoring of replica health |
| Graceful failover | Automatic promotion with minimal downtime |

## Related Patterns

- [Database Sharding](./database-sharding.md) - Horizontal partitioning
- [Failover Strategies](./failover-strategies.md) - High availability setup
- [Backup Strategies](./backup-strategies.md) - Database backup approaches
- [Caching Strategies](./caching-strategies.md) - Reduce database load