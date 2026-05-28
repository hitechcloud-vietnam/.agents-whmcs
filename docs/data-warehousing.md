# Data Warehousing Concepts

Data warehousing provides centralized storage for business intelligence and analytics, enabling WHMCS installations to support complex reporting and data analysis.

## Data Warehouse Architecture

### Star Schema Design

```php
<?php
/**
 * Star schema dimension
 */
class Dimension
{
    private string $name;
    private array $columns = [];
    private array $surrogateKeys = [];
    private array $naturalKeys = [];

    public function __construct(string $name)
    {
        $this->name = $name;
    }

    public function addColumn(string $name, string $type, ?string $description = null): self
    {
        $this->columns[] = [
            'name' => $name,
            'type' => $type,
            'description' => $description,
        ];

        return $this;
    }

    public function setSurrogateKey(string $column): self
    {
        $this->surrogateKeys[] = $column;
        return $this;
    }

    public function setNaturalKey(array $columns): self
    {
        $this->naturalKeys = $columns;
        return $this;
    }

    public function createTable(): string
    {
        $sql = "CREATE TABLE dim_{$this->name} (\n";
        $sql .= "    {$this->name}_id INT AUTO_INCREMENT PRIMARY KEY,\n";

        foreach ($this->columns as $column) {
            $sql .= "    {$column['name']} {$column['type']}";
            $sql .= $column['description'] ? " COMMENT '{$column['description']}'" : '';
            $sql .= ",\n";
        }

        foreach ($this->naturalKeys as $key) {
            $sql .= "    UNIQUE KEY uk_{$key} ({$key}),\n";
        }

        $sql = rtrim($sql, ",\n");
        $sql .= "\n) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";

        return $sql;
    }

    public function getName(): string
    {
        return $this->name;
    }
}

/**
 * Fact table definition
 */
class FactTable
{
    private string $name;
    private array $measures = [];
    private array $foreignKeys = [];
    private array $aggregations = [];

    public function __construct(string $name)
    {
        $this->name = $name;
    }

    public function addMeasure(string $name, string $type, string $aggregation = 'SUM'): self
    {
        $this->measures[] = [
            'name' => $name,
            'type' => $type,
            'aggregation' => $aggregation,
        ];

        return $this;
    }

    public function addForeignKey(string $dimensionName, string $column): self
    {
        $this->foreignKeys[] = [
            'dimension' => $dimensionName,
            'column' => $column,
        ];

        return $this;
    }

    public function createTable(): string
    {
        $sql = "CREATE TABLE fact_{$this->name} (\n";
        $sql .= "    fact_id BIGINT AUTO_INCREMENT PRIMARY KEY,\n";

        // Foreign keys
        foreach ($this->foreignKeys as $fk) {
            $sql .= "    {$fk['column']} INT NOT NULL,\n";
        }

        // Measures
        foreach ($this->measures as $measure) {
            $sql .= "    {$measure['name']} {$measure['type']} DEFAULT 0,\n";
        }

        // Date key (usually required)
        $sql .= "    created_date_key INT NOT NULL,\n";

        // Indexes
        foreach ($this->foreignKeys as $fk) {
            $sql .= "    INDEX idx_{$fk['column']} ({$fk['column']}),\n";
        }

        $sql = rtrim($sql, ",\n");
        $sql .= "\n) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";

        return $sql;
    }
}
```

### Dimension Tables

```php
<?php
<?php
/**
 * Date dimension builder
 */
class DateDimensionBuilder
{
    public static function build(DateTime $startDate, DateTime $endDate): array
    {
        $dates = [];
        $current = clone $startDate;

        while ($current <= $endDate) {
            $dates[] = [
                'date_key' => (int) $current->format('Ymd'),
                'date' => $current->format('Y-m-d'),
                'day_of_week' => (int) $current->format('w'),
                'day_name' => $current->format('l'),
                'day_of_month' => (int) $current->format('j'),
                'day_of_year' => (int) $current->format('z') + 1,
                'week_of_year' => (int) $current->format('W'),
                'month' => (int) $current->format('n'),
                'month_name' => $current->format('F'),
                'month_name_short' => $current->format('M'),
                'quarter' => (int) ceil((int) $current->format('n') / 3),
                'year' => (int) $current->format('Y'),
                'year_month' => $current->format('Y-m'),
                'is_weekend' => in_array((int) $current->format('w'), [0, 6]),
                'is_leap_year' => (int) $current->format('L') === 1,
                'fiscal_year' => self::getFiscalYear($current),
            ];

            $current->modify('+1 day');
        }

        return $dates;
    }

    private static function getFiscalYear(DateTime $date): int
    {
        // Assuming fiscal year starts in July
        return (int) $date->format('n') >= 7
            ? (int) $date->format('Y') + 1
            : (int) $date->format('Y');
    }

    public static function createTableSql(): string
    {
        return "CREATE TABLE dim_date (
            date_key INT PRIMARY KEY,
            date DATE NOT NULL,
            day_of_week TINYINT,
            day_name VARCHAR(10),
            day_of_month TINYINT,
            day_of_year SMALLINT,
            week_of_year TINYINT,
            month TINYINT,
            month_name VARCHAR(10),
            month_name_short VARCHAR(3),
            quarter TINYINT,
            year SMALLINT,
            year_month VARCHAR(7),
            is_weekend BOOLEAN,
            is_leap_year BOOLEAN,
            fiscal_year SMALLINT
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    }
}

/**
 * Customer dimension
 */
class DimCustomer
{
    public static function create(): Dimension
    {
        return (new Dimension('customer'))
            ->addColumn('client_id', 'INT', 'WHMCS client ID')
            ->addColumn('customer_type', 'VARCHAR(50)', 'Type of customer')
            ->addColumn('first_name', 'VARCHAR(100)')
            ->addColumn('last_name', 'VARCHAR(100)')
            ->addColumn('full_name', 'VARCHAR(200)', 'Computed full name')
            ->addColumn('email', 'VARCHAR(255)')
            ->addColumn('company_name', 'VARCHAR(255)')
            ->addColumn('country', 'VARCHAR(100)')
            ->addColumn('state', 'VARCHAR(100)')
            ->addColumn('city', 'VARCHAR(100)')
            ->addColumn('registration_date', 'DATE')
            ->addColumn('registration_year', 'INT')
            ->addColumn('lifetime_value', 'DECIMAL(12,2)', 'Total revenue from customer')
            ->addColumn('total_orders', 'INT', 'Count of orders')
            ->addColumn('total_invoices', 'INT', 'Count of invoices')
            ->addColumn('active_services', 'INT', 'Current active services')
            ->setSurrogateKey('customer_key')
            ->setNaturalKey(['client_id']);
    }
}
```

## Slowly Changing Dimensions

### SCD Type 2 Implementation

```php
<?php
<?php
/**
 * Slowly Changing Dimension Type 2 handler
 */
class SlowlyChangingDimension
{
    private string $table;
    private array $trackingColumns = [];
    private array $naturalKeyColumns = [];

    public function __construct(string $table)
    {
        $this->table = $table;
    }

    public function trackColumns(array $columns): self
    {
        $this->trackingColumns = $columns;
        return $this;
    }

    public function naturalKey(array $columns): self
    {
        $this->naturalKeyColumns = $columns;
        return $this;
    }

    /**
     * Process a record and handle SCD type 2
     */
    public function process(array $record): void
    {
        $naturalKey = $this->getNaturalKey($record);

        // Find current active record
        $current = Capsule::table($this->table)
            ->where('is_current', true)
            ->where(function ($query) use ($naturalKey) {
                foreach ($naturalKey as $column => $value) {
                    $query->where($column, $value);
                }
            })
            ->first();

        if ($current) {
            // Check if any tracked columns changed
            $hasChanges = $this->hasChanges($current, $record);

            if ($hasChanges) {
                // Expire current record
                $this->expireCurrent($current->{$this->table . '_key'});

                // Insert new record
                $this->insertNew($record, $naturalKey);
            }
        } else {
            // Insert new record
            $this->insertNew($record, $naturalKey);
        }
    }

    private function getNaturalKey(array $record): array
    {
        $key = [];

        foreach ($this->naturalKeyColumns as $column) {
            $key[$column] = $record[$column] ?? null;
        }

        return $key;
    }

    private function hasChanges($current, array $newRecord): bool
    {
        foreach ($this->trackingColumns as $column) {
            $currentValue = $current->$column ?? null;
            $newValue = $newRecord[$column] ?? null;

            if ((string) $currentValue !== (string) $newValue) {
                return true;
            }
        }

        return false;
    }

    private function expireCurrent(int $surrogateKey): void
    {
        Capsule::table($this->table)
            ->where($this->table . '_key', $surrogateKey)
            ->update([
                'is_current' => false,
                'valid_to' => date('Y-m-d H:i:s'),
            ]);
    }

    private function insertNew(array $record, array $naturalKey): void
    {
        $insertData = array_merge($record, [
            $this->table . '_key' => $this->getNextKey(),
            'is_current' => true,
            'valid_from' => date('Y-m-d H:i:s'),
            'valid_to' => null,
        ]);

        Capsule::table($this->table)->insert($insertData);
    }

    private function getNextKey(): int
    {
        $max = Capsule::table($this->table)->max($this->table . '_key');
        return ($max ?? 0) + 1;
    }
}

/**
 * SCD Type 1 handler (overwrite)
 */
class SlowlyChangingDimensionType1
{
    private string $table;
    private array $naturalKeyColumns = [];

    public function __construct(string $table)
    {
        $this->table = $table;
    }

    public function naturalKey(array $columns): self
    {
        $this->naturalKeyColumns = $columns;
        return $this;
    }

    public function process(array $record): void
    {
        $key = $this->getNaturalKey($record);

        $exists = Capsule::table($this->table)
            ->where(function ($query) use ($key) {
                foreach ($key as $column => $value) {
                    $query->where($column, $value);
                }
            })
            ->exists();

        if ($exists) {
            Capsule::table($this->table)
                ->where(function ($query) use ($key) {
                    foreach ($key as $column => $value) {
                        $query->where($column, $value);
                    }
                })
                ->update($record);
        } else {
            Capsule::table($this->table)->insert($record);
        }
    }

    private function getNaturalKey(array $record): array
    {
        $key = [];

        foreach ($this->naturalKeyColumns as $column) {
            $key[$column] = $record[$column] ?? null;
        }

        return $key;
    }
}
```

## Aggregations

### Aggregation Tables

```php
<?php
<?php
/**
 * Aggregation builder
 */
class AggregationBuilder
{
    private string $name;
    private array $groupByColumns = [];
    private array $measures = [];
    private string $factTable;
    private array $joinTables = [];

    public function __construct(string $name)
    {
        $this->name = $name;
    }

    public function fromFact(string $factTable): self
    {
        $this->factTable = $factTable;
        return $this;
    }

    public function groupBy(string $column): self
    {
        $this->groupByColumns[] = $column;
        return $this;
    }

    public function measure(string $column, string $aggregation, ?string $alias = null): self
    {
        $this->measures[] = [
            'column' => $column,
            'aggregation' => $aggregation,
            'alias' => $alias ?? "{$aggregation}_{$column}",
        ];

        return $this;
    }

    public function join(string $table, string $condition): self
    {
        $this->joinTables[] = ['table' => $table, 'condition' => $condition];
        return $this;
    }

    /**
     * Build the aggregation table
     */
    public function build(): string
    {
        $sql = "CREATE TABLE agg_{$this->name} (\n";
        $sql .= "    agg_key BIGINT AUTO_INCREMENT PRIMARY KEY,\n";

        // Group by columns
        foreach ($this->groupByColumns as $column) {
            $sql .= "    {$column} INT NOT NULL,\n";
        }

        // Measures
        foreach ($this->measures as $measure) {
            $type = str_contains($measure['column'], 'count') ? 'INT' : 'DECIMAL(12,2)';
            $sql .= "    {$measure['alias']} {$type} DEFAULT 0,\n";
        }

        // Date dimension key (required)
        $sql .= "    date_key INT NOT NULL,\n";

        $sql = rtrim($sql, ",\n");
        $sql .= "\n) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4\n\n";

        // Create index
        $indexColumns = implode(', ', $this->groupByColumns);
        $sql .= "CREATE INDEX idx_group_by ON agg_{$this->name} ({$indexColumns});";

        return $sql;
    }

    /**
     * Generate the population query
     */
    public function getPopulateQuery(): string
    {
        $selectParts = $this->groupByColumns;
        $selectParts[] = 'd.date_key';

        foreach ($this->measures as $measure) {
            $selectParts[] = "{$measure['aggregation']}(f.{$measure['column']}) AS {$measure['alias']}";
        }

        $sql = "INSERT INTO agg_{$this->name} (";
        $sql .= implode(', ', $this->groupByColumns);
        $sql .= ", date_key, ";
        $sql .= implode(', ', array_column($this->measures, 'alias'));
        $sql .= ")\n";

        $sql .= "SELECT " . implode(', ', $selectParts) . "\n";
        $sql .= "FROM {$this->factTable} f\n";
        $sql .= "JOIN dim_date d ON DATE(f.created_at) = d.date\n";

        foreach ($this->joinTables as $join) {
            $sql .= "JOIN {$join['table']} ON {$join['condition']}\n";
        }

        $sql .= "GROUP BY " . implode(', ', $this->groupByColumns) . ", d.date_key";

        return $sql;
    }
}

/**
 * Daily sales aggregation
 */
class DailySalesAggregation
{
    public static function create(): AggregationBuilder
    {
        return (new AggregationBuilder('daily_sales'))
            ->fromFact('fact_orders')
            ->groupBy('customer_key')
            ->groupBy('product_key')
            ->measure('order_total', 'SUM', 'total_revenue')
            ->measure('order_count', 'COUNT', 'order_count')
            ->measure('quantity', 'SUM', 'total_quantity')
            ->join('dim_date d', 'DATE(f.order_date) = d.date');
    }
}
```

## Data Warehouse Integration

### WHMCS Data Warehouse

```php
<?php
<?php
/**
 * WHMCS Data Warehouse Manager
 */
class DataWarehouseManager
{
    private string $schema = 'whmcs_dw';

    /**
     * Initialize data warehouse schema
     */
    public function initialize(): void
    {
        // Create dimensions
        $this->createTable(DateDimensionBuilder::createTableSql());
        $this->createTable(DimCustomer::create()->createTable());

        // Create fact tables
        $factOrders = (new FactTable('orders'))
            ->addForeignKey('customer', 'customer_key')
            ->addMeasure('order_total', 'DECIMAL(12,2)')
            ->addMeasure('order_count', 'INT');

        $this->createTable($factOrders->createTable());
    }

    private function createTable(string $sql): void
    {
        try {
            Capsule::statement($sql);
        } catch (\Throwable $e) {
            // Table might already exist
            logActivity("Data warehouse table creation: " . $e->getMessage());
        }
    }

    /**
     * Load date dimension
     */
    public function loadDateDimension(int $yearsBack = 5, int $yearsForward = 1): void
    {
        $startDate = new DateTime("-{$yearsBack} years");
        $endDate = new DateTime("+{$yearsForward} years");

        $dates = DateDimensionBuilder::build($startDate, $endDate);

        $destination = new BulkInsertDestination('dim_date');
        $destination->ignoreDuplicates(true);
        $destination->load($dates);
    }

    /**
     * Load customer dimension
     */
    public function loadCustomerDimension(): int
    {
        $clients = Capsule::table('tblclients')->get();

        $records = array_map(function ($client) {
            // Calculate aggregates
            $lifetimeValue = Capsule::table('tblorders')
                ->where('userid', $client->id)
                ->where('status', 'Completed')
                ->sum('total') ?? 0;

            $totalOrders = Capsule::table('tblorders')
                ->where('userid', $client->id)
                ->count();

            $totalInvoices = Capsule::table('tblinvoices')
                ->where('userid', $client->id)
                ->count();

            $activeServices = Capsule::table('tblhosting')
                ->where('userid', $client->id)
                ->where('domainstatus', 'Active')
                ->count();

            return [
                'client_id' => $client->id,
                'customer_type' => 'standard',
                'first_name' => $client->firstname,
                'last_name' => $client->lastname,
                'full_name' => $client->firstname . ' ' . $client->lastname,
                'email' => $client->email,
                'company_name' => $client->companyname,
                'country' => $client->country,
                'state' => $client->state,
                'city' => $client->city,
                'registration_date' => $client->datecreated,
                'registration_year' => (int) date('Y', strtotime($client->datecreated)),
                'lifetime_value' => $lifetimeValue,
                'total_orders' => $totalOrders,
                'total_invoices' => $totalInvoices,
                'active_services' => $activeServices,
            ];
        }, $clients);

        $destination = new BulkInsertDestination('dim_customer');
        $destination->mapColumns([
            'client_id' => 'client_id',
        ]);
        $destination->ignoreDuplicates(true);
        $destination->onDuplicateKeyUpdate(['first_name', 'last_name', 'full_name', 'company_name']);

        return $destination->load($records);
    }

    /**
     * Load fact orders
     */
    public function loadFactOrders(): int
    {
        $orders = Capsule::table('tblorders')
            ->select('tblorders.*', 'tblclients.id as client_id')
            ->join('tblclients', 'tblclients.id', '=', 'tblorders.userid')
            ->get();

        $records = [];

        foreach ($orders as $order) {
            // Get customer key
            $customerKey = Capsule::table('dim_customer')
                ->where('client_id', $order->userid)
                ->value('customer_key');

            if (!$customerKey) {
                continue;
            }

            $records[] = [
                'customer_key' => $customerKey,
                'order_id' => $order->id,
                'order_total' => $order->total,
                'order_status' => $order->status,
                'date_key' => (int) date('Ymd', strtotime($order->date)),
                'created_date_key' => (int) date('Ymd', strtotime($order->date)),
            ];
        }

        $destination = new BulkInsertDestination('fact_orders');
        $destination->ignoreDuplicates(true);

        return $destination->load($records);
    }
}
```

## Best Practices

1. **Use surrogate keys** - Avoid depending on source system keys
2. **Separate dimensions and facts** - Enable flexible querying
3. **Implement SCD type 2** - Track historical changes
4. **Pre-aggregate data** - Improve query performance
5. **Schedule regular loads** - Keep warehouse current
6. **Index appropriately** - Optimize for common queries
7. **Document schema** - Make it understandable
8. **Test transformations** - Validate data quality

## Related Patterns

- [ETL Patterns](./etl-patterns.md) - Data loading processes
- [Data Pipeline](./data-pipeline.md) - Data processing
- [Analytics Engineering](./analytics-engineering.md) - Analytics implementation
