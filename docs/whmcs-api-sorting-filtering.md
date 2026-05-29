# WHMCS API Sorting & Filtering

## Overview

Learn how to sort and filter API results efficiently in WHMCS.

## Basic Filtering

### Common Filter Parameters

```php
<?php
// Filter by status
$params = [
    'action' => 'GetClients',
    'status' => 'Active',
];

// Filter by date range
$params = [
    'action' => 'GetOrders',
    'datecreated' => 'asc',
    'startdate' => '2024-01-01',
    'enddate' => '2024-12-31',
];

// Filter by client ID
$params = [
    'action' => 'GetInvoices',
    'userid' => 12345,
];

// Multiple filters
$params = [
    'action' => 'GetServices',
    'clientid' => 12345,
    'domain' => 'example.com',
    'status' => 'Active',
];
```

## Sorting

### Available Sort Parameters

```php
<?php
// Sort by field
$params = [
    'action' => 'GetClients',
    'sorting' => 'firstname', // asc or firstname_desc
];

// Sort by multiple fields
$params = [
    'action' => 'GetInvoices',
    'orderby' => 'duedate|asc',
];

// Common sort fields
$sortFields = [
    'id', 'firstname', 'lastname', 'email',
    'datecreated', 'duedate', 'amount',
    'status', 'domain', 'name',
];
```

## Implementation

### Filter Builder Class

```php
<?php
class WhmcsFilterBuilder {
    private array $filters = [];
    private array $sorting = [];
    private int $limit = 25;
    private int $offset = 0;
    
    public function where(string $field, mixed $value, string $operator = '='): self
    {
        $this->filters[$field] = [
            'value' => $value,
            'operator' => $operator,
        ];
        return $this;
    }
    
    public function whereIn(string $field, array $values): self
    {
        $this->filters[$field] = [
            'value' => $values,
            'operator' => 'IN',
        ];
        return $this;
    }
    
    public function whereBetween(string $field, mixed $start, mixed $end): self
    {
        $this->filters[$field . '_from'] = $start;
        $this->filters[$field . '_to'] = $end;
        return $this;
    }
    
    public function whereLike(string $field, string $pattern): self
    {
        $this->filters[$field] = [
            'value' => '%' . $pattern . '%',
            'operator' => 'LIKE',
        ];
        return $this;
    }
    
    public function orderBy(string $field, string $direction = 'asc'): self
    {
        $this->sorting[$field] = strtolower($direction) === 'desc' ? 'DESC' : 'ASC';
        return $this;
    }
    
    public function limit(int $num, int $start = 0): self
    {
        $this->limit = min($num, 1000);
        $this->offset = $start;
        return $this;
    }
    
    public function build(): array
    {
        $params = [];
        
        foreach ($this->filters as $field => $filter) {
            $params[$field] = $filter['value'];
        }
        
        foreach ($this->sorting as $field => $direction) {
            $params['orderby'] = $field;
            $params['sorting'] = $direction;
            break; // Only first sorting is used
        }
        
        $params['limitnum'] = $this->limit;
        $params['limitstart'] = $this->offset;
        
        return $params;
    }
}

// Usage
$params = (new WhmcsFilterBuilder())
    ->where('status', 'Active')
    ->whereBetween('datecreated', '2024-01-01', '2024-12-31')
    ->orderBy('datecreated', 'desc')
    ->limit(50)
    ->build();
```

### Advanced Query Builder

```php
<?php
class WhmcsQueryBuilder {
    private string $action;
    private array $conditions = [];
    private array $sorting = [];
    private int $page = 1;
    private int $perPage = 25;
    private array $joins = [];
    
    public function __construct(string $action)
    {
        $this->action = $action;
    }
    
    public function select(array $fields): self
    {
        $this->conditions['_fields'] = $fields;
        return $this;
    }
    
    public function where(mixed $field, mixed $value = null): self
    {
        if (is_array($field)) {
            foreach ($field as $k => $v) {
                $this->conditions[$k] = $v;
            }
        } else {
            $this->conditions[$field] = $value;
        }
        return $this;
    }
    
    public function status(string|array $status): self
    {
        if (is_array($status)) {
            $this->conditions['status'] = implode(',', $status);
        } else {
            $this->conditions['status'] = $status;
        }
        return $this;
    }
    
    public function dateRange(string $field, ?string $start = null, ?string $end = null): self
    {
        if ($start) {
            $this->conditions[$field . 'from'] = $start;
        }
        if ($end) {
            $this->conditions[$field . 'to'] = $end;
        }
        return $this;
    }
    
    public function search(string $term, array $fields = []): self
    {
        $this->conditions['search'] = $term;
        if (!empty($fields)) {
            $this->conditions['searchin'] = implode(',', $fields);
        }
        return $this;
    }
    
    public function orderBy(string $field, string $direction = 'ASC'): self
    {
        $this->sorting['orderby'] = $field;
        $this->sorting['sorting'] = strtoupper($direction);
        return $this;
    }
    
    public function paginate(int $page, int $perPage = 25): self
    {
        $this->page = $page;
        $this->perPage = min($perPage, 1000);
        return $this;
    }
    
    public function getParams(): array
    {
        $params = array_merge(
            ['action' => $this->action],
            $this->conditions,
            $this->sorting,
            [
                'limitstart' => ($this->page - 1) * $this->perPage,
                'limitnum' => $this->perPage,
            ]
        );
        
        return array_filter($params, fn($v) => $v !== null && $v !== '');
    }
}

// Usage examples
$query = new WhmcsQueryBuilder('GetClients');

$activeClients = $query
    ->select(['id', 'email', 'firstname', 'lastname'])
    ->status('Active')
    ->search('john')
    ->orderBy('created_at', 'desc')
    ->limit(100)
    ->getParams();

$overdueInvoices = (new WhmcsQueryBuilder('GetInvoices'))
    ->status(['Overdue', 'Unpaid'])
    ->dateRange('duedate', null, date('Y-m-d'))
    ->orderBy('duedate', 'asc')
    ->getParams();
```

### Smart Filters

```php
<?php
class WhmcsSmartFilters {
    
    public static function activeClients(WhmcsApiClient $api): array
    {
        return $api->makeRequest([
            'action' => 'GetClients',
            'status' => 'Active',
            'limitnum' => 10000,
        ]);
    }
    
    public static function overdueInvoices(WhmcsApiClient $api): array
    {
        return $api->makeRequest([
            'action' => 'GetInvoices',
            'status' => 'Overdue',
            'limitnum' => 1000,
        ]);
    }
    
    public static function expiringDomains(WhmcsApiClient $api, int $days = 30): array
    {
        $expiryDate = date('Y-m-d', strtotime("+{$days} days"));
        
        return $api->makeRequest([
            'action' => 'GetDomains',
            'status' => 'Active',
            'expiryfrom' => date('Y-m-d'),
            'expiryto' => $expiryDate,
        ]);
    }
    
    public static function expiringServices(WhmcsApiClient $api, int $days = 30): array
    {
        return $api->makeRequest([
            'action' => 'GetServices',
            'status' => 'Active',
            'domain' => '', // Match all
            'nextduedatefrom' => date('Y-m-d'),
            'nextduedateto' => date('Y-m-d', strtotime("+{$days} days")),
        ]);
    }
    
    public static function recentOrders(WhmcsApiClient $api, int $hours = 24): array
    {
        return $api->makeRequest([
            'action' => 'GetOrders',
            'status' => ['Pending', 'Active'],
            'datecreated' => 'desc',
        ]);
    }
    
    public static function unsuspendedRecently(WhmcsApiClient $api, int $days = 7): array
    {
        $orders = $api->makeRequest([
            'action' => 'GetOrders',
            'limitnum' => 1000,
        ]);
        
        // Filter by recent unsuspension in local processing
        return array_filter($orders['orders'], function($order) use ($days) {
            if (($order['status'] ?? '') !== 'Active') {
                return false;
            }
            // Additional filtering would happen here
            return true;
        });
    }
}
```

### Search Implementation

```php
<?php
class WhmcsSearchQuery {
    private string $term;
    private array $searchFields = ['email', 'firstname', 'lastname', 'companyname'];
    private float $fuzzyThreshold = 0.7;
    
    public function __construct(string $term)
    {
        $this->term = $term;
    }
    
    public function fields(array $fields): self
    {
        $this->searchFields = $fields;
        return $this;
    }
    
    public function exactMatch(bool $exact = true): self
    {
        $this->fuzzyThreshold = $exact ? 1.0 : 0.7;
        return $this;
    }
    
    public function toApiParams(): array
    {
        return [
            'search' => $this->term,
            'searchin' => implode(',', $this->searchFields),
        ];
    }
    
    public function matches(string $value): bool
    {
        if ($this->fuzzyThreshold >= 1.0) {
            return stripos($value, $this->term) !== false;
        }
        
        similar_text(strtolower($this->term), strtolower($value), $percent);
        return $percent >= ($this->fuzzyThreshold * 100);
    }
}

// Usage
$search = (new WhmcsSearchQuery('john'))
    ->fields(['email', 'firstname', 'lastname'])
    ->exactMatch(false);

$params = array_merge(
    ['action' => 'GetClients'],
    $search->toApiParams()
);
```

## Best Practices

1. **Use specific filters** - Narrow results before fetching
2. **Limit result sets** - Avoid fetching entire tables
3. **Index filtering** - Filter by indexed fields (id, status, dates)
4. **Cache common queries** - Store frequently accessed filtered results
5. **Use date ranges wisely** - Avoid open-ended date filters on large tables

## Related Documentation

- [WHMCS API Pagination](/docs/whmcs-api-pagination.md)
- [WHMCS API Batch Operations](/docs/whmcs-api-batch-operations.md)