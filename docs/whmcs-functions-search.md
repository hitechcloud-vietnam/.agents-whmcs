# WHMCS Search Functions

Complete reference for search functionality in WHMCS.

## Overview

WHMCS provides comprehensive search capabilities across clients, orders, invoices, tickets, domains, and services.

## Core Search Functions

### search()

Generic search function.

```php
/**
 * Generic search function
 * 
 * @param string $query Search query
 * @param array $options Search options
 * @return array Search results
 */
function search(string $query, array $options = []): array
{
    $type = $options['type'] ?? 'all';
    $limit = $options['limit'] ?? 20;
    $filters = $options['filters'] ?? [];
    
    switch ($type) {
        case 'clients':
            return searchClients($query, $filters, $limit);
        
        case 'orders':
            return searchOrders($query, $filters, $limit);
        
        case 'invoices':
            return searchInvoices($query, $filters, $limit);
        
        case 'tickets':
            return searchTickets($query, $filters, $limit);
        
        case 'domains':
            return searchDomains($query, $filters, $limit);
        
        case 'services':
            return searchServices($query, $filters, $limit);
        
        case 'all':
        default:
            return globalSearch($query, $limit);
    }
}
```

**Example:**
```php
// Global search
$results = search('john');

// Type-specific search
$clients = search('john', ['type' => 'clients']);
$orders = search('2024', ['type' => 'orders']);
```

### globalSearch()

Searches across all entities.

```php
/**
 * Global search across all entities
 * 
 * @param string $query Search query
 * @param int $limit Per entity limit
 * @return array Results by type
 */
function globalSearch(string $query, int $limit = 5): array
{
    return [
        'clients' => searchClients($query, [], $limit),
        'orders' => searchOrders($query, [], $limit),
        'invoices' => searchInvoices($query, [], $limit),
        'tickets' => searchTickets($query, [], $limit),
        'domains' => searchDomains($query, [], $limit),
        'services' => searchServices($query, [], $limit),
    ];
}
```

**Example:**
```php
$results = globalSearch('acme');

foreach ($results as $type => $items) {
    echo "{$type}: " . count($items) . " found\n";
    foreach ($items as $item) {
        echo "  - " . $item['name'] . "\n";
    }
}
```

## Client Search

### searchClients()

Searches for clients.

```php
/**
 * Search clients
 * 
 * @param string $query Search query
 * @param array $filters Additional filters
 * @param int $limit Result limit
 * @return array Matching clients
 */
function searchClients(string $query, array $filters = [], int $limit = 20): array
{
    $searchFields = ['firstname', 'lastname', 'email', 'companyname', 'phonenumber', 'address1'];
    
    $queryBuilder = Capsule::table('tblclients')
        ->select('tblclients.*')
        ->limit($limit);
    
    // Build search conditions
    $queryBuilder->where(function($q) use ($query, $searchFields) {
        foreach ($searchFields as $field) {
            $q->orWhere($field, 'like', '%' . $query . '%');
        }
    });
    
    // Apply additional filters
    foreach ($filters as $field => $value) {
        $queryBuilder->where($field, $value);
    }
    
    return $queryBuilder->get()->toArray();
}
```

**Example:**
```php
$clients = searchClients('john', [], 50);

// With filters
$clients = searchClients('acme', ['status' => 'Active'], 10);
```

### searchClientsByEmail()

Searches clients by email.

```php
/**
 * Search clients by email
 * 
 * @param string $email Email query
 * @return array Matching clients
 */
function searchClientsByEmail(string $email): array
{
    return Capsule::table('tblclients')
        ->where('email', 'like', '%' . $email . '%')
        ->limit(20)
        ->get()
        ->toArray();
}
```

### searchClientsAdvanced()

Advanced client search with multiple criteria.

```php
/**
 * Advanced client search
 * 
 * @param array $criteria Search criteria
 * @return array Matching clients
 */
function searchClientsAdvanced(array $criteria): array
{
    $query = Capsule::table('tblclients')->select('tblclients.*');
    
    if (!empty($criteria['name'])) {
        $query->where(function($q) use ($criteria) {
            $q->where('firstname', 'like', '%' . $criteria['name'] . '%')
              ->orWhere('lastname', 'like', '%' . $criteria['name'] . '%')
              ->orWhere('companyname', 'like', '%' . $criteria['name'] . '%');
        });
    }
    
    if (!empty($criteria['email'])) {
        $query->where('email', 'like', '%' . $criteria['email'] . '%');
    }
    
    if (!empty($criteria['phone'])) {
        $query->where('phonenumber', 'like', '%' . preg_replace('/[^0-9]/', '', $criteria['phone']) . '%');
    }
    
    if (!empty($criteria['country'])) {
        $query->where('country', $criteria['country']);
    }
    
    if (!empty($criteria['dateFrom'])) {
        $query->where('created_at', '>=', $criteria['dateFrom']);
    }
    
    if (!empty($criteria['dateTo'])) {
        $query->where('created_at', '<=', $criteria['dateTo']);
    }
    
    return $query->limit(100)->get()->toArray();
}
```

## Order Search

### searchOrders()

Searches for orders.

```php
/**
 * Search orders
 * 
 * @param string $query Search query
 * @param array $filters Additional filters
 * @param int $limit Result limit
 * @return array Matching orders
 */
function searchOrders(string $query, array $filters = [], int $limit = 20): array
{
    $searchFields = ['ordernum', 'transfersecret'];
    
    $queryBuilder = Capsule::table('tblorders')
        ->select('tblorders.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tblorders.userid')
        ->limit($limit);
    
    $queryBuilder->where(function($q) use ($query, $searchFields) {
        foreach ($searchFields as $field) {
            $q->orWhere('tblorders.' . $field, 'like', '%' . $query . '%');
        }
        // Also search by client name
        $q->orWhere('tblclients.firstname', 'like', '%' . $query . '%')
          ->orWhere('tblclients.lastname', 'like', '%' . $query . '%');
    });
    
    foreach ($filters as $field => $value) {
        $queryBuilder->where('tblorders.' . $field, $value);
    }
    
    return $queryBuilder->get()->toArray();
}
```

**Example:**
```php
$orders = searchOrders('ORD-2024', ['status' => 'Active']);
```

## Invoice Search

### searchInvoices()

Searches for invoices.

```php
/**
 * Search invoices
 * 
 * @param string $query Search query
 * @param array $filters Additional filters
 * @param int $limit Result limit
 * @return array Matching invoices
 */
function searchInvoices(string $query, array $filters = [], int $limit = 20): array
{
    $searchFields = ['invoicenum'];
    
    $queryBuilder = Capsule::table('tblinvoices')
        ->select('tblinvoices.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tblinvoices.userid')
        ->limit($limit);
    
    $queryBuilder->where(function($q) use ($query, $searchFields) {
        foreach ($searchFields as $field) {
            $q->orWhere('tblinvoices.' . $field, 'like', '%' . $query . '%');
        }
        $q->orWhere('tblinvoices.id', 'like', '%' . $query . '%');
    });
    
    foreach ($filters as $field => $value) {
        $queryBuilder->where('tblinvoices.' . $field, $value);
    }
    
    return $queryBuilder->get()->toArray();
}
```

**Example:**
```php
$invoices = searchInvoices('2024', ['status' => 'Unpaid']);
```

## Ticket Search

### searchTickets()

Searches for tickets.

```php
/**
 * Search tickets
 * 
 * @param string $query Search query
 * @param array $filters Additional filters
 * @param int $limit Result limit
 * @return array Matching tickets
 */
function searchTickets(string $query, array $filters = [], int $limit = 20): array
{
    $queryBuilder = Capsule::table('tbltickets')
        ->select('tbltickets.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tbltickets.userid')
        ->limit($limit);
    
    // Search in ticket ID, subject
    $queryBuilder->where(function($q) use ($query) {
        $q->where('tbltickets.tid', 'like', '%' . $query . '%')
          ->orWhere('tbltickets.subject', 'like', '%' . $query . '%')
          ->orWhere('tbltickets.id', 'like', '%' . $query . '%');
    });
    
    // Search in replies
    $replyMatches = Capsule::table('tblticketreplies')
        ->select('tid')
        ->where('message', 'like', '%' . $query . '%')
        ->limit(100)
        ->get()
        ->pluck('tid')
        ->toArray();
    
    if (!empty($replyMatches)) {
        $queryBuilder->orWhereIn('tbltickets.id', array_unique($replyMatches));
    }
    
    foreach ($filters as $field => $value) {
        $queryBuilder->where('tbltickets.' . $field, $value);
    }
    
    return $queryBuilder->get()->toArray();
}
```

## Domain Search

### searchDomains()

Searches for domains.

```php
/**
 * Search domains
 * 
 * @param string $query Search query
 * @param array $filters Additional filters
 * @param int $limit Result limit
 * @return array Matching domains
 */
function searchDomains(string $query, array $filters = [], int $limit = 20): array
{
    $queryBuilder = Capsule::table('tbldomains')
        ->select('tbldomains.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tbldomains.userid')
        ->limit($limit);
    
    $queryBuilder->where(function($q) use ($query) {
        $q->where('tbldomains.domain', 'like', '%' . $query . '%');
    });
    
    foreach ($filters as $field => $value) {
        $queryBuilder->where('tbldomains.' . $field, $value);
    }
    
    return $queryBuilder->get()->toArray();
}
```

**Example:**
```php
$domains = searchDomains('example', ['domainstatus' => 'Active']);
```

## Service Search

### searchServices()

Searches for services.

```php
/**
 * Search services
 * 
 * @param string $query Search query
 * @param array $filters Additional filters
 * @param int $limit Result limit
 * @return array Matching services
 */
function searchServices(string $query, array $filters = [], int $limit = 20): array
{
    $queryBuilder = Capsule::table('tblhosting')
        ->select('tblhosting.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tblhosting.userid')
        ->limit($limit);
    
    $queryBuilder->where(function($q) use ($query) {
        $q->where('tblhosting.domain', 'like', '%' . $query . '%')
          ->orWhere('tblhosting.username', 'like', '%' . $query . '%');
    });
    
    foreach ($filters as $field => $value) {
        $queryBuilder->where('tblhosting.' . $field, $value);
    }
    
    return $queryBuilder->get()->toArray();
}
```

## Full-Text Search

### fullTextSearch()

Performs full-text search.

```php
/**
 * Full-text search across multiple tables
 * 
 * @param string $query Search query
 * @param array $tables Tables to search
 * @param int $limit Per table limit
 * @return array Results
 */
function fullTextSearch(string $query, array $tables = [], int $limit = 10): array
{
    $tables = $tables ?: ['clients', 'orders', 'invoices', 'tickets', 'domains', 'services'];
    $results = [];
    
    foreach ($tables as $table) {
        switch ($table) {
            case 'clients':
                $results['clients'] = searchClients($query, [], $limit);
                break;
            case 'orders':
                $results['orders'] = searchOrders($query, [], $limit);
                break;
            case 'invoices':
                $results['invoices'] = searchInvoices($query, [], $limit);
                break;
            case 'tickets':
                $results['tickets'] = searchTickets($query, [], $limit);
                break;
            case 'domains':
                $results['domains'] = searchDomains($query, [], $limit);
                break;
            case 'services':
                $results['services'] = searchServices($query, [], $limit);
                break;
        }
    }
    
    return $results;
}
```

## Search Suggestions

### getSearchSuggestions()

Provides search suggestions as user types.

```php
/**
 * Get search suggestions
 * 
 * @param string $query Partial query
 * @param string $type Entity type
 * @return array Suggestions
 */
function getSearchSuggestions(string $query, string $type = 'all'): array
{
    if (strlen($query) < 2) {
        return [];
    }
    
    $suggestions = [];
    
    // Client suggestions
    if ($type === 'all' || $type === 'clients') {
        $clients = Capsule::table('tblclients')
            ->select('id', 'firstname', 'lastname', 'email')
            ->where(function($q) use ($query) {
                $q->where('firstname', 'like', $query . '%')
                  ->orWhere('lastname', 'like', $query . '%')
                  ->orWhere('email', 'like', $query . '%');
            })
            ->limit(5)
            ->get();
        
        foreach ($clients as $client) {
            $suggestions[] = [
                'type' => 'client',
                'text' => "{$client->firstname} {$client->lastname}",
                'subtext' => $client->email,
                'url' => 'clients.php?action=edit&id=' . $client->id
            ];
        }
    }
    
    // Domain suggestions
    if ($type === 'all' || $type === 'domains') {
        $domains = Capsule::table('tbldomains')
            ->select('id', 'domain')
            ->where('domain', 'like', '%' . $query . '%')
            ->limit(5)
            ->get();
        
        foreach ($domains as $domain) {
            $suggestions[] = [
                'type' => 'domain',
                'text' => $domain->domain,
                'url' => 'domains.php?action=edit&id=' . $domain->id
            ];
        }
    }
    
    return $suggestions;
}
```

## Search Indexing

### rebuildSearchIndex()

Rebuilds the search index.

```php
/**
 * Rebuild search index
 * 
 * @return array Results
 */
function rebuildSearchIndex(): array
{
    $indexed = 0;
    
    // Index clients
    $clients = Capsule::table('tblclients')->get();
    foreach ($clients as $client) {
        indexEntity('client', $client->id, [
            'firstname' => $client->firstname,
            'lastname' => $client->lastname,
            'email' => $client->email,
            'companyname' => $client->companyname
        ]);
        $indexed++;
    }
    
    // Index domains
    $domains = Capsule::table('tbldomains')->get();
    foreach ($domains as $domain) {
        indexEntity('domain', $domain->id, [
            'domain' => $domain->domain
        ]);
        $indexed++;
    }
    
    return ['indexed' => $indexed];
}

/**
 * Index an entity
 * 
 * @param string $type Entity type
 * @param int $entityId Entity ID
 * @param array $data Data to index
 * @return bool Success status
 */
function indexEntity(string $type, int $entityId, array $data): bool
{
    $text = implode(' ', array_filter($data));
    
    // Store in search index table
    Capsule::table('tblsearchindex')->updateOrInsert(
        ['entity_type' => $type, 'entity_id' => $entityId],
        [
            'search_text' => strtolower($text),
            'updated_at' => date('Y-m-d H:i:s')
        ]
    );
    
    return true;
}
```

## Best Practices

1. **Use indexes** - Ensure database indexes on searchable fields
2. **Limit results** - Cap results to reasonable limits
3. **Escape input** - Sanitize search queries
4. **Use wildcards wisely** - Leading wildcards can be slow
5. **Cache frequent searches** - Cache popular search results
6. **Provide suggestions** - Help users find what they need

## Related Functions

- [whmcs-functions-pagination.md](whmcs-functions-pagination.md) - Result pagination
- [whmcs-functions-utility.md](whmcs-functions-utility.md) - Utility functions