# WHMCS Database Raw Queries

## Skill Description
Implement secure raw SQL query execution for WHMCS modules with proper parameter binding, query building, and security considerations.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+
- MySQL/MariaDB database
- Understanding of SQL injection prevention

## Step-by-Step Implementation

### 1. Secure Query Builder
```php
<?php
// includes/database/RawQueryBuilder.php

namespace WHMCS\Module\YourModule\Database;

class RawQueryBuilder
{
    private $db;
    private string $query = '';
    private array $bindings = [];
    private array $wheres = [];
    private array $orders = [];
    private ?int $limit = null;
    private ?int $offset = null;
    private array $selects = ['*'];
    private array $joins = [];
    private string $table = '';

    public function __construct(string $table = '')
    {
        global $db;
        $this->db = $db;
        $this->table = $table;
    }

    public static function table(string $table): self
    {
        return new self($table);
    }

    public function select(array $columns): self
    {
        $this->selects = $columns;
        return $this;
    }

    public function where(string $column, $operator, $value = null): self
    {
        if ($value === null) {
            $value = $operator;
            $operator = '=';
        }

        $this->wheres[] = [
            'column' => $column,
            'operator' => $operator,
            'value' => $value,
            'connector' => 'AND'
        ];

        return $this;
    }

    public function orWhere(string $column, $operator, $value = null): self
    {
        if ($value === null) {
            $value = $operator;
            $operator = '=';
        }

        $this->wheres[] = [
            'column' => $column,
            'operator' => $operator,
            'value' => $value,
            'connector' => 'OR'
        ];

        return $this;
    }

    public function whereIn(string $column, array $values): self
    {
        if (empty($values)) {
            $this->wheres[] = [
                'column' => $column,
                'operator' => 'IN',
                'value' => $values,
                'connector' => 'AND'
            ];
            return $this;
        }

        $placeholders = implode(',', array_fill(0, count($values), '?'));
        $this->wheres[] = [
            'column' => $column,
            'operator' => 'IN',
            'value' => $values,
            'connector' => 'AND'
        ];

        return $this;
    }

    public function whereNull(string $column): self
    {
        $this->wheres[] = [
            'column' => $column,
            'operator' => 'IS NULL',
            'value' => null,
            'connector' => 'AND'
        ];

        return $this;
    }

    public function whereNotNull(string $column): self
    {
        $this->wheres[] = [
            'column' => $column,
            'operator' => 'IS NOT NULL',
            'value' => null,
            'connector' => 'AND'
        ];

        return $this;
    }

    public function whereBetween(string $column, $from, $to): self
    {
        $this->wheres[] = [
            'column' => $column,
            'operator' => 'BETWEEN',
            'value' => [$from, $to],
            'connector' => 'AND'
        ];

        return $this;
    }

    public function join(string $table, string $first, string $operator, string $second): self
    {
        $this->joins[] = [
            'type' => 'INNER JOIN',
            'table' => $table,
            'first' => $first,
            'operator' => $operator,
            'second' => $second
        ];

        return $this;
    }

    public function leftJoin(string $table, string $first, string $operator, string $second): self
    {
        $this->joins[] = [
            'type' => 'LEFT JOIN',
            'table' => $table,
            'first' => $first,
            'operator' => $operator,
            'second' => $second
        ];

        return $this;
    }

    public function orderBy(string $column, string $direction = 'ASC'): self
    {
        $this->orders[] = [
            'column' => $column,
            'direction' => strtoupper($direction) === 'DESC' ? 'DESC' : 'ASC'
        ];

        return $this;
    }

    public function limit(int $value): self
    {
        $this->limit = $value;
        return $this;
    }

    public function offset(int $value): self
    {
        $this->offset = $value;
        return $this;
    }

    public function paginate(int $page = 1, int $perPage = 20): array
    {
        $this->limit($perPage);
        $this->offset(($page - 1) * $perPage);

        // Get total count
        $countQuery = clone $this;
        $countQuery->selects = ['COUNT(*) as total'];
        $countQuery->orders = [];
        $countQuery->limit = null;
        $countQuery->offset = null;

        $total = $this->db->select($countQuery->toSql())[0]['total'] ?? 0;

        return [
            'data' => $this->get(),
            'pagination' => [
                'total' => (int) $total,
                'per_page' => $perPage,
                'current_page' => $page,
                'total_pages' => (int) ceil($total / $perPage)
            ]
        ];
    }

    public function get(): array
    {
        $this->db->query($this->toSql());

        return $this->db->fetchAll();
    }

    public function first(): ?array
    {
        $this->limit(1);
        $this->db->query($this->toSql());

        $result = $this->db->fetch();

        return $result ?: null;
    }

    public function count(): int
    {
        $query = clone $this;
        $query->selects = ['COUNT(*) as cnt'];
        $query->orders = [];

        $result = $this->db->select($query->toSql());

        return (int) ($result[0]['cnt'] ?? 0);
    }

    public function sum(string $column): float
    {
        $query = clone $this;
        $query->selects = ["SUM({$column}) as total"];
        $query->orders = [];

        $result = $this->db->select($query->toSql());

        return (float) ($result[0]['total'] ?? 0);
    }

    public function avg(string $column): float
    {
        $query = clone $this;
        $query->selects = ["AVG({$column}) as average"];
        $query->orders = [];

        $result = $this->db->select($query->toSql());

        return (float) ($result[0]['average'] ?? 0);
    }

    public function insert(array $data): int
    {
        $columns = implode(', ', array_keys($data));
        $placeholders = implode(', ', array_fill(0, count($data), '?'));

        $sql = "INSERT INTO {$this->table} ({$columns}) VALUES ({$placeholders})";

        $this->db->query($sql, array_values($data));

        return $this->db->getLastInsertID();
    }

    public function update(array $data): int
    {
        $sets = [];
        foreach (array_keys($data) as $column) {
            $sets[] = "{$column} = ?";
        }

        $sql = "UPDATE {$this->table} SET " . implode(', ', $sets);
        $sql .= $this->buildWhereClause();

        $values = array_merge(array_values($data), $this->bindings);

        $this->db->query($sql, $values);

        return $this->db->affectedRows();
    }

    public function delete(): int
    {
        $sql = "DELETE FROM {$this->table}";
        $sql .= $this->buildWhereClause();

        $this->db->query($sql, $this->bindings);

        return $this->db->affectedRows();
    }

    public function toSql(): string
    {
        $sql = "SELECT " . implode(', ', $this->selects);
        $sql .= " FROM {$this->table}";

        // Add joins
        foreach ($this->joins as $join) {
            $sql .= " {$join['type']} {$join['table']} ON {$join['first']} {$join['operator']} {$join['second']}";
        }

        // Add where
        $sql .= $this->buildWhereClause();

        // Add order
        if (!empty($this->orders)) {
            $sql .= " ORDER BY ";
            $orderParts = [];

            foreach ($this->orders as $order) {
                $orderParts[] = "{$order['column']} {$order['direction']}";
            }

            $sql .= implode(', ', $orderParts);
        }

        // Add limit
        if ($this->limit !== null) {
            $sql .= " LIMIT {$this->limit}";
        }

        // Add offset
        if ($this->offset !== null) {
            $sql .= " OFFSET {$this->offset}";
        }

        return $sql;
    }

    private function buildWhereClause(): string
    {
        if (empty($this->wheres)) {
            return '';
        }

        $sql = ' WHERE ';
        $parts = [];

        foreach ($this->wheres as $index => $where) {
            $column = $where['column'];
            $operator = $where['operator'];
            $value = $where['value'];
            $connector = $index === 0 ? '' : " {$where['connector']} ";

            if ($operator === 'IN' && is_array($value)) {
                $placeholders = implode(',', array_fill(0, count($value), '?'));
                $parts[] = "{$connector}{$column} IN ({$placeholders})";
                $this->bindings = array_merge($this->bindings, $value);
            } elseif ($operator === 'BETWEEN' && is_array($value)) {
                $parts[] = "{$connector}{$column} BETWEEN ? AND ?";
                $this->bindings = array_merge($this->bindings, $value);
            } elseif ($operator === 'IS NULL' || $operator === 'IS NOT NULL') {
                $parts[] = "{$connector}{$column} {$operator}";
            } else {
                $parts[] = "{$connector}{$column} {$operator} ?";
                $this->bindings[] = $value;
            }
        }

        return $sql . implode('', $parts);
    }

    public function getBindings(): array
    {
        return $this->bindings;
    }
}
```

### 2. Advanced Query Examples
```php
<?php
// Example raw query patterns

namespace WHMCS\Module\YourModule\Services;

use WHMCS\Module\YourModule\Database\RawQueryBuilder;

class ReportingService
{
    public function getMonthlyRevenue(int $year, int $month): float
    {
        $startDate = sprintf('%d-%02d-01', $year, $month);
        $endDate = date('Y-m-t', strtotime($startDate));

        return RawQueryBuilder::table('tblinvoices')
            ->select(['SUM(total) as revenue'])
            ->where('status', 'Paid')
            ->whereBetween('date', $startDate, $endDate)
            ->sum('total');
    }

    public function getTopProducts(int $limit = 10): array
    {
        return RawQueryBuilder::table('tblhosting h')
            ->select([
                'p.name as product_name',
                'p.gid',
                'COUNT(*) as total_orders',
                'SUM(CASE WHEN h.domainstatus = "Active" THEN 1 ELSE 0 END) as active_services'
            ])
            ->join('tblproducts p', 'h.packageid', '=', 'p.id')
            ->groupBy('p.id', 'p.name', 'p.gid')
            ->orderBy('total_orders', 'DESC')
            ->limit($limit)
            ->get();
    }

    public function getClientLifetimeValue(int $clientId): array
    {
        $result = RawQueryBuilder::table('tblinvoices')
            ->select([
                'COUNT(*) as total_invoices',
                'SUM(total) as total_spent',
                'AVG(total) as average_invoice',
                'MIN(date) as first_invoice',
                'MAX(date) as last_invoice'
            ])
            ->where('userid', $clientId)
            ->where('status', 'Paid')
            ->first();

        return $result ?? [];
    }

    public function getOverdueInvoicesReport(): array
    {
        return RawQueryBuilder::table('tblinvoices i')
            ->select([
                'i.*',
                'c.firstname',
                'c.lastname',
                'c.email'
            ])
            ->join('tblclients c', 'i.userid', '=', 'c.id')
            ->where(function ($q) {
                $q->where('i.status', 'Overdue')
                  ->orWhere(function ($q2) {
                      $q2->where('i.status', 'Unpaid')
                         ->where('i.duedate', '<', date('Y-m-d'));
                  });
            })
            ->orderBy('i.duedate', 'ASC')
            ->get();
    }

    public function searchClients(string $query): array
    {
        $searchTerm = '%' . $query . '%';

        return RawQueryBuilder::table('tblclients')
            ->select(['id', 'firstname', 'lastname', 'email', 'companyname'])
            ->where(function ($q) use ($searchTerm) {
                $q->where('firstname', 'LIKE', $searchTerm)
                  ->orWhere('lastname', 'LIKE', $searchTerm)
                  ->orWhere('email', 'LIKE', $searchTerm)
                  ->orWhere('companyname', 'LIKE', $searchTerm);
            })
            ->limit(20)
            ->get();
    }

    public function bulkUpdateServiceStatus(array $serviceIds, string $newStatus): int
    {
        if (empty($serviceIds)) {
            return 0;
        }

        return RawQueryBuilder::table('tblhosting')
            ->whereIn('id', $serviceIds)
            ->update([
                'domainstatus' => $newStatus,
                'updated_at' => date('Y-m-d H:i:s')
            ]);
    }

    public function getServiceTerminationStats(string $startDate, string $endDate): array
    {
        return RawQueryBuilder::table('tblhosting_history')
            ->select([
                'termination_date',
                'COUNT(*) as terminated_count',
                'SUM(termination_fee) as total_fees'
            ])
            ->where('termination_date', '>=', $startDate)
            ->where('termination_date', '<=', $endDate)
            ->where('termination_type', '!=', 'None')
            ->groupBy('termination_date')
            ->orderBy('termination_date', 'ASC')
            ->get();
    }
}
```

### 3. Safe Raw Query Wrapper
```php
<?php
// includes/database/SafeRawQuery.php

namespace WHMCS\Module\YourModule\Database;

class SafeRawQuery
{
    private $db;

    public function __construct()
    {
        global $db;
        $this->db = $db;
    }

    /**
     * Execute a safe raw query with parameter binding
     */
    public function query(string $sql, array $params = []): array
    {
        $this->validateQuery($sql);

        $this->db->query($sql, $params);

        return $this->db->fetchAll();
    }

    /**
     * Validate the SQL query for security
     */
    private function validateQuery(string $sql): void
    {
        $sql = trim($sql);
        $upperSql = strtoupper($sql);

        // Only allow SELECT, INSERT, UPDATE, DELETE
        if (!preg_match('/^(SELECT|INSERT|UPDATE|DELETE|REPLACE)\s+/i', $sql)) {
            throw new \InvalidArgumentException('Only SELECT, INSERT, UPDATE, DELETE, and REPLACE queries are allowed');
        }

        // Block dangerous operations
        $blockedPatterns = [
            '/DROP\s+(TABLE|DATABASE)/i',
            '/TRUNCATE\s+/i',
            '/ALTER\s+/i',
            '/CREATE\s+(TABLE|DATABASE|INDEX)/i',
            '/LOAD_FILE\s*\(/i',
            '/INTO\s+(OUTFILE|DUMPFILE)/i',
            '/SHOW\s+(TABLES|DATABASES|CREATE)/i',
            '/INFORMATION_SCHEMA/i',
            '/PROCESSLIST/i'
        ];

        foreach ($blockedPatterns as $pattern) {
            if (preg_match($pattern, $sql)) {
                throw new \InvalidArgumentException('Query contains forbidden operations');
            }
        }
    }

    /**
     * Get a single row
     */
    public function queryOne(string $sql, array $params = []): ?array
    {
        $results = $this->query($sql, $params);
        return $results[0] ?? null;
    }

    /**
     * Get a scalar value
     */
    public function scalar(string $sql, array $params = [])
    {
        $result = $this->queryOne($sql, $params);

        if ($result) {
            return reset($result);
        }

        return null;
    }

    /**
     * Execute multiple queries in a transaction
     */
    public function transaction(callable $callback): mixed
    {
        $this->db->query('START TRANSACTION');

        try {
            $result = $callback($this);
            $this->db->query('COMMIT');
            return $result;
        } catch (\Exception $e) {
            $this->db->query('ROLLBACK');
            throw $e;
        }
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| SQL injection | Always use parameterized queries |
| Unvalidated queries | Use query validation wrapper |
| Missing error handling | Wrap queries in try-catch |
| Transaction failures | Implement automatic rollback |
| Query performance | Use EXPLAIN before execution |

## Security Considerations

1. **Never concatenate user input** - Always use parameter binding
2. **Validate queries** - Use whitelist approach for allowed operations
3. **Limit permissions** - Use database user with minimal permissions
4. **Log queries** - Track executed queries for audit
5. **Block dangerous operations** - Prevent DROP, TRUNCATE, etc.

## Testing Checklist

- [ ] Test parameterized queries
- [ ] Test query validation
- [ ] Test blocked operations
- [ ] Test transaction handling
- [ ] Test error handling
- [ ] Test with empty results
- [ ] Test with NULL values
- [ ] Test with special characters

## Reference Links

- [MySQL Prepared Statements](https://dev.mysql.com/doc/refman/8.0/en/sql-prepared-statements.html)
- [SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [Query Builder Pattern](https://www.martinfowler.com/eaaCatalog/queryObject.html)
