# WHMCS Custom Report Workflow

## Overview
This workflow enables creation of custom reports with flexible queries and visualizations.

## Prerequisites
- WHMCS with database access
- Admin access for custom reports
- Report builder module

## Step-by-Step Process

### Step 1: Create Custom Report Builder
```php
<?php
// /includes/reports/CustomReportBuilder.php

class CustomReportBuilder
{
    private $report = [];
    private $columns = [];
    private $filters = [];
    private $joins = [];
    private $groupBy = [];
    private $orderBy = [];

    /**
     * Add a table and optional alias
     */
    public function from(string $table, string $alias = null): self
    {
        $this->report['from'] = $alias ? "{$table} {$alias}" : $table;
        return $this;
    }

    /**
     * Add columns to select
     */
    public function select(array $columns): self
    {
        foreach ($columns as $alias => $column) {
            if (is_numeric($alias)) {
                $this->columns[] = $column;
            } else {
                $this->columns[] = "{$column} as {$alias}";
            }
        }
        return $this;
    }

    /**
     * Add join clause
     */
    public function join(string $table, string $condition, string $type = 'INNER'): self
    {
        $this->joins[] = "{$type} JOIN {$table} ON {$condition}";
        return $this;
    }

    /**
     * Add where clause
     */
    public function where(string $column, $value, string $operator = '='): self
    {
        $this->filters[] = "{$column} {$operator} ?";
        $this->params[] = $value;
        return $this;
    }

    /**
     * Add group by clause
     */
    public function groupBy(array $columns): self
    {
        $this->groupBy = array_merge($this->groupBy, $columns);
        return $this;
    }

    /**
     * Add order by clause
     */
    public function orderBy(string $column, string $direction = 'ASC'): self
    {
        $this->orderBy[] = "{$column} {$direction}";
        return $this;
    }

    /**
     * Add date range filter
     */
    public function dateRange(string $column, string $start, string $end): self
    {
        $this->filters[] = "{$column} BETWEEN ? AND ?";
        $this->params[] = $start;
        $this->params[] = $end;
        return $this;
    }

    /**
     * Execute the report
     */
    public function execute(int $limit = 1000): array
    {
        $sql = $this->buildSQL();

        return Capsule::select($sql, $this->params ?? []);
    }

    /**
     * Execute and get count
     */
    public function count(): int
    {
        $sql = $this->buildSQL(true);
        $result = Capsule::select($sql, $this->params ?? []);

        return $result[0]->count ?? 0;
    }

    private function buildSQL(bool $countOnly = false): string
    {
        if ($countOnly) {
            $select = 'COUNT(*) as count';
        } else {
            $select = implode(', ', $this->columns) ?: '*';
        }

        $sql = "SELECT {$select} FROM {$this->report['from']}";

        if (!empty($this->joins)) {
            $sql .= ' ' . implode(' ', $this->joins);
        }

        if (!empty($this->filters)) {
            $sql .= ' WHERE ' . implode(' AND ', $this->filters);
        }

        if (!empty($this->groupBy) && !$countOnly) {
            $sql .= ' GROUP BY ' . implode(', ', $this->groupBy);
        }

        if (!empty($this->orderBy) && !$countOnly) {
            $sql .= ' ORDER BY ' . implode(', ', $this->orderBy);
        }

        if (!$countOnly) {
            $sql .= ' LIMIT 1000';
        }

        return $sql;
    }
}
```

### Step 2: Create Report Templates
```php
<?php
// /includes/reports/custom_templates.php

return [
    'sales_by_product' => [
        'title' => 'Sales by Product',
        'description' => 'Revenue and order count by product',
        'builder' => function() {
            return (new CustomReportBuilder())
                ->from('tblproducts p')
                ->select([
                    'product_name' => 'p.name',
                    'orders' => 'COUNT(DISTINCT oi.invoiceid)',
                    'revenue' => 'SUM(oi.amount)',
                    'avg_price' => 'AVG(oi.amount)'
                ])
                ->join('tblinvoiceitems oi ON p.id = oi.relid AND oi.type = "Hosting"')
                ->join('tblinvoices o ON oi.invoiceid = o.id AND o.status = "Paid"')
                ->groupBy(['p.id', 'p.name'])
                ->orderBy('revenue', 'DESC');
        }
    ],

    'client_activity_summary' => [
        'title' => 'Client Activity Summary',
        'description' => 'Overview of client engagement',
        'builder' => function() {
            return (new CustomReportBuilder())
                ->from('tblclients c')
                ->select([
                    'client_name' => "CONCAT(c.firstname, ' ', c.lastname)",
                    'email' => 'c.email',
                    'services' => 'COUNT(h.id)',
                    'total_revenue' => 'SUM(i.total)',
                    'last_login' => 'c.lastlogin'
                ])
                ->leftJoin('tblhosting h ON c.id = h.userid')
                ->leftJoin('tblinvoices i ON c.id = i.userid AND i.status = "Paid"')
                ->groupBy(['c.id', 'c.firstname', 'c.lastname', 'c.email', 'c.lastlogin'])
                ->orderBy('total_revenue', 'DESC');
        }
    ]
];
```

### Step 3: Save Custom Reports
```php
<?php
// /includes/hooks/custom_report_hooks.php

add_hook('AdminHomepage', 1, function($vars) {
    // Check if user has permission for custom reports
    if (!hasPermission('CustomReports')) {
        return [];
    }

    // Get saved custom reports
    $savedReports = Capsule::table('mod_custom_reports')
        ->where('userid', adminId())
        ->orWhere('is_shared', 1)
        ->get();

    return [
        'customReports' => $savedReports
    ];
});
```

## Report Builder Features

| Feature | Description |
|---------|-------------|
| Table Selection | Select base table with aliases |
| Column Selection | Choose columns with aliases |
| Joins | INNER, LEFT, RIGHT joins |
| Filters | Where clauses with operators |
| Grouping | GROUP BY support |
| Sorting | ORDER BY with ASC/DESC |
| Limits | Result limits |

## Related Workflows
- [WHMCS Revenue Report](./whmcs-revenue-report.md)
- [WHMCS Report Automation](./whmcs-report-automation.md)