# Database Sharding Patterns for WHMCS

Database sharding is a horizontal partitioning strategy that distributes data across multiple database servers, enabling WHMCS to scale beyond a single database server's capacity.

## Understanding Sharding Architecture

### Sharding Concepts

```php
<?php
/**
 * Shard configuration for WHMCS
 */
class ShardConfig
{
    public const SHARD_COUNT = 4;
    public const SHARD_KEY_CLIENT = 'client_id';

    /**
     * Shard routing configuration
     */
    public static function getShardConfig(): array
    {
        return [
            'shards' => [
                0 => [
                    'host' => '10.0.0.1',
                    'port' => 3306,
                    'database' => 'whmcs_shard_0',
                    'username' => 'whmcs_user',
                    'password' => 'secure_password',
                    'weight' => 100,
                ],
                1 => [
                    'host' => '10.0.0.2',
                    'port' => 3306,
                    'database' => 'whmcs_shard_1',
                    'username' => 'whmcs_user',
                    'password' => 'secure_password',
                    'weight' => 100,
                ],
                2 => [
                    'host' => '10.0.0.3',
                    'port' => 3306,
                    'database' => 'whmcs_shard_2',
                    'username' => 'whmcs_user',
                    'password' => 'secure_password',
                    'weight' => 100,
                ],
                3 => [
                    'host' => '10.0.0.4',
                    'port' => 3306,
                    'database' => 'whmcs_shard_3',
                    'username' => 'whmcs_user',
                    'password' => 'secure_password',
                    'weight' => 100,
                ],
            ],
            'global_tables' => [
                'tbladmins',
                'tbladmin_roles',
                'tblconfiguration',
                'tblpricing',
                'tblcurrencies',
                'tblproductgroups',
                'tblproducts',
                'tbldomains_global',
                'tbladdons',
            ],
        ];
    }

    /**
     * Calculate shard index from client ID
     */
    public static function getShardForClient(int $clientId): int
    {
        return $clientId % self::SHARD_COUNT;
    }

    /**
     * Get shard connection for client
     */
    public static function getShardConnection(int $clientId): array
    {
        $shardIndex = self::getShardForClient($clientId);
        $config = self::getShardConfig();
        return $config['shards'][$shardIndex];
    }
}
```

## Shard Router Implementation

### Core Shard Router

```php
<?php
/**
 * Shard router for distributing database operations
 */
class ShardRouter
{
    private array $connections = [];
    private array $config;
    private ?PDO $globalConnection = null;

    public function __construct()
    {
        $this->config = ShardConfig::getShardConfig();
        $this->initializeGlobalConnection();
    }

    /**
     * Get connection for specific shard
     */
    public function getShardConnection(int $shardIndex): PDO
    {
        if (!isset($this->connections[$shardIndex])) {
            $this->connections[$shardIndex] = $this->createConnection(
                $this->config['shards'][$shardIndex]
            );
        }

        return $this->connections[$shardIndex];
    }

    /**
     * Get connection for specific client
     */
    public function getConnectionForClient(int $clientId): PDO
    {
        $shardIndex = ShardConfig::getShardForClient($clientId);
        return $this->getShardConnection($shardIndex);
    }

    /**
     * Get global (non-sharded) connection
     */
    public function getGlobalConnection(): PDO
    {
        return $this->globalConnection;
    }

    /**
     * Determine if table is global
     */
    public function isGlobalTable(string $table): bool
    {
        return in_array($table, $this->config['global_tables']);
    }

    /**
     * Route query to appropriate connection(s)
     */
    public function routeQuery(string $table, int $clientId = null): PDO
    {
        if ($this->isGlobalTable($table)) {
            return $this->getGlobalConnection();
        }

        if ($clientId === null) {
            throw new InvalidArgumentException(
                'Client ID required for sharded table: ' . $table
            );
        }

        return $this->getConnectionForClient($clientId);
    }

    /**
     * Execute query across all shards (map operation)
     */
    public function executeOnAllShards(callable $callback): array
    {
        $results = [];

        foreach ($this->config['shards'] as $index => $shard) {
            $connection = $this->getShardConnection($index);
            $results[$index] = $callback($connection);
        }

        return $results;
    }

    private function initializeGlobalConnection(): void
    {
        $globalShard = $this->config['shards'][0];
        $this->globalConnection = $this->createConnection($globalShard);
    }

    private function createConnection(array $config): PDO
    {
        $dsn = sprintf(
            'mysql:host=%s;port=%d;dbname=%s;charset=utf8mb4',
            $config['host'],
            $config['port'],
            $config['database']
        );

        return new PDO($dsn, $config['username'], $config['password'], [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_OBJ,
            PDO::ATTR_EMULATE_PREPARES => false,
        ]);
    }
}
```

## Client Data Sharding

### Sharded Client Repository

```php
<?php
/**
 * Sharded client repository
 */
class ShardedClientRepository
{
    private ShardRouter $router;

    public function __construct(ShardRouter $router)
    {
        $this->router = $router;
    }

    /**
     * Find client by ID
     */
    public function find(int $clientId): ?object
    {
        $connection = $this->router->getConnectionForClient($clientId);
        $stmt = $connection->prepare(
            'SELECT * FROM tblclients WHERE id = ?'
        );
        $stmt->execute([$clientId]);
        return $stmt->fetch() ?: null;
    }

    /**
     * Find client with related data
     */
    public function findWithRelations(int $clientId): ?object
    {
        $connection = $this->router->getConnectionForClient($clientId);

        $client = $this->find($clientId);
        if (!$client) {
            return null;
        }

        // Get related data from same shard
        $client->domains = $this->getClientDomains($clientId);
        $client->invoices = $this->getClientInvoices($clientId);
        $client->services = $this->getClientServices($clientId);
        $client->tickets = $this->getClientTickets($clientId);

        return $client;
    }

    /**
     * Get client domains
     */
    public function getClientDomains(int $clientId): array
    {
        $connection = $this->router->getConnectionForClient($clientId);
        $stmt = $connection->prepare(
            'SELECT * FROM tbldomains WHERE userid = ? ORDER BY registrationdate DESC'
        );
        $stmt->execute([$clientId]);
        return $stmt->fetchAll();
    }

    /**
     * Get client invoices
     */
    public function getClientInvoices(int $clientId): array
    {
        $connection = $this->router->getConnectionForClient($clientId);
        $stmt = $connection->prepare(
            'SELECT * FROM tblinvoices WHERE userid = ? ORDER BY date DESC'
        );
        $stmt->execute([$clientId]);
        return $stmt->fetchAll();
    }

    /**
     * Get client services
     */
    public function getClientServices(int $clientId): array
    {
        $connection = $this->router->getConnectionForClient($clientId);
        $stmt = $connection->prepare(
            'SELECT * FROM tblhosting WHERE userid = ? ORDER BY id DESC'
        );
        $stmt->execute([$clientId]);
        return $stmt->fetchAll();
    }

    /**
     * Get client tickets
     */
    public function getClientTickets(int $clientId): array
    {
        $connection = $this->router->getConnectionForClient($clientId);
        $stmt = $connection->prepare(
            'SELECT * FROM tbltickets WHERE userid = ? ORDER BY created DESC'
        );
        $stmt->execute([$clientId]);
        return $stmt->fetchAll();
    }

    /**
     * Create new client
     */
    public function create(array $data): int
    {
        $clientId = $this->generateClientId();
        $data['id'] = $clientId;
        $data['created_at'] = date('Y-m-d H:i:s');

        $connection = $this->router->getConnectionForClient($clientId);
        $columns = implode(', ', array_keys($data));
        $placeholders = implode(', ', array_fill(0, count($data), '?'));

        $stmt = $connection->prepare(
            "INSERT INTO tblclients ({$columns}) VALUES ({$placeholders})"
        );
        $stmt->execute(array_values($data));

        return $clientId;
    }

    /**
     * Update client
     */
    public function update(int $clientId, array $data): bool
    {
        $data['updated_at'] = date('Y-m-d H:i:s');

        $connection = $this->router->getConnectionForClient($clientId);
        $sets = implode(' = ?, ', array_keys($data)) . ' = ?';

        $stmt = $connection->prepare(
            "UPDATE tblclients SET {$sets} WHERE id = ?"
        );
        $stmt->execute([...array_values($data), $clientId]);

        return $stmt->rowCount() > 0;
    }

    private function generateClientId(): int
    {
        // Use global DB for auto-increment coordination
        $globalConn = $this->router->getGlobalConnection();
        $stmt = $globalConn->query(
            'INSERT INTO tblclient_id_generation () VALUES ()'
        );
        return (int) $globalConn->lastInsertId();
    }
}
```

## Cross-Shard Operations

### Distributed Query Executor

```php
<?php
/**
 * Execute queries across multiple shards
 */
class DistributedQueryExecutor
{
    private ShardRouter $router;

    public function __construct(ShardRouter $router)
    {
        $this->router = $router;
    }

    /**
     * Count records across all shards
     */
    public function countAll(string $table, array $conditions = []): int
    {
        $results = $this->router->executeOnAllShards(function ($connection) use ($table, $conditions) {
            $sql = "SELECT COUNT(*) as cnt FROM {$table}";
            $params = [];

            if (!empty($conditions)) {
                $where = [];
                foreach ($conditions as $column => $value) {
                    $where[] = "{$column} = ?";
                    $params[] = $value;
                }
                $sql .= ' WHERE ' . implode(' AND ', $where);
            }

            $stmt = $connection->prepare($sql);
            $stmt->execute($params);
            return (int) $stmt->fetch()->cnt;
        });

        return array_sum($results);
    }

    /**
     * Search across all shards
     */
    public function searchAll(string $table, string $field, string $query, int $limit = 100): array
    {
        $results = $this->router->executeOnAllShards(function ($connection) use ($table, $field, $query, $limit) {
            $stmt = $connection->prepare(
                "SELECT * FROM {$table} WHERE {$field} LIKE ? LIMIT ?"
            );
            $stmt->execute(["%{$query}%", $limit]);
            return $stmt->fetchAll();
        });

        // Merge and deduplicate results
        $merged = [];
        $seenIds = [];

        foreach ($results as $shardResults) {
            foreach ($shardResults as $row) {
                if (!isset($seenIds[$row->id])) {
                    $seenIds[$row->id] = true;
                    $merged[] = $row;
                }
            }
        }

        return array_slice($merged, 0, $limit);
    }

    /**
     * Aggregate data across shards
     */
    public function aggregate(string $table, string $field, string $function = 'SUM'): array
    {
        $results = $this->router->executeOnAllShards(function ($connection) use ($table, $field, $function) {
            $stmt = $connection->query(
                "SELECT {$function}({$field}) as result FROM {$table}"
            );
            return $stmt->fetch()->result ?? 0;
        });

        return [
            'total' => array_sum($results),
            'by_shard' => $results,
        ];
    }

    /**
     * Paginated search across shards
     */
    public function paginatedSearch(
        string $table,
        string $field,
        string $query,
        int $page,
        int $perPage
    ): array {
        $offset = ($page - 1) * $perPage;
        $results = [];

        $this->router->executeOnAllShards(function ($connection) use ($table, $field, $query, $perPage, &$results) {
            $stmt = $connection->prepare(
                "SELECT * FROM {$table} WHERE {$field} LIKE ? LIMIT ?"
            );
            $stmt->execute(["%{$query}%", $perPage * 10]); // Fetch extra for merging
            $results = array_merge($results, $stmt->fetchAll());
        });

        // Sort and paginate merged results
        usort($results, function ($a, $b) {
            return strcmp($a->$field, $b->$field);
        });

        $total = count($results);
        $paginated = array_slice($results, $offset, $perPage);

        return [
            'data' => $paginated,
            'total' => $total,
            'page' => $page,
            'per_page' => $perPage,
            'total_pages' => ceil($total / $perPage),
        ];
    }
}
```

## Shard Rebalancing

### Shard Migration Tool

```php
<?php
/**
 * Shard rebalancing and migration tool
 */
class ShardMigrationTool
{
    private ShardRouter $router;
    private PDO $sourceConnection;
    private PDO $targetConnection;

    /**
     * Migrate client data to new shard
     */
    public function migrateClient(int $clientId, int $targetShard): array
    {
        $sourceConnection = $this->router->getConnectionForClient($clientId);
        $targetConnection = $this->router->getShardConnection($targetShard);

        $this->sourceConnection = $sourceConnection;
        $this->targetConnection = $targetConnection;

        $migrated = [];
        $errors = [];

        // Tables to migrate per client
        $tables = [
            'tblclients',
            'tblorders',
            'tblhosting',
            'tblhostingaddons',
            'tbldomains',
            'tblinvoices',
            'tblinvoiceitems',
            'tblcredit',
            'tblactivitylog',
            'tbltickets',
            'tblticketreplies',
        ];

        foreach ($tables as $table) {
            try {
                $this->migrateTable($table, $clientId);
                $migrated[$table] = true;
            } catch (Exception $e) {
                $errors[$table] = $e->getMessage();
            }
        }

        return [
            'client_id' => $clientId,
            'source_shard' => ShardConfig::getShardForClient($clientId),
            'target_shard' => $targetShard,
            'migrated_tables' => $migrated,
            'errors' => $errors,
            'completed_at' => date('Y-m-d H:i:s'),
        ];
    }

    private function migrateTable(string $table, int $clientId): void
    {
        // Fetch from source
        $stmt = $this->sourceConnection->prepare(
            "SELECT * FROM {$table} WHERE userid = ?"
        );
        $stmt->execute([$clientId]);
        $rows = $stmt->fetchAll();

        if (empty($rows)) {
            return;
        }

        // Insert into target
        foreach ($rows as $row) {
            $columns = implode(', ', array_keys((array) $row));
            $placeholders = implode(', ', array_fill(0, count((array) $row), '?'));
            $values = array_values((array) $row);

            $insertStmt = $this->targetConnection->prepare(
                "INSERT INTO {$table} ({$columns}) VALUES ({$placeholders})
                 ON DUPLICATE KEY UPDATE " .
                implode(' = VALUES(', array_keys((array) $row)) . ' = VALUES(' .
                implode('), VALUES(', array_keys((array) $row)) . ')'
            );

            $insertStmt->execute($values);
        }

        // Delete from source
        $deleteStmt = $this->sourceConnection->prepare(
            "DELETE FROM {$table} WHERE userid = ?"
        );
        $deleteStmt->execute([$clientId]);
    }

    /**
     * Validate data integrity after migration
     */
    public function validateMigration(int $clientId): array
    {
        $client = Capsule::table('tblclients')->find($clientId);

        return [
            'client_exists' => $client !== null,
            'has_services' => Capsule::table('tblhosting')
                ->where('userid', $clientId)->exists(),
            'has_domains' => Capsule::table('tbldomains')
                ->where('userid', $clientId)->exists(),
            'has_invoices' => Capsule::table('tblinvoices')
                ->where('userid', $clientId)->exists(),
        ];
    }
}
```

## Shard-Aware WHMCS Hook

```php
<?php
/**
 * Hook for shard-aware WHMCS operations
 */
add_hook('ClientAreaPage', 1, function ($vars) {
    if (!isset($_SESSION['uid'])) {
        return;
    }

    $clientId = (int) $_SESSION['uid'];

    // Use shard-aware repository
    $router = new ShardRouter();
    $repository = new ShardedClientRepository($router);

    // Smart caching per shard
    $cacheKey = "client_data_shard_{$clientId}";
    $clientData = App::make('cache')->remember($cacheKey, 300, function () use ($repository, $clientId) {
        return $repository->findWithRelations($clientId);
    });

    return ['clientData' => $clientData];
});
```

## Use Cases

1. **High-Volume Hosting Providers**: Distribute thousands of client accounts across shards
2. **Multi-Tenant SaaS**: Isolate tenant data on dedicated shards
3. **Geographic Distribution**: Place shards in different regions for lower latency
4. **Regulatory Compliance**: Keep certain data in specific jurisdictions
5. **Performance Optimization**: Separate hot data (recent invoices) from cold data

## Best Practices

| Practice | Description |
|----------|-------------|
| Consistent sharding key | Always use client_id as shard key to co-locate related data |
| Global tables strategy | Keep reference data (products, pricing) on single instance |
| Cross-shard queries | Minimize cross-shard operations; batch when needed |
| Connection pooling | Reuse connections to avoid connection exhaustion |
| Monitoring | Track query latencies per shard |
| Backup strategy | Coordinate backups across all shards |

## Related Patterns

- [Database Replication](./replication-patterns.md) - Master-slave setup
- [Failover Strategies](./failover-strategies.md) - High availability
- [Repository Pattern](./repository-pattern.md) - Data access abstraction
- [Caching Strategies](./caching-strategies.md) - Reduce database load