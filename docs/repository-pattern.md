# Repository Pattern for Data Access

The Repository Pattern abstracts data access logic, providing a clean interface for working with domain entities.

## Repository Interface

```php
<?php
/**
 * Repository interface
 */
interface RepositoryInterface
{
    /**
     * Find entity by ID
     */
    public function find(int $id): ?object;

    /**
     * Find all entities
     */
    public function findAll(array $criteria = [], int $limit = 100, int $offset = 0): array;

    /**
     * Save entity
     */
    public function save(object $entity): void;

    /**
     * Delete entity
     */
    public function delete(int $id): bool;

    /**
     * Count entities
     */
    public function count(array $criteria = []): int;
}
```

## Base Repository

```php
<?php
/**
 * Abstract base repository
 */
abstract class AbstractRepository implements RepositoryInterface
{
    protected string $table;
    protected string $entityClass;
    protected string $primaryKey = 'id';

    public function find(int $id): ?object
    {
        $data = Capsule::table($this->table)
            ->where($this->primaryKey, $id)
            ->first();

        if (!$data) {
            return null;
        }

        return $this->hydrate($data);
    }

    public function findAll(array $criteria = [], int $limit = 100, int $offset = 0): array
    {
        $query = Capsule::table($this->table);

        foreach ($criteria as $field => $value) {
            if (is_array($value)) {
                $query->whereIn($field, $value);
            } else {
                $query->where($field, $value);
            }
        }

        $records = $query
            ->orderBy($this->primaryKey, 'desc')
            ->limit($limit)
            ->offset($offset)
            ->get();

        return array_map(fn($r) => $this->hydrate($r), $records);
    }

    public function save(object $entity): void
    {
        $data = $this->extract($entity);
        $idField = $this->primaryKey;

        if (isset($entity->$idField) && $entity->$idField) {
            Capsule::table($this->table)
                ->where($this->primaryKey, $entity->$idField)
                ->update($data);
        } else {
            $newId = Capsule::table($this->table)->insertGetId($data);
            $entity->$idField = $newId;
        }
    }

    public function delete(int $id): bool
    {
        return Capsule::table($this->table)
            ->where($this->primaryKey, $id)
            ->delete() > 0;
    }

    public function count(array $criteria = []): int
    {
        $query = Capsule::table($this->table);

        foreach ($criteria as $field => $value) {
            $query->where($field, $value);
        }

        return $query->count();
    }

    /**
     * Hydrate array to entity
     */
    abstract protected function hydrate(stdClass $data): object;

    /**
     * Extract entity to array
     */
    abstract protected function extract(object $entity): array;
}
```

## Client Repository

```php
<?php
/**
 * Client repository interface
 */
interface ClientRepositoryInterface
{
    public function findByEmail(string $email): ?Client;
    public function findByCompany(string $companyName): array;
    public function findActiveClients(): array;
    public function search(string $term): array;
}

/**
 * Client repository implementation
 */
class ClientRepository extends AbstractRepository implements ClientRepositoryInterface
{
    protected string $table = 'tblclients';
    protected string $entityClass = Client::class;

    public function findByEmail(string $email): ?Client
    {
        $data = Capsule::table($this->table)
            ->where('email', $email)
            ->first();

        if (!$data) {
            return null;
        }

        return $this->hydrate($data);
    }

    public function findByCompany(string $companyName): array
    {
        $records = Capsule::table($this->table)
            ->where('companyname', 'LIKE', "%{$companyName}%")
            ->orderBy('companyname', 'asc')
            ->get();

        return array_map(fn($r) => $this->hydrate($r), $records);
    }

    public function findActiveClients(): array
    {
        return $this->findAll(['status' => 'Active']);
    }

    public function search(string $term): array
    {
        $term = '%' . $term . '%';

        $records = Capsule::table($this->table)
            ->where(function ($query) use ($term) {
                $query->where('firstname', 'LIKE', $term)
                    ->orWhere('lastname', 'LIKE', $term)
                    ->orWhere('email', 'LIKE', $term)
                    ->orWhere('companyname', 'LIKE', $term);
            })
            ->orderBy('id', 'desc')
            ->limit(50)
            ->get();

        return array_map(fn($r) => $this->hydrate($r), $records);
    }

    protected function hydrate(stdClass $data): Client
    {
        return Client::fromArray((array) $data);
    }

    protected function extract(object $entity): array
    {
        return $entity->toArray();
    }
}
```

## Order Repository

```php
<?php
/**
 * Order repository with advanced queries
 */
class OrderRepository extends AbstractRepository
{
    protected string $table = 'tblorders';
    protected string $entityClass = Order::class;

    public function findByClient(int $clientId, int $limit = 50): array
    {
        return $this->findAll(['userid' => $clientId], $limit);
    }

    public function findByStatus(string $status): array
    {
        return $this->findAll(['status' => $status]);
    }

    public function findPendingOrders(): array
    {
        return $this->findAll(['status' => 'Pending']);
    }

    public function findRecent(int $days = 30): array
    {
        $cutoff = date('Y-m-d H:i:s', strtotime("-{$days} days"));

        $records = Capsule::table($this->table)
            ->where('date', '>=', $cutoff)
            ->orderBy('date', 'desc')
            ->get();

        return array_map(fn($r) => $this->hydrate($r), $records);
    }

    public function getOrderStats(int $clientId): array
    {
        $stats = Capsule::table($this->table)
            ->where('userid', $clientId)
            ->selectRaw("
                COUNT(*) as total_orders,
                SUM(total) as total_spent,
                MAX(date) as last_order_date
            ")
            ->first();

        return [
            'total_orders' => $stats->total_orders ?? 0,
            'total_spent' => $stats->total_spent ?? 0,
            'last_order_date' => $stats->last_order_date ?? null,
        ];
    }

    protected function hydrate(stdClass $data): Order
    {
        return Order::fromArray((array) $data);
    }

    protected function extract(object $entity): array
    {
        return $entity->toArray();
    }
}
```

## Cached Repository

```php
<?php
/**
 * Cached repository wrapper
 */
class CachedRepository implements RepositoryInterface
{
    private RepositoryInterface $repository;
    private CacheManagerInterface $cache;
    private int $cacheTtl;
    private string $cachePrefix;

    public function __construct(
        RepositoryInterface $repository,
        CacheManagerInterface $cache,
        int $cacheTtl = 3600
    ) {
        $this->repository = $repository;
        $this->cache = $cache;
        $this->cacheTtl = $cacheTtl;
        $this->cachePrefix = 'repo_' . strtolower(basename(str_replace('\\', '/', get_class($repository))));
    }

    public function find(int $id): ?object
    {
        $key = "{$this->cachePrefix}_find_{$id}";

        return $this->cache->remember($key, $this->cacheTtl, function () use ($id) {
            return $this->repository->find($id);
        });
    }

    public function findAll(array $criteria = [], int $limit = 100, int $offset = 0): array
    {
        $key = $this->buildCacheKey('findAll', $criteria, $limit, $offset);

        return $this->cache->remember($key, $this->cacheTtl, function () use ($criteria, $limit, $offset) {
            return $this->repository->findAll($criteria, $limit, $offset);
        });
    }

    public function save(object $entity): void
    {
        $this->repository->save($entity);
        $this->invalidateCache($entity);
    }

    public function delete(int $id): bool
    {
        $result = $this->repository->delete($id);

        if ($result) {
            $this->cache->delete("{$this->cachePrefix}_find_{$id}");
        }

        return $result;
    }

    public function count(array $criteria = []): int
    {
        $key = $this->buildCacheKey('count', $criteria);

        return $this->cache->remember($key, $this->cacheTtl, function () use ($criteria) {
            return $this->repository->count($criteria);
        });
    }

    /**
     * Invalidate cache for entity
     */
    public function invalidate(object $entity): void
    {
        $this->invalidateCache($entity);
    }

    /**
     * Invalidate all cache for this repository
     */
    public function invalidateAll(): void
    {
        $this->cache->invalidateTags([$this->cachePrefix]);
    }

    private function invalidateCache(object $entity): void
    {
        $idField = 'id';

        if (isset($entity->$idField)) {
            $this->cache->delete("{$this->cachePrefix}_find_{$entity->$idField}");
        }

        // Invalidate list caches
        $this->cache->invalidateTags([$this->cachePrefix]);
    }

    private function buildCacheKey(string $method, array $params = [], ...$args): string
    {
        return "{$this->cachePrefix}_{$method}_" . md5(json_encode([$params, $args]));
    }
}
```

## Unit of Work

### Unit of Work Pattern

```php
<?php
/**
 * Unit of Work for managing transactions
 */
class UnitOfWork
{
    private array $newEntities = [];
    private array $dirtyEntities = [];
    private array $removedEntities = [];
    private RepositoryInterface $repository;
    private bool $committed = false;

    public function __construct(RepositoryInterface $repository)
    {
        $this->repository = $repository;
    }

    /**
     * Register new entity
     */
    public function registerNew(object $entity): void
    {
        $this->newEntities[spl_object_id($entity)] = $entity;
        $this->committed = false;
    }

    /**
     * Register dirty entity
     */
    public function registerDirty(object $entity): void
    {
        $id = spl_object_id($entity);

        if (!isset($this->newEntities[$id])) {
            $this->dirtyEntities[$id] = $entity;
        }

        $this->committed = false;
    }

    /**
     * Register removal
     */
    public function registerRemoved(object $entity): void
    {
        $id = spl_object_id($entity);

        unset($this->newEntities[$id]);
        unset($this->dirtyEntities[$id]);

        $this->removedEntities[$id] = $entity;
        $this->committed = false;
    }

    /**
     * Commit all changes
     */
    public function commit(): void
    {
        try {
            Capsule::connection()->transaction(function () {
                // Insert new entities
                foreach ($this->newEntities as $entity) {
                    $this->repository->save($entity);
                }

                // Update dirty entities
                foreach ($this->dirtyEntities as $entity) {
                    $this->repository->save($entity);
                }

                // Remove entities
                foreach ($this->removedEntities as $entity) {
                    $id = $this->getEntityId($entity);
                    $this->repository->delete($id);
                }
            });

            $this->clear();
            $this->committed = true;
        } catch (\Throwable $e) {
            throw new UnitOfWorkException('Commit failed: ' . $e->getMessage(), 0, $e);
        }
    }

    /**
     * Rollback all changes
     */
    public function rollback(): void
    {
        $this->clear();
        $this->committed = false;
    }

    /**
     * Clear tracked changes
     */
    private function clear(): void
    {
        $this->newEntities = [];
        $this->dirtyEntities = [];
        $this->removedEntities = [];
    }

    private function getEntityId(object $entity): int
    {
        $idField = 'id';

        if (!isset($entity->$idField)) {
            throw new UnitOfWorkException('Entity has no ID');
        }

        return (int) $entity->$idField;
    }

    public function isCommitted(): bool
    {
        return $this->committed;
    }

    public function getChangeCount(): int
    {
        return count($this->newEntities)
            + count($this->dirtyEntities)
            + count($this->removedEntities);
    }
}
```

## Specification Pattern

### Query Specification

```php
<?php
/**
 * Specification interface
 */
interface SpecificationInterface
{
    public function isSatisfiedBy(object $candidate): bool;
    public function toQuery(): array;
}

/**
 * Client specification
 */
class ClientSpecification implements SpecificationInterface
{
    private array $criteria = [];

    public static function active(): self
    {
        return new self(['status' => 'Active']);
    }

    public static function withEmail(string $email): self
    {
        return new self(['email' => $email]);
    }

    public static function inCountry(string $country): self
    {
        return new self(['country' => $country]);
    }

    public static function search(string $term): self
    {
        $spec = new self([]);
        $spec->criteria['search'] = $term;
        return $spec;
    }

    public function isSatisfiedBy(object $candidate): bool
    {
        foreach ($this->criteria as $field => $value) {
            if (!isset($candidate->$field) || $candidate->$field !== $value) {
                return false;
            }
        }

        return true;
    }

    public function toQuery(): array
    {
        return $this->criteria;
    }

    private function __construct(array $criteria)
    {
        $this->criteria = $criteria;
    }
}

/**
 * Specification repository
 */
class SpecificationRepository
{
    private ClientRepository $clientRepo;

    public function __construct(ClientRepository $clientRepo)
    {
        $this->clientRepo = $clientRepo;
    }

    public function findBySpecification(SpecificationInterface $spec): array
    {
        return $this->clientRepo->findAll($spec->toQuery());
    }

    public function countBySpecification(SpecificationInterface $spec): int
    {
        return $this->clientRepo->count($spec->toQuery());
    }
}

// Usage
$spec = ClientSpecification::active()->inCountry('US');
$usClients = $specRepo->findBySpecification($spec);
```

## Best Practices

1. **Define interfaces** - Program to abstractions
2. **Single responsibility** - Each repository handles one entity type
3. **Use domain objects** - Return entities, not raw data
4. **Handle relationships** - Load related entities carefully
5. **Cache strategically** - Cache expensive operations
6. **Use unit of work** - Batch related changes
7. **Specification pattern** - Build complex queries composably

## Related Patterns

- [Dependency Injection](./dependency-injection.md) - Repository injection
- [Service Layer](./service-layer.md) - Service use of repositories
- [Cache Strategies](./cache-strategies.md) - Repository caching