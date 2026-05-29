# WHMCS Pagination Functions

Complete reference for pagination utility functions in WHMCS.

## Overview

WHMCS provides pagination utilities for managing paginated data display in the admin area and client portal.

## Core Pagination Functions

### paginate()

Generic pagination function.

```php
/**
 * Paginate query results
 * 
 * @param Builder $query Query builder instance
 * @param int $page Current page
 * @param int $perPage Items per page
 * @return array Pagination result
 */
function paginate($query, int $page = 1, int $perPage = 20): array
{
    $total = $query->count();
    $offset = ($page - 1) * $perPage;
    
    $items = $query->limit($perPage)->offset($offset)->get();
    
    return [
        'data' => $items,
        'total' => $total,
        'per_page' => $perPage,
        'current_page' => $page,
        'last_page' => ceil($total / $perPage),
        'from' => $offset + 1,
        'to' => min($offset + $perPage, $total),
    ];
}
```

**Example:**
```php
$query = Capsule::table('tblorders')
    ->where('status', 'Active')
    ->orderBy('id', 'desc');

$result = paginate($query, 1, 25);

echo "Page {$result['current_page']} of {$result['last_page']}";
echo "Showing {$result['from']} to {$result['to']} of {$result['total']}";
```

### getPaginationLinks()

Generates pagination links for template.

```php
/**
 * Generate pagination links
 * 
 * @param array $pagination Pagination data
 * @param string $baseUrl Base URL for links
 * @param array $params Additional URL parameters
 * @return string HTML pagination links
 */
function getPaginationLinks(array $pagination, string $baseUrl, array $params = []): string
{
    $current = $pagination['current_page'];
    $last = $pagination['last_page'];
    $html = '<div class="pagination">';
    
    // Previous link
    if ($current > 1) {
        $url = buildPaginationUrl($baseUrl, $current - 1, $params);
        $html .= '<a href="' . $url . '" class="prev">&laquo; Previous</a>';
    }
    
    // Page links
    $range = getPaginationRange($current, $last, 5);
    
    if ($range[0] > 1) {
        $url = buildPaginationUrl($baseUrl, 1, $params);
        $html .= '<a href="' . $url . '">1</a>';
        if ($range[0] > 2) {
            $html .= '<span class="ellipsis">...</span>';
        }
    }
    
    foreach ($range as $page) {
        $url = buildPaginationUrl($baseUrl, $page, $params);
        $active = $page === $current ? ' active' : '';
        $html .= '<a href="' . $url . '" class="page' . $active . '">' . $page . '</a>';
    }
    
    if ($range[count($range) - 1] < $last) {
        if ($range[count($range) - 1] < $last - 1) {
            $html .= '<span class="ellipsis">...</span>';
        }
        $url = buildPaginationUrl($baseUrl, $last, $params);
        $html .= '<a href="' . $url . '">' . $last . '</a>';
    }
    
    // Next link
    if ($current < $last) {
        $url = buildPaginationUrl($baseUrl, $current + 1, $params);
        $html .= '<a href="' . $url . '" class="next">Next &raquo;</a>';
    }
    
    $html .= '</div>';
    
    return $html;
}

/**
 * Build pagination URL
 * 
 * @param string $baseUrl Base URL
 * @param int $page Page number
 * @param array $params Parameters
 * @return string URL
 */
function buildPaginationUrl(string $baseUrl, int $page, array $params): string
{
    $params['page'] = $page;
    return $baseUrl . '?' . http_build_query($params);
}

/**
 * Get pagination range
 * 
 * @param int $current Current page
 * @param int $last Last page
 * @param int $range Range size
 * @return array Page numbers
 */
function getPaginationRange(int $current, int $last, int $range = 5): array
{
    if ($last <= $range) {
        return range(1, $last);
    }
    
    $start = max(1, $current - floor($range / 2));
    $end = min($last, $start + $range - 1);
    
    if ($end - $start < $range - 1) {
        $start = max(1, $end - $range + 1);
    }
    
    return range($start, $end);
}
```

### getOffsetLimit()

Gets offset and limit for queries.

```php
/**
 * Get offset and limit from page info
 * 
 * @param int $page Current page
 * @param int $perPage Items per page
 * @return array ['offset' => int, 'limit' => int]
 */
function getOffsetLimit(int $page, int $perPage): array
{
    return [
        'offset' => ($page - 1) * $perPage,
        'limit' => $perPage
    ];
}
```

## Paginated Data Fetching

### getPaginatedClients()

Gets paginated client list.

```php
/**
 * Get paginated client list
 * 
 * @param array $filters Filter options
 * @param int $page Current page
 * @param int $perPage Items per page
 * @return array Paginated result
 */
function getPaginatedClients(array $filters = [], int $page = 1, int $perPage = 20): array
{
    $query = Capsule::table('tblclients')
        ->orderBy('id', 'desc');
    
    // Apply filters
    if (!empty($filters['search'])) {
        $term = $filters['search'];
        $query->where(function($q) use ($term) {
            $q->where('firstname', 'like', '%' . $term . '%')
              ->orWhere('lastname', 'like', '%' . $term . '%')
              ->orWhere('email', 'like', '%' . $term . '%')
              ->orWhere('companyname', 'like', '%' . $term . '%');
        });
    }
    
    if (!empty($filters['status'])) {
        $query->where('status', $filters['status']);
    }
    
    return paginate($query, $page, $perPage);
}
```

### getPaginatedOrders()

Gets paginated order list.

```php
/**
 * Get paginated order list
 * 
 * @param array $filters Filter options
 * @param int $page Current page
 * @param int $perPage Items per page
 * @return array Paginated result
 */
function getPaginatedOrders(array $filters = [], int $page = 1, int $perPage = 20): array
{
    $query = Capsule::table('tblorders')
        ->select('tblorders.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tblorders.userid')
        ->orderBy('tblorders.id', 'desc');
    
    if (!empty($filters['status'])) {
        $query->where('tblorders.status', $filters['status']);
    }
    
    if (!empty($filters['clientId'])) {
        $query->where('tblorders.userid', $filters['clientId']);
    }
    
    if (!empty($filters['dateFrom'])) {
        $query->where('tblorders.date', '>=', $filters['dateFrom']);
    }
    
    if (!empty($filters['dateTo'])) {
        $query->where('tblorders.date', '<=', $filters['dateTo']);
    }
    
    return paginate($query, $page, $perPage);
}
```

### getPaginatedInvoices()

Gets paginated invoice list.

```php
/**
 * Get paginated invoice list
 * 
 * @param array $filters Filter options
 * @param int $page Current page
 * @param int $perPage Items per page
 * @return array Paginated result
 */
function getPaginatedInvoices(array $filters = [], int $page = 1, int $perPage = 20): array
{
    $query = Capsule::table('tblinvoices')
        ->select('tblinvoices.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tblinvoices.userid')
        ->orderBy('tblinvoices.id', 'desc');
    
    if (!empty($filters['status'])) {
        $query->where('tblinvoices.status', $filters['status']);
    }
    
    if (!empty($filters['clientId'])) {
        $query->where('tblinvoices.userid', $filters['clientId']);
    }
    
    return paginate($query, $page, $perPage);
}
```

### getPaginatedTickets()

Gets paginated ticket list.

```php
/**
 * Get paginated ticket list
 * 
 * @param array $filters Filter options
 * @param int $page Current page
 * @param int $perPage Items per page
 * @return array Paginated result
 */
function getPaginatedTickets(array $filters = [], int $page = 1, int $perPage = 20): array
{
    $query = Capsule::table('tbltickets')
        ->select('tbltickets.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tbltickets.userid')
        ->orderBy('tbltickets.id', 'desc');
    
    if (!empty($filters['status'])) {
        $query->where('tbltickets.status', $filters['status']);
    }
    
    if (!empty($filters['departmentId'])) {
        $query->where('tbltickets.did', $filters['departmentId']);
    }
    
    if (!empty($filters['priority'])) {
        $query->where('tbltickets.priority', $filters['priority']);
    }
    
    return paginate($query, $page, $perPage);
}
```

## Pagination in Custom Queries

```php
/**
 * Custom paginated query
 * 
 * @param string $table Table name
 * @param array $options Query options
 * @return array Paginated result
 */
function customPaginatedQuery(string $table, array $options): array
{
    $page = $options['page'] ?? 1;
    $perPage = $options['per_page'] ?? 20;
    $orderBy = $options['order_by'] ?? 'id';
    $orderDir = $options['order_dir'] ?? 'desc';
    $filters = $options['filters'] ?? [];
    
    $query = Capsule::table($table)->orderBy($orderBy, $orderDir);
    
    // Apply filters
    foreach ($filters as $field => $value) {
        if (is_array($value)) {
            $query->whereIn($field, $value);
        } elseif (strpos($field, '>') !== false) {
            $actualField = str_replace('>', '', $field);
            $query->where($actualField, '>', $value);
        } elseif (strpos($field, '<') !== false) {
            $actualField = str_replace('<', '', $field);
            $query->where($actualField, '<', $value);
        } else {
            $query->where($field, $value);
        }
    }
    
    return paginate($query, $page, $perPage);
}
```

**Example:**
```php
$result = customPaginatedQuery('tblorders', [
    'page' => 2,
    'per_page' => 50,
    'order_by' => 'date',
    'order_dir' => 'desc',
    'filters' => [
        'status' => 'Active',
        'userid' => 123,
        'date>' => '2024-01-01'
    ]
]);

foreach ($result['data'] as $row) {
    // Process row
}
```

## Cursor Pagination

```php
/**
 * Cursor-based pagination
 * 
 * @param string $table Table name
 * @param int $cursor Last seen ID
 * @param int $limit Page size
 * @param array $filters Filters
 * @param string $orderBy Order column
 * @return array Paginated result
 */
function cursorPaginate(
    string $table,
    int $cursor = 0,
    int $limit = 20,
    array $filters = [],
    string $orderBy = 'id'
): array {
    $query = Capsule::table($table)
        ->where($orderBy, '>', $cursor)
        ->orderBy($orderBy, 'asc')
        ->limit($limit + 1); // Fetch one extra to check if more exist
    
    foreach ($filters as $field => $value) {
        $query->where($field, $value);
    }
    
    $items = $query->get()->toArray();
    
    $hasMore = count($items) > $limit;
    if ($hasMore) {
        array_pop($items);
    }
    
    return [
        'data' => $items,
        'next_cursor' => $hasMore && !empty($items) 
            ? $items[count($items) - 1]->$orderBy 
            : null,
        'has_more' => $hasMore
    ];
}
```

**Example:**
```php
$result = cursorPaginate('tblorders', 1000, 50, [
    'status' => 'Active'
]);

// Next page fetch
if ($result['has_more']) {
    $nextPage = cursorPaginate('tblorders', $result['next_cursor'], 50, [
        'status' => 'Active'
    ]);
}
```

## Pagination Information Display

```php
/**
 * Get pagination info text
 * 
 * @param array $pagination Pagination data
 * @return string Info text
 */
function getPaginationInfo(array $pagination): string
{
    if ($pagination['total'] === 0) {
        return 'No results found';
    }
    
    return sprintf(
        'Showing %d to %d of %d results',
        $pagination['from'],
        $pagination['to'],
        $pagination['total']
    );
}

/**
 * Get page size selector
 * 
 * @param array $options Available page sizes
 * @param int $current Current size
 * @param string $baseUrl Base URL
 * @return string HTML select
 */
function getPageSizeSelector(array $options, int $current, string $baseUrl): string
{
    $html = '<select name="per_page" class="page-size-selector" onchange="window.location.href=this.value">';
    
    foreach ($options as $size) {
        $selected = $size === $current ? ' selected' : '';
        $url = buildPaginationUrl($baseUrl, 1, ['per_page' => $size]);
        $html .= '<option value="' . $url . '"' . $selected . '>' . $size . ' per page</option>';
    }
    
    $html .= '</select>';
    
    return $html;
}
```

## JSON API Pagination

```php
/**
 * Format pagination for JSON API
 * 
 * @param array $pagination Pagination data
 * @param string $baseUrl Base URL for next/prev links
 * @return array JSON API pagination
 */
function formatApiPagination(array $pagination, string $baseUrl): array
{
    $current = $pagination['current_page'];
    $last = $pagination['last_page'];
    
    $links = [
        'self' => buildPaginationUrl($baseUrl, $current, []),
        'first' => buildPaginationUrl($baseUrl, 1, []),
        'last' => buildPaginationUrl($baseUrl, $last, [])
    ];
    
    if ($current > 1) {
        $links['prev'] = buildPaginationUrl($baseUrl, $current - 1, []);
    }
    
    if ($current < $last) {
        $links['next'] = buildPaginationUrl($baseUrl, $current + 1, []);
    }
    
    return [
        'total' => $pagination['total'],
        'per_page' => $pagination['per_page'],
        'current_page' => $current,
        'last_page' => $last,
        'from' => $pagination['from'],
        'to' => $pagination['to'],
        'links' => $links
    ];
}
```

## Best Practices

1. **Use consistent page sizes** - Standardize per-page values (10, 25, 50, 100)
2. **Preserve filters in URLs** - Include all filter params in pagination links
3. **Handle empty results** - Show appropriate message when no results
4. **Limit max per page** - Cap at reasonable maximum (100)
5. **Cache counts** - For large tables, consider caching total counts
6. **Use cursor for large datasets** - Better performance for infinite scroll

## Related Functions

- [whmcs-functions-search.md](whmcs-functions-search.md) - Search functionality
- [whmcs-functions-utility.md](whmcs-functions-utility.md) - Utility functions