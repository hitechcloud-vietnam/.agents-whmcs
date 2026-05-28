# Data Pipeline Patterns

Data pipeline patterns provide systematic approaches for moving, transforming, and processing data in WHMCS integrations.

## Pipeline Architecture

### Pipeline Interface

```php
<?php
/**
 * Pipeline interface for data processing
 */
interface PipelineInterface
{
    public function pipe(callable $stage): self;
    public function process($input);
    public function through(array $stages): self;
}
```

### Pipeline Implementation

```php
<?php
/**
 * Data pipeline implementation
 */
class DataPipeline implements PipelineInterface
{
    private array $stages = [];
    private array $middlewares = [];

    /**
     * Add a processing stage
     */
    public function pipe(callable $stage): self
    {
        $this->stages[] = $stage;
        return $this;
    }

    /**
     * Set multiple stages at once
     */
    public function through(array $stages): self
    {
        foreach ($stages as $stage) {
            $this->pipe($stage);
        }
        return $this;
    }

    /**
     * Process data through all stages
     */
    public function process($input)
    {
        $data = $input;

        foreach ($this->stages as $stage) {
            $data = $stage($data);

            // Stop processing if stage returns null
            if ($data === null) {
                return null;
            }
        }

        return $data;
    }

    /**
     * Add middleware (before/after hooks)
     */
    public function middleware(callable $middleware): self
    {
        $this->middlewares[] = $middleware;
        return $this;
    }

    /**
     * Process with middleware support
     */
    public function processWithMiddleware($input)
    {
        $context = new PipelineContext($input);

        // Before middlewares
        foreach ($this->middlewares as $middleware) {
            $middleware($context, 'before');
        }

        // Process stages
        $result = $this->process($context->getData());

        $context->setResult($result);

        // After middlewares
        foreach ($this->middlewares as $middleware) {
            $middleware($context, 'after');
        }

        return $result;
    }
}

/**
 * Pipeline context for passing data between stages
 */
class PipelineContext
{
    private $data;
    private $result;
    private array $metadata = [];

    public function __construct($data)
    {
        $this->data = $data;
    }

    public function getData()
    {
        return $this->data;
    }

    public function setData($data): void
    {
        $this->data = $data;
    }

    public function getResult()
    {
        return $this->result;
    }

    public function setResult($result): void
    {
        $this->result = $result;
    }

    public function getMetadata(string $key, $default = null)
    {
        return $this->metadata[$key] ?? $default;
    }

    public function setMetadata(string $key, $value): void
    {
        $this->metadata[$key] = $value;
    }
}
```

## Common Pipeline Stages

### Data Transformation Stages

```php
<?php
/**
 * Pipeline stage for mapping data
 */
class MapStage
{
    private callable $mapper;

    public function __construct(callable $mapper)
    {
        $this->mapper = $mapper;
    }

    public function __invoke(array $data): array
    {
        return array_map($this->mapper, $data);
    }
}

/**
 * Pipeline stage for filtering data
 */
class FilterStage
{
    private callable $predicate;

    public function __construct(callable $predicate)
    {
        $this->predicate = $predicate;
    }

    public function __invoke(array $data): array
    {
        return array_values(array_filter($data, $this->predicate));
    }
}

/**
 * Pipeline stage for flattening nested arrays
 */
class FlattenStage
{
    public function __invoke(array $data): array
    {
        return array_reduce($data, function ($carry, $item) {
            if (is_array($item)) {
                return array_merge($carry, $item);
            }
            $carry[] = $item;
            return $carry;
        }, []);
    }
}

/**
 * Pipeline stage for grouping data
 */
class GroupByStage
{
    private string $key;

    public function __construct(string $key)
    {
        $this->key = $key;
    }

    public function __invoke(array $data): array
    {
        $grouped = [];

        foreach ($data as $item) {
            $groupKey = $item[$this->key] ?? '_undefined';

            if (!isset($grouped[$groupKey])) {
                $grouped[$groupKey] = [];
            }

            $grouped[$groupKey][] = $item;
        }

        return $grouped;
    }
}

/**
 * Pipeline stage for sorting data
 */
class SortStage
{
    private string $key;
    private string $direction;

    public function __construct(string $key, string $direction = 'asc')
    {
        $this->key = $key;
        $this->direction = strtolower($direction) === 'desc' ? 'desc' : 'asc';
    }

    public function __invoke(array $data): array
    {
        usort($data, function ($a, $b) {
            $aValue = $a[$this->key] ?? null;
            $bValue = $b[$this->key] ?? null;

            $comparison = $aValue <=> $bValue;

            return $this->direction === 'desc' ? -$comparison : $comparison;
        });

        return $data;
    }
}

/**
 * Pipeline stage for selecting fields
 */
class SelectStage
{
    private array $fields;

    public function __construct(array $fields)
    {
        $this->fields = $fields;
    }

    public function __invoke(array $data): array
    {
        return array_map(function ($item) {
            $selected = [];

            foreach ($this->fields as $field) {
                if (isset($item[$field])) {
                    $selected[$field] = $item[$field];
                }
            }

            return $selected;
        }, $data);
    }
}
```

## Data Sources

### Database Source

```php
<?php
/**
 * Database data source for pipelines
 */
class DatabaseSource
{
    private string $table;
    private array $columns = ['*'];
    private array $where = [];
    private array $joins = [];
    private ?string $orderBy = null;
    private int $limit = 1000;
    private int $offset = 0;

    public function __construct(string $table)
    {
        $this->table = $table;
    }

    public function select(array $columns): self
    {
        $this->columns = $columns;
        return $this;
    }

    public function where(string $column, $operator, $value = null): self
    {
        if ($value === null) {
            $value = $operator;
            $operator = '=';
        }

        $this->where[] = [$column, $operator, $value];
        return $this;
    }

    public function join(string $table, string $condition): self
    {
        $this->joins[] = [$table, $condition];
        return $this;
    }

    public function orderBy(string $column, string $direction = 'asc'): self
    {
        $this->orderBy = "{$column} {$direction}";
        return $this;
    }

    public function limit(int $limit): self
    {
        $this->limit = $limit;
        return $this;
    }

    public function offset(int $offset): self
    {
        $this->offset = $offset;
        return $this;
    }

    /**
     * Fetch data in batches
     */
    public function fetchBatch(int $batchSize = 100, callable $callback): int
    {
        $totalProcessed = 0;
        $offset = $this->offset;

        do {
            $query = Capsule::table($this->table)
                ->select($this->columns)
                ->limit($batchSize)
                ->offset($offset);

            foreach ($this->where as [$column, $operator, $value]) {
                $query->where($column, $operator, $value);
            }

            foreach ($this->joins as [$table, $condition]) {
                $query->join($table, $condition);
            }

            if ($this->orderBy) {
                $parts = explode(' ', $this->orderBy);
                $query->orderBy($parts[0], $parts[1] ?? 'asc');
            }

            $records = $query->get();

            if (empty($records)) {
                break;
            }

            $callback($records);
            $totalProcessed += count($records);
            $offset += $batchSize;

        } while (count($records) === $batchSize);

        return $totalProcessed;
    }

    /**
     * Execute and return all results
     */
    public function get(): array
    {
        $query = Capsule::table($this->table)
            ->select($this->columns)
            ->limit($this->limit)
            ->offset($this->offset);

        foreach ($this->where as [$column, $operator, $value]) {
            $query->where($column, $operator, $value);
        }

        if ($this->orderBy) {
            $parts = explode(' ', $this->orderBy);
            $query->orderBy($parts[0], $parts[1] ?? 'asc');
        }

        return $query->get();
    }
}
```

## Data Destinations

### Database Destination

```php
<?php
<?php
/**
 * Database destination for pipeline output
 */
class DatabaseDestination
{
    private string $table;
    private array $columnMapping = [];
    private string $writeMode = 'insert';
    private int $batchSize = 100;

    public function __construct(string $table)
    {
        $this->table = $table;
    }

    public function mapColumns(array $mapping): self
    {
        $this->columnMapping = $mapping;
        return $this;
    }

    public function mode(string $mode): self
    {
        $validModes = ['insert', 'upsert', 'update'];

        if (!in_array($mode, $validModes)) {
            throw new InvalidArgumentException("Invalid mode: {$mode}");
        }

        $this->writeMode = $mode;
        return $this;
    }

    public function batchSize(int $size): self
    {
        $this->batchSize = $size;
        return $this;
    }

    /**
     * Write data to destination
     */
    public function write(array $data): int
    {
        if (empty($data)) {
            return 0;
        }

        $totalWritten = 0;
        $batches = array_chunk($data, $this->batchSize);

        foreach ($batches as $batch) {
            $this->writeBatch($batch);
            $totalWritten += count($batch);
        }

        return $totalWritten;
    }

    private function writeBatch(array $batch): void
    {
        $records = array_map(function ($item) {
            return $this->transformRecord($item);
        }, $batch);

        switch ($this->writeMode) {
            case 'insert':
                Capsule::table($this->table)->insert($records);
                break;

            case 'upsert':
                foreach ($records as $record) {
                    $uniqueKey = $this->columnMapping['unique_key'] ?? 'id';

                    if (isset($record[$uniqueKey])) {
                        $exists = Capsule::table($this->table)
                            ->where($uniqueKey, $record[$uniqueKey])
                            ->exists();

                        if ($exists) {
                            Capsule::table($this->table)
                                ->where($uniqueKey, $record[$uniqueKey])
                                ->update($record);
                        } else {
                            Capsule::table($this->table)->insert($record);
                        }
                    } else {
                        Capsule::table($this->table)->insert($record);
                    }
                }
                break;

            case 'update':
                foreach ($records as $record) {
                    $uniqueKey = $this->columnMapping['unique_key'] ?? 'id';

                    if (isset($record[$uniqueKey])) {
                        $id = $record[$uniqueKey];
                        unset($record[$uniqueKey]);
                        unset($record['id']);

                        Capsule::table($this->table)
                            ->where('id', $id)
                            ->update($record);
                    }
                }
                break;
        }
    }

    private function transformRecord(array $record): array
    {
        if (empty($this->columnMapping)) {
            return $record;
        }

        $transformed = [];

        foreach ($this->columnMapping as $sourceKey => $destKey) {
            if ($sourceKey === 'unique_key') {
                continue;
            }

            $transformed[$destKey] = $record[$sourceKey] ?? null;
        }

        return $transformed;
    }
}
```

## Pipeline Examples

### Client Sync Pipeline

```php
<?php
<?php
/**
 * Client synchronization pipeline
 */
class ClientSyncPipeline
{
    /**
     * Sync clients from external CRM
     */
    public static function syncFromCrm(ApiClient $crmClient): int
    {
        $pipeline = (new DataPipeline())
            ->pipe(function () use ($crmClient) {
                // Stage 1: Fetch from external source
                return $crmClient->get('/contacts');
            })
            ->pipe(new MapStage(function ($contact) {
                // Stage 2: Transform to WHMCS format
                return [
                    'firstname' => $contact['first_name'],
                    'lastname' => $contact['last_name'],
                    'email' => $contact['email'],
                    'companyname' => $contact['company'] ?? '',
                    'phone' => $contact['phone'] ?? '',
                    'address1' => $contact['address']['street'] ?? '',
                    'city' => $contact['address']['city'] ?? '',
                    'state' => $contact['address']['state'] ?? '',
                    'postcode' => $contact['address']['zip'] ?? '',
                    'country' => $contact['address']['country'] ?? '',
                    'external_id' => $contact['id'],
                ];
            }))
            ->pipe(new FilterStage(function ($client) {
                // Stage 3: Filter valid clients
                return !empty($client['email']) && filter_var($client['email'], FILTER_VALIDATE_EMAIL);
            }))
            ->pipe(function ($clients) {
                // Stage 4: Remove duplicates
                $existingEmails = Capsule::table('tblclients')
                    ->select('email')
                    ->whereIn('email', array_column($clients, 'email'))
                    ->pluck('email')
                    ->toArray();

                return array_filter($clients, function ($client) use ($existingEmails) {
                    return !in_array($client['email'], $existingEmails);
                });
            })
            ->pipe(new FlattenStage());

        $newClients = $pipeline->process([]);

        // Write to database
        $destination = new DatabaseDestination('tblclients');
        return $destination->write($newClients);
    }
}
```

### Reporting Pipeline

```php
<?php
/**
 * Monthly reporting pipeline
 */
class MonthlyReportPipeline
{
    public static function generate(array $options = []): array
    {
        $year = $options['year'] ?? date('Y');
        $month = $options['month'] ?? date('m');

        $startDate = "{$year}-{$month}-01 00:00:00";
        $endDate = date('Y-m-t 23:59:59', strtotime($startDate));

        // Source: Orders
        $ordersSource = (new DatabaseSource('tblorders'))
            ->where('date', '>=', $startDate)
            ->where('date', '<=', $endDate);

        // Source: Payments
        $paymentsSource = (new DatabaseSource('tblaccounts'))
            ->where('date', '>=', $startDate)
            ->where('date', '<=', $endDate);

        $pipeline = (new DataPipeline())
            ->pipe(function () use ($ordersSource, $paymentsSource) {
                return [
                    'orders' => $ordersSource->get(),
                    'payments' => $paymentsSource->get(),
                ];
            })
            ->pipe(function ($data) {
                // Calculate metrics
                $totalOrders = count($data['orders']);
                $totalRevenue = array_sum(array_column($data['orders'], 'total'));
                $totalPayments = array_sum(array_column($data['payments'], 'amount'));

                // Group orders by status
                $ordersByStatus = [];
                foreach ($data['orders'] as $order) {
                    $status = $order['status'];
                    if (!isset($ordersByStatus[$status])) {
                        $ordersByStatus[$status] = 0;
                    }
                    $ordersByStatus[$status]++;
                }

                // Group orders by product
                $ordersByProduct = [];
                foreach ($data['orders'] as $order) {
                    $pid = $order['package_id'];
                    if (!isset($ordersByProduct[$pid])) {
                        $ordersByProduct[$pid] = ['count' => 0, 'revenue' => 0];
                    }
                    $ordersByProduct[$pid]['count']++;
                    $ordersByProduct[$pid]['revenue'] += $order['total'];
                }

                return [
                    'period' => [
                        'start' => $startDate ?? null,
                        'end' => $endDate ?? null,
                    ],
                    'metrics' => [
                        'total_orders' => $totalOrders,
                        'total_revenue' => $totalRevenue,
                        'total_payments' => $totalPayments,
                        'average_order_value' => $totalOrders > 0 ? $totalRevenue / $totalOrders : 0,
                    ],
                    'orders_by_status' => $ordersByStatus,
                    'orders_by_product' => $ordersByProduct,
                ];
            });

        return $pipeline->process(null);
    }
}
```

## Best Practices

1. **Make stages pure** - Same input always produces same output
2. **Validate early** - Check data at the start of pipeline
3. **Handle errors gracefully** - Continue or fail based on configuration
4. **Log pipeline execution** - Track data lineage
5. **Use batch processing** - Process large datasets efficiently
6. **Monitor performance** - Track timing of each stage
7. **Make pipelines reusable** - Compose from reusable stages
8. **Version your pipelines** - Track changes over time

## Related Patterns

- [ETL Patterns](./etl-patterns.md) - Extract, transform, load operations
- [Queue Processing](./queue-processing.md) - Async pipeline execution
- [Event Sourcing](./event-sourcing.md) - Event-driven data flows
