# WHMCS Client Search

## Overview

Client search in WHMCS provides powerful functionality to find clients by various criteria. This includes basic searches, advanced filters, and quick search capabilities for efficient client management.

## Search Interface

### Quick Search

**Admin: Header Search Bar**

```php
// Quick search input
[
    'placeholder' => 'Search clients...',
    'search_fields' => ['email', 'name', 'company', 'client_id'],
    'autocomplete' => true,
    'max_results' => 10
]
```

### Advanced Search

**Admin: Clients > Search Clients**

```php
// Advanced search form
[
    'firstname' => null,
    'lastname' => null,
    'email' => null,
    'company' => null,
    'client_id' => null,
    'address' => null,
    'phone' => null
]
```

## Search Methods

### Basic Search

```php
// Simple search
[
    'term' => 'john',
    'search_in' => ['email', 'firstname', 'lastname', 'company']
]
```

### Field-Specific Search

```php
// Search specific field
[
    'field' => 'email',
    'operator' => 'equals',
    'value' => 'john@example.com'
]
```

## Search Operators

### Available Operators

| Operator | Description | Example |
|----------|-------------|---------|
| equals | Exact match | email = 'john@example.com' |
| contains | Partial match | email contains 'example' |
| starts_with | Begins with | name starts_with 'John' |
| ends_with | Ends with | email ends_with '.com' |
| in | In list | country in ['US', 'UK'] |
| between | Range | created between '2024-01-01' and '2024-05-31' |

### Operator Examples

```php
// Email search
['field' => 'email', 'operator' => 'contains', 'value' => 'gmail']

// Date range
['field' => 'created', 'operator' => 'between', 'value' => ['2024-01-01', '2024-05-31']]

// Multiple values
['field' => 'country', 'operator' => 'in', 'value' => ['US', 'UK', 'CA']]
```

## Advanced Filters

### Client Status Filter

```php
// Filter by status
[
    'field' => 'status',
    'value' => ['Active', 'Inactive']
]
```

### Client Group Filter

```php
// Filter by group
[
    'field' => 'group',
    'value' => 'Premium'
]
```

### Date Filters

```php
// Created date filter
[
    'field' => 'created',
    'operator' => '>=',
    'value' => '2024-01-01'
]

// Last login filter
[
    'field' => 'last_login',
    'operator' => '<=',
    'value' => '2024-03-01'
]
```

### Financial Filters

```php
// Filter by spending
[
    'field' => 'total_spent',
    'operator' => '>=',
    'value' => 1000
]

// Filter by balance
[
    'field' => 'account_balance',
    'operator' => '>',
    'value' => 0
]
```

## Combined Filters

### Multiple Criteria

```php
// Combine multiple filters
[
    'filters' => [
        ['field' => 'status', 'value' => 'Active'],
        ['field' => 'group', 'value' => 'Premium'],
        ['field' => 'total_spent', 'operator' => '>=', 'value' => 500]
    ],
    'match' => 'all'            // all, any
]
```

### Save Filters

```php
// Save filter for reuse
[
    'name' => 'Premium Active High Spenders',
    'filters' => [...],
    'created_by' => 'admin_id'
]
```

## Search Results

### Result Display

```php
// Search results
[
    ['id' => 123, 'name' => 'John Doe', 'email' => 'john@example.com', 'group' => 'Premium'],
    ['id' => 456, 'name' => 'Jane Smith', 'email' => 'jane@example.com', 'group' => 'Premium']
]
```

### Result Columns

```php
// Configure result display
[
    'columns' => ['id', 'name', 'email', 'company', 'group', 'status', 'created'],
    'sort_by' => 'name',
    'sort_order' => 'asc',
    'per_page' => 25
]
```

### Pagination

```php
// Pagination settings
[
    'total_results' => 150,
    'per_page' => 25,
    'current_page' => 1,
    'total_pages' => 6
]
```

## Search Actions

### Bulk Actions

```php
// Actions on search results
[
    'export' => true,
    'send_email' => true,
    'add_to_group' => true,
    'add_tag' => true
]
```

### Individual Actions

```php
// Actions per client
[
    'view' => true,
    'edit' => true,
    'impersonate' => true,
    'delete' => true,
    'add_note' => true
]
```

## Search History

### Recent Searches

```php
// Track search history
[
    ['term' => 'premium', 'results' => 25, 'searched_at' => '2024-05-15'],
    ['term' => 'gmail', 'results' => 10, 'searched_at' => '2024-05-14']
]
```

### Saved Searches

```php
// User saved searches
[
    ['name' => 'High Value Clients', 'filters' => [...], 'run_by' => 'admin_id']
]
```

## Global Search

### Search All Types

```php
// Search across multiple types
[
    'search_clients' => true,
    'search_services' => true,
    'search_domains' => true,
    'search_invoices' => true,
    'search_tickets' => true
]
```

### Unified Results

```php
// Results grouped by type
[
    'clients' => [['id' => 123, 'name' => 'John Doe']],
    'services' => [['id' => 1, 'name' => 'Web Hosting']],
    'domains' => [['id' => 1, 'domain' => 'example.com']],
    'invoices' => [['id' => 5678, 'number' => 'INV-5678']]
]
```

## API Functions

```php
// Search clients
$result = localAPI('SearchClients', [
    'search' => 'john',
    'limit' => 10
]);

// Advanced search
$result = localAPI('GetClients', [
    'limitstart' => 0,
    'limitnum' => 25,
    'sorting' => 'name-asc',
    'filters' => [
        'status' => 'Active',
        'group' => 'Premium'
    ]
]);
```

## Optimization

### Search Performance

```php
// Optimize search
[
    'use_index' => true,
    'limit_results' => true,
    'max_results' => 1000,
    'cache_results' => true,
    'cache_ttl' => 300
]
```

## Best Practices

1. **Use specific terms**: More specific searches return better results
2. **Combine filters**: Use multiple criteria for precise results
3. **Save common searches**: Create saved filters for frequently used searches
4. **Review results**: Verify search results are relevant
5. **Use wildcards**: Use * for partial matches when needed

## Related Documentation

- [Client Filtering](./whmcs-client-filtering.md)
- [Client Sorting](./whmcs-client-sorting.md)
- [Client Tags](./whmcs-client-tags.md)
- [Client Export](./whmcs-client-export.md)