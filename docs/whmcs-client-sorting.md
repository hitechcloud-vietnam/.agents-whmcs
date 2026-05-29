# WHMCS Client Sorting

## Overview

Client sorting in WHMCS allows administrators to organize client lists by various fields. Sorting helps quickly find specific clients and manage client databases efficiently.

## Sort Options

### Basic Sort Fields

```php
// Common sort options
[
    'id' => 'Client ID',
    'name' => 'Name',
    'email' => 'Email',
    'company' => 'Company',
    'created' => 'Created Date',
    'last_login' => 'Last Login',
    'total_spent' => 'Total Spent'
]
```

### Sort Direction

```php
// Sort order options
[
    'asc' => 'Ascending (A-Z, 0-9)',
    'desc' => 'Descending (Z-A, 9-0)'
]
```

## Default Sorting

### System Default

**Configuration > General > Client Settings**

```php
[
    'default_sort_field' => 'name',
    'default_sort_order' => 'asc',
    'remember_sort' => true
]
```

## Multi-Column Sorting

### Sort by Multiple Fields

```php
// Multi-level sorting
[
    'sort_by' => [
        ['field' => 'group', 'order' => 'asc'],
        ['field' => 'name', 'order' => 'asc']
    ]
]

// Result: Group A clients sorted by name, then Group B clients sorted by name
```

## Common Sort Patterns

### Alphabetical Sort

```php
// Sort by name
['field' => 'name', 'order' => 'asc']
// Results: Adam, Brian, Carol, David

['field' => 'name', 'order' => 'desc']
// Results: David, Carol, Brian, Adam
```

### Date Sort

```php
// Sort by creation date
['field' => 'created', 'order' => 'desc']
// Most recent first

['field' => 'created', 'order' => 'asc']
// Oldest first
```

### Value Sort

```php
// Sort by total spent
['field' => 'total_spent', 'order' => 'desc']
// Highest spenders first

['field' => 'total_spent', 'order' => 'asc']
// Lowest spenders first
```

## Sort in Client List

### Admin Client List

**Admin: Clients > List All Clients**

```php
// Click column headers to sort
[
    'Click ID' => sort by ID,
    'Click Name' => sort by name,
    'Click Email' => sort by email,
    'Click Company' => sort by company,
    'Click Created' => sort by date,
    'Click Actions' => no sort
]
```

### Sort Indicator

```php
// Visual sort indicator
[
    'current_sort' => ['field' => 'name', 'order' => 'asc'],
    'indicator' => 'asc_arrow'    // or 'desc_arrow'
]
```

## Saved Sort Preferences

### Remember User Preference

```php
// Persist sort preference
[
    'admin_id' => 1,
    'default_sort_field' => 'total_spent',
    'default_sort_order' => 'desc',
    'set_by_admin' => true
]
```

## API Sorting

```php
// Sort via API
$result = localAPI('GetClients', [
    'sorting' => 'name-asc'      // field-direction
]);

// Multi-sort
$result = localAPI('GetClients', [
    'sorting' => [
        ['field' => 'group', 'order' => 'asc'],
        ['field' => 'name', 'order' => 'asc']
    ]
]);
```

## Sorting Best Practices

1. **Choose relevant field**: Sort by most useful field for your task
2. **Remember preferences**: Let system remember your preferred sort
3. **Use multi-sort**: Sort by multiple fields for better organization
4. **Check results**: Verify sorted results are correct

## Related Documentation

- [Client Search](./whmcs-client-search.md)
- [Client Filtering](./whmcs-client-filtering.md)
- [Client Tags](./whmcs-client-tags.md)
- [Client Custom Fields](./whmcs-client-custom-fields.md)