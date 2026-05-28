# Analytics Engineering Patterns

Analytics engineering focuses on transforming raw data into analysis-ready datasets using dbt-like patterns adapted for WHMCS.

## Model System

### Analytics Model Base

```php
<?php
/**
 * Base analytics model
 */
abstract class AnalyticsModel
{
    protected string $name;
    protected array $columns = [];
    protected array $schema = [];
    protected string $database = 'analytics';

    abstract public function build(): string;
    abstract public function select(): array;

    public function getFullyQualifiedName(): string
    {
        return "`{$this->database}`.`{$this->name}`";
    }

    public function getColumns(): array
    {
        return $this->columns;
    }

    public function getSchema(): string
    {
        return $this->database;
    }

    /**
     * Execute the model
     */
    public function execute(): array
    {
        $sql = $this->build();

        $results = Capsule::connection()->select($sql);

        return array_map(function ($row) {
            return (array) $row;
        }, $results);
    }

    /**
     * Materialize as a table
     */
    public function materialize(): void
    {
        $sql = $this->build();

        $createSql = "CREATE OR REPLACE TABLE {$this->getFullyQualifiedName()} AS\n{$sql}";

        Capsule::statement($createSql);
    }

    /**
     * Get column definitions
     */
    protected function defineColumns(array $definitions): array
    {
        foreach ($definitions as $name => $type) {
            $this->columns[$name] = $type;
        }

        return $this->columns;
    }
}
```

### Source Models

```php
<?php
<?php
/**
 * WHMCS source definitions
 */
class WhmcsSources
{
    public static function clients(): array
    {
        return [
            'source_table' => 'tblclients',
            'columns' => [
                'id' => 'client_id',
                'firstname' => 'first_name',
                'lastname' => 'last_name',
                'email' => 'email',
                'companyname' => 'company_name',
                'country' => 'country',
                'datecreated' => 'created_at',
                'status' => 'status',
            ],
        ];
    }

    public static function orders(): array
    {
        return [
            'source_table' => 'tblorders',
            'columns' => [
                'id' => 'order_id',
                'userid' => 'client_id',
                'ordernum' => 'order_number',
                'date' => 'order_date',
                'total' => 'order_total',
                'status' => 'status',
                'paymentmethod' => 'payment_method',
            ],
        ];
    }

    public static function invoices(): array
    {
        return [
            'source_table' => 'tblinvoices',
            'columns' => [
                'id' => 'invoice_id',
                'userid' => 'client_id',
                'invoicenum' => 'invoice_number',
                'date' => 'invoice_date',
                'duedate' => 'due_date',
                'total' => 'invoice_total',
                'status' => 'status',
            ],
        ];
    }

    public static function transactions(): array
    {
        return [
            'source_table' => 'tblaccounts',
            'columns' => [
                'id' => 'transaction_id',
                'userid' => 'client_id',
                'invoiceid' => 'invoice_id',
                'date' => 'transaction_date',
                'description' => 'description',
                'amountin' => 'amount_in',
                'amountout' => 'amount_out',
                'fees' => 'fees',
            ],
        ];
    }

    public static function services(): array
    {
        return [
            'source_table' => 'tblhosting',
            'columns' => [
                'id' => 'service_id',
                'userid' => 'client_id',
                'packageid' => 'product_id',
                'domain' => 'domain',
                'regdate' => 'start_date',
                'nextduedate' => 'next_due_date',
                'domainstatus' => 'status',
                'billingcycle' => 'billing_cycle',
            ],
        ];
    }
}

/**
 * Staging model
 */
class StagingModel extends AnalyticsModel
{
    protected array $sourceConfig;
    protected array $selectColumns = [];

    public function __construct(array $sourceConfig)
    {
        $this->name = 'stg_' . $sourceConfig['source_table'];
        $this->sourceConfig = $sourceConfig;
        $this->columns = array_values($sourceConfig['columns']);
    }

    public function build(): string
    {
        $sourceTable = $this->sourceConfig['source_table'];
        $columnMappings = $this->sourceConfig['columns'];

        $selectParts = [];

        foreach ($columnMappings as $sourceCol => $alias) {
            $selectParts[] = "    {$sourceCol} AS {$alias}";
        }

        return "SELECT\n" . implode(",\n", $selectParts) . "\nFROM `{$sourceTable}`";
    }

    public function select(): array
    {
        return $this->selectColumns;
    }

    /**
     * Add a column transformation
     */
    public function addColumn(string $expression, string $alias): self
    {
        $this->selectColumns[] = ['expression' => $expression, 'alias' => $alias];
        return $this;
    }

    /**
     * Add a filter
     */
    public function where(string $condition): self
    {
        $this->schema['where'] = $condition;
        return $this;
    }
}
```

## Intermediate Models

### Intermediate Analytics Models

```php
<?php
<?php
/**
 * Client with orders intermediate model
 */
class IntClientOrders extends AnalyticsModel
{
    protected array $dependencies = ['stg_clients', 'stg_orders'];

    public function __construct()
    {
        $this->name = 'int_client_orders';
        $this->defineColumns([
            'client_id' => 'INT',
            'order_id' => 'INT',
            'order_date' => 'DATE',
            'order_total' => 'DECIMAL(10,2)',
            'order_status' => 'VARCHAR(50)',
        ]);
    }

    public function build(): string
    {
        return <<<SQL
SELECT
    c.client_id,
    o.order_id,
    o.order_date,
    o.order_total,
    o.status AS order_status
FROM stg_clients c
JOIN stg_orders o ON c.client_id = o.client_id
SQL;
    }

    public function select(): array
    {
        return [
            'client_id',
            'order_id',
            'order_date',
            'order_total',
            'order_status',
        ];
    }
}

/**
 * Client revenue intermediate model
 */
class IntClientRevenue extends AnalyticsModel
{
    protected array $dependencies = ['stg_clients', 'stg_transactions'];

    public function __construct()
    {
        $this->name = 'int_client_revenue';
        $this->defineColumns([
            'client_id' => 'INT',
            'total_revenue' => 'DECIMAL(12,2)',
            'total_transactions' => 'INT',
            'first_transaction_date' => 'DATE',
            'last_transaction_date' => 'DATE',
            'average_transaction' => 'DECIMAL(10,2)',
        ]);
    }

    public function build(): string
    {
        return <<<SQL
SELECT
    c.client_id,
    COALESCE(SUM(t.amount_in), 0) AS total_revenue,
    COUNT(t.transaction_id) AS total_transactions,
    MIN(t.transaction_date) AS first_transaction_date,
    MAX(t.transaction_date) AS last_transaction_date,
    COALESCE(AVG(t.amount_in), 0) AS average_transaction
FROM stg_clients c
LEFT JOIN stg_transactions t ON c.client_id = t.client_id
GROUP BY c.client_id
SQL;
    }

    public function select(): array
    {
        return [
            'client_id',
            'total_revenue',
            'total_transactions',
            'first_transaction_date',
            'last_transaction_date',
            'average_transaction',
        ];
    }
}

/**
 * Monthly recurring revenue
 */
class IntMonthlyMrr extends AnalyticsModel
{
    protected array $dependencies = ['stg_services'];

    public function __construct()
    {
        $this->name = 'int_monthly_mrr';
        $this->defineColumns([
            'year_month' => 'VARCHAR(7)',
            'billing_cycle' => 'VARCHAR(50)',
            'service_count' => 'INT',
            'mrr_amount' => 'DECIMAL(12,2)',
        ]);
    }

    public function build(): string
    {
        return <<<SQL
SELECT
    DATE_FORMAT(start_date, '%Y-%m') AS year_month,
    billing_cycle,
    COUNT(*) AS service_count,
    SUM(
        CASE billing_cycle
            WHEN 'Monthly' THEN 1
            WHEN 'Quarterly' THEN 0.33
            WHEN 'Semi-Annually' THEN 0.17
            WHEN 'Annually' THEN 0.083
            ELSE 0
        END
    ) * 100 AS mrr_amount
FROM stg_services
WHERE status = 'Active'
GROUP BY year_month, billing_cycle
SQL;
    }

    public function select(): array
    {
        return [
            'year_month',
            'billing_cycle',
            'service_count',
            'mrr_amount',
        ];
    }
}
```

## Metrics

### Metric Definitions

```php
<?php
<?php
/**
 * Metric definition
 */
class Metric
{
    private string $name;
    private string $description;
    private string $calculation;
    private array $dimensions = [];
    private array $filters = [];
    private string $model;

    public function __construct(string $name, string $calculation, string $model)
    {
        $this->name = $name;
        $this->calculation = $calculation;
        $this->model = $model;
    }

    public function description(string $description): self
    {
        $this->description = $description;
        return $this;
    }

    public function dimension(string $dimension): self
    {
        $this->dimensions[] = $dimension;
        return $this;
    }

    public function filter(string $condition): self
    {
        $this->filters[] = $condition;
        return $this;
    }

    public function build(): string
    {
        $sql = "{$this->calculation} AS {$this->name}";

        if (!empty($this->dimensions)) {
            $sql .= "\nGROUP BY " . implode(', ', $this->dimensions);
        }

        return $sql;
    }

    public function toArray(): array
    {
        return [
            'name' => $this->name,
            'description' => $this->description ?? '',
            'calculation' => $this->calculation,
            'dimensions' => $this->dimensions,
            'filters' => $this->filters,
            'model' => $this->model,
        ];
    }
}

/**
 * Metric registry
 */
class MetricRegistry
{
    private static array $metrics = [];

    public static function register(Metric $metric): void
    {
        self::$metrics[$metric->getName()] = $metric;
    }

    public static function get(string $name): ?Metric
    {
        return self::$metrics[$name] ?? null;
    }

    public static function all(): array
    {
        return self::$metrics;
    }

    public static function getByModel(string $model): array
    {
        return array_filter(self::$metrics, function ($metric) use ($model) {
            return $metric->toArray()['model'] === $model;
        });
    }
}
```

### Predefined Metrics

```php
<?php
<?php
/**
 * Standard WHMCS metrics
 */
class WhmcsMetrics
{
    public static function registerAll(): void
    {
        // Revenue metrics
        MetricRegistry::register(
            (new Metric('total_revenue', 'SUM(order_total)', 'int_client_revenue'))
                ->description('Total revenue from all completed orders')
                ->filter("order_status = 'Completed'")
        );

        MetricRegistry::register(
            (new Metric('total_revenue', 'SUM(amount_in)', 'stg_transactions'))
                ->description('Total payment transactions received')
        );

        MetricRegistry::register(
            (new Metric('mrr', 'SUM(mrr_amount)', 'int_monthly_mrr'))
                ->description('Monthly Recurring Revenue')
        );

        MetricRegistry::register(
            (new Metric('arr', 'SUM(mrr_amount) * 12', 'int_monthly_mrr'))
                ->description('Annual Recurring Revenue')
        );

        // Customer metrics
        MetricRegistry::register(
            (new Metric('total_customers', 'COUNT(DISTINCT client_id)', 'stg_clients'))
                ->description('Total number of customers')
        );

        MetricRegistry::register(
            (new Metric('active_customers', 'COUNT(DISTINCT client_id)', 'stg_services'))
                ->description('Number of customers with active services')
                ->filter("status = 'Active'")
        );

        MetricRegistry::register(
            (new Metric('new_customers', 'COUNT(DISTINCT client_id)', 'stg_clients'))
                ->description('New customers this period')
                ->dimension('created_at')
        );

        // Order metrics
        MetricRegistry::register(
            (new Metric('total_orders', 'COUNT(*)', 'stg_orders'))
                ->description('Total number of orders')
        );

        MetricRegistry::register(
            (new Metric('completed_orders', 'COUNT(*)', 'stg_orders'))
                ->description('Completed orders')
                ->filter("status = 'Completed'")
        );

        MetricRegistry::register(
            (new Metric('pending_orders', 'COUNT(*)', 'stg_orders'))
                ->description('Pending orders')
                ->filter("status = 'Pending'")
        );

        MetricRegistry::register(
            (new Metric('average_order_value', 'AVG(order_total)', 'stg_orders'))
                ->description('Average order value')
                ->filter("status = 'Completed'")
        );

        // Service metrics
        MetricRegistry::register(
            (new Metric('total_services', 'COUNT(*)', 'stg_services'))
                ->description('Total services')
        );

        MetricRegistry::register(
            (new Metric('active_services', 'COUNT(*)', 'stg_services'))
                ->description('Active services')
                ->filter("status = 'Active'")
        );

        MetricRegistry::register(
            (new Metric('churned_services', 'COUNT(*)', 'stg_services'))
                ->description('Terminated services')
                ->filter("status = 'Terminated'")
        );

        // Invoice metrics
        MetricRegistry::register(
            (new Metric('total_invoices', 'COUNT(*)', 'stg_invoices'))
                ->description('Total invoices')
        );

        MetricRegistry::register(
            (new Metric('paid_invoices', 'COUNT(*)', 'stg_invoices'))
                ->description('Paid invoices')
                ->filter("status = 'Paid'")
        );

        MetricRegistry::register(
            (new Metric('overdue_invoices', 'COUNT(*)', 'stg_invoices'))
                ->description('Overdue invoices')
                ->filter("status = 'Unpaid' AND due_date < CURDATE()")
        );
    }
}
```

## Analytics Queries

### Common Analytics Queries

```php
<?php
<?php
/**
 * Analytics query builder
 */
class AnalyticsQuery
{
    private array $metrics = [];
    private array $dimensions = [];
    private array $filters = [];
    private ?DateRange $dateRange = null;
    private string $orderBy = '';
    private int $limit = 1000;

    public function select(array $metrics, array $dimensions = []): self
    {
        $this->metrics = $metrics;
        $this->dimensions = $dimensions;
        return $this;
    }

    public function where(string $condition): self
    {
        $this->filters[] = $condition;
        return $this;
    }

    public function dateRange(DateRange $range): self
    {
        $this->dateRange = $range;
        return $this;
    }

    public function orderBy(string $column, string $direction = 'DESC'): self
    {
        $this->orderBy = "ORDER BY {$column} {$direction}";
        return $this;
    }

    public function limit(int $limit): self
    {
        $this->limit = $limit;
        return $this;
    }

    public function execute(): array
    {
        $sql = $this->build();

        $results = Capsule::connection()->select($sql);

        return array_map(fn($row) => (array) $row, $results);
    }

    public function build(): string
    {
        // Build SELECT clause
        $selectParts = [];

        foreach ($this->dimensions as $dimension) {
            $selectParts[] = $dimension;
        }

        foreach ($this->metrics as $metric) {
            $metricObj = MetricRegistry::get($metric);

            if ($metricObj) {
                $selectParts[] = $metricObj->build();
            } else {
                $selectParts[] = $metric;
            }
        }

        $sql = "SELECT " . implode(",\n       ", $selectParts) . "\n";

        // FROM (use first metric's model as base)
        if (!empty($this->metrics)) {
            $firstMetric = MetricRegistry::get($this->metrics[0]);
            if ($firstMetric) {
                $sql .= "FROM {$firstMetric->toArray()['model']}\n";
            }
        }

        // WHERE
        if (!empty($this->filters)) {
            $sql .= "WHERE " . implode("\n   AND ", $this->filters) . "\n";
        }

        // Date range filter
        if ($this->dateRange) {
            if (!empty($this->filters)) {
                $sql .= "   AND ";
            } else {
                $sql .= "WHERE ";
            }
            $sql .= $this->dateRange->toSql() . "\n";
        }

        // ORDER BY
        if ($this->orderBy) {
            $sql .= $this->orderBy . "\n";
        }

        // LIMIT
        $sql .= "LIMIT {$this->limit}";

        return $sql;
    }
}

/**
 * Date range helper
 */
class DateRange
{
    private ?DateTime $start;
    private ?DateTime $end;
    private string $column = 'created_at';

    public function __construct(?DateTime $start = null, ?DateTime $end = null)
    {
        $this->start = $start ?? new DateTime('-30 days');
        $this->end = $end ?? new DateTime('today');
    }

    public function column(string $column): self
    {
        $this->column = $column;
        return $this;
    }

    public static function lastDays(int $days): self
    {
        return new self(
            new DateTime("-{$days} days"),
            new DateTime('today')
        );
    }

    public static function thisMonth(): self
    {
        return new self(
            new DateTime('first day of this month'),
            new DateTime('today')
        );
    }

    public static function lastMonth(): self
    {
        return new self(
            new DateTime('first day of last month'),
            new DateTime('last day of last month')
        );
    }

    public static function thisYear(): self
    {
        return new self(
            new DateTime('first day of January this year'),
            new DateTime('today')
        );
    }

    public function toSql(): string
    {
        $start = $this->start ? $this->start->format('Y-m-d') : '1970-01-01';
        $end = $this->end ? $this->end->format('Y-m-d') : '2038-01-19';

        return "{$this->column} >= '{$start}' AND {$this->column} <= '{$end}'";
    }
}
```

## Reporting

### Report Generator

```php
<?php
<?php
/**
 * Analytics report generator
 */
class AnalyticsReport
{
    private string $name;
    private array $metrics;
    private array $dimensions;
    private array $filters = [];
    private ?DateRange $dateRange = null;

    public function __construct(string $name, array $metrics, array $dimensions = [])
    {
        $this->name = $name;
        $this->metrics = $metrics;
        $this->dimensions = $dimensions;
    }

    public function filter(string $condition): self
    {
        $this->filters[] = $condition;
        return $this;
    }

    public function dateRange(DateRange $range): self
    {
        $this->dateRange = $range;
        return $this;
    }

    public function execute(): array
    {
        $query = new AnalyticsQuery();
        $query->select($this->metrics, $this->dimensions);

        foreach ($this->filters as $filter) {
            $query->where($filter);
        }

        if ($this->dateRange) {
            $query->dateRange($this->dateRange);
        }

        return $query->execute();
    }

    public function toCsv(): string
    {
        $data = $this->execute();

        if (empty($data)) {
            return '';
        }

        $headers = array_keys($data[0]);
        $lines = [implode(',', $headers)];

        foreach ($data as $row) {
            $lines[] = implode(',', array_values($row));
        }

        return implode("\n", $lines);
    }
}

/**
 * Predefined reports
 */
class WhmcsReports
{
    public static function revenueReport(DateRange $dateRange): AnalyticsReport
    {
        return (new AnalyticsReport('Revenue Report', [
            'total_revenue',
            'total_orders',
            'average_order_value',
        ], ['DATE_FORMAT(order_date, "%Y-%m")']))
            ->filter("order_status = 'Completed'")
            ->dateRange($dateRange);
    }

    public static function customerReport(DateRange $dateRange): AnalyticsReport
    {
        return (new AnalyticsReport('Customer Report', [
            'total_customers',
            'new_customers',
        ], ['DATE_FORMAT(created_at, "%Y-%m")']))
            ->dateRange($dateRange);
    }

    public static function mrrReport(): AnalyticsReport
    {
        return (new AnalyticsReport('MRR Report', [
            'mrr',
            'total_services',
        ], ['year_month']));
    }

    public static function churnReport(DateRange $dateRange): AnalyticsReport
    {
        return (new AnalyticsReport('Churn Report', [
            'churned_services',
            'total_services',
        ], ['DATE_FORMAT(start_date, "%Y-%m")']))
            ->filter("status = 'Terminated'")
            ->dateRange($dateRange);
    }
}
```

## Best Practices

1. **Stage raw data first** - Keep original data untouched
2. **Build incrementally** - Use intermediate models for complex logic
3. **Document metrics** - Clear definitions prevent confusion
4. **Use consistent naming** - Conventions make code readable
5. **Test models** - Validate data at each stage
6. **Version control** - Track changes to analytics code
7. **Parameterize dates** - Enable flexible time ranges
8. **Cache expensive queries** - Improve performance

## Related Patterns

- [Data Warehousing](./data-warehousing.md) - Data storage architecture
- [ETL Patterns](./etl-patterns.md) - Data loading
- [Real-time Analytics](./real-time-analytics.md) - Live data analysis
