# WHMCS API Pagination

## Overview

When querying lists with the WHMCS API, results are paginated to manage response sizes and server load.

## Pagination Parameters

### Request Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limitstart` | integer | 0 | Starting offset |
| `limitnum` | integer | 25 | Number of results per page |
| `page` | integer | 1 | Page number (alternative to limitstart) |

### Response Structure

```json
{
    "result": "success",
    "totalresults": 150,
    "startnumber": 0,
    "numreturned": 25,
    "page": 1,
    "pages": 6,
    "clients": [...]
}
```

## Implementation

### Pagination Helper Class

```php
<?php
class WhmcsPaginationHelper {
    private int $totalResults;
    private int $limitNum;
    private int $startNumber;
    private int $numReturned;
    private int $page;
    private int $pages;
    
    public function __construct(array $response)
    {
        $this->totalResults = (int) ($response['totalresults'] ?? 0);
        $this->limitNum = (int) ($response['limitnum'] ?? 25);
        $this->startNumber = (int) ($response['startnumber'] ?? 0);
        $this->numReturned = (int) ($response['numreturned'] ?? 0);
        $this->pages = (int) ceil($this->totalResults / $this->limitNum);
        $this->page = (int) floor($this->startNumber / $this->limitNum) + 1;
    }
    
    public function hasNextPage(): bool
    {
        return $this->page < $this->pages;
    }
    
    public function hasPreviousPage(): bool
    {
        return $this->page > 1;
    }
    
    public function getNextOffset(): int
    {
        return $this->startNumber + $this->numReturned;
    }
    
    public function getPreviousOffset(): int
    {
        return max(0, $this->startNumber - $this->limitNum);
    }
    
    public function getOffsetForPage(int $page): int
    {
        return ($page - 1) * $this->limitNum;
    }
    
    public function getMetadata(): array
    {
        return [
            'page' => $this->page,
            'pages' => $this->pages,
            'total_results' => $this->totalResults,
            'per_page' => $this->limitNum,
            'has_next' => $this->hasNextPage(),
            'has_previous' => $this->hasPreviousPage(),
        ];
    }
}
```

### Fetch All Results

```php
<?php
class WhmcsBulkFetcher {
    private WhmcsApiClient $api;
    private int $defaultLimit = 100;
    private int $maxLimit = 1000;
    
    public function __construct(WhmcsApiClient $api)
    {
        $this->api = $api;
    }
    
    public function fetchAll(string $action, array $params = []): array
    {
        $allResults = [];
        $offset = 0;
        
        do {
            $params['limitstart'] = $offset;
            $params['limitnum'] = $this->defaultLimit;
            
            $response = $this->api->makeRequest(
                array_merge(['action' => $action], $params)
            );
            
            $pagination = new WhmcsPaginationHelper($response);
            
            $dataKey = $this->getDataKey($action);
            if (isset($response[$dataKey])) {
                $allResults = array_merge($allResults, $response[$dataKey]);
            }
            
            $offset = $pagination->getNextOffset();
            
        } while ($pagination->hasNextPage());
        
        return $allResults;
    }
    
    public function fetchAllParallel(string $action, array $params = [], int $maxConcurrent = 5): array
    {
        // Get total count first
        $initialResponse = $this->api->makeRequest([
            'action' => $action,
            'limitstart' => 0,
            'limitnum' => 1,
        ] + $params);
        
        $pagination = new WhmcsPaginationHelper($initialResponse);
        $totalPages = $pagination->getMetadata()['pages'];
        
        // Fetch all pages concurrently
        $chunks = array_chunk(
            range(0, $totalPages - 1),
            $maxConcurrent
        );
        
        $allResults = [];
        
        foreach ($chunks as $pageChunk) {
            $handles = [];
            
            // Start all requests in chunk
            foreach ($pageChunk as $pageNum) {
                $offset = $pageNum * $this->defaultLimit;
                $handles[$pageNum] = $this->startRequest($action, $params, $offset);
            }
            
            // Collect results
            foreach ($handles as $pageNum => $handle) {
                curl_close($handle);
                // Parse response (simplified)
            }
        }
        
        return $allResults;
    }
    
    private function startRequest(string $action, array $params, int $offset)
    {
        $params['limitstart'] = $offset;
        $params['limitnum'] = $this->defaultLimit;
        
        $ch = curl_init($this->api->getEndpoint());
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query(array_merge(['action' => $action], $params)),
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        curl_exec($ch);
        return $ch;
    }
    
    private function getDataKey(string $action): string
    {
        $keyMap = [
            'GetClients' => 'clients',
            'GetInvoices' => 'invoices',
            'GetOrders' => 'orders',
            'GetServices' => 'services',
            'GetTickets' => 'tickets',
            'GetDomains' => 'domains',
        ];
        
        return $keyMap[$action] ?? strtolower(substr($action, 3)) . 's';
    }
}
```

### Cursor-Based Pagination

```php
<?php
class WhmcsCursorPaginator {
    private WhmcsApiClient $api;
    private int $limit;
    private ?string $lastId = null;
    private bool $hasMore = true;
    
    public function __construct(WhmcsApiClient $api, int $limit = 100)
    {
        $this->api = $api;
        $this->limit = min($limit, 1000); // Enforce max limit
    }
    
    public function fetchPage(): array
    {
        if (!$this->hasMore) {
            return [];
        }
        
        $params = [
            'limitnum' => $this->limit,
            'sorting' => 'asc',
        ];
        
        if ($this->lastId !== null) {
            $params['after_id'] = $this->lastId;
        }
        
        $response = $this->api->makeRequest([
            'action' => 'GetClients',
        ] + $params);
        
        $clients = $response['clients'] ?? [];
        
        if (count($clients) < $this->limit) {
            $this->hasMore = false;
        } else {
            $this->lastId = end($clients)['id'] ?? null;
        }
        
        return $clients;
    }
    
    public function fetchAll(): Generator
    {
        while ($this->hasMore) {
            $page = $this->fetchPage();
            
            foreach ($page as $item) {
                yield $item;
            }
            
            if (empty($page)) {
                break;
            }
        }
    }
    
    public function hasMoreResults(): bool
    {
        return $this->hasMore;
    }
}
```

### Iterator Implementation

```php
<?php
class WhmcsClientIterator implements Iterator {
    private WhmcsApiClient $api;
    private int $limit;
    private int $offset = 0;
    private array $currentPage = [];
    private int $currentIndex = 0;
    private int $totalResults = PHP_INT_MAX;
    
    public function __construct(WhmcsApiClient $api, int $limit = 100)
    {
        $this->api = $api;
        $this->limit = $limit;
    }
    
    public function current(): mixed
    {
        return $this->currentPage[$this->currentIndex] ?? null;
    }
    
    public function key(): int
    {
        return $this->offset + $this->currentIndex;
    }
    
    public function next(): void
    {
        $this->currentIndex++;
        
        if (!isset($this->currentPage[$this->currentIndex])) {
            $this->fetchNextPage();
        }
    }
    
    public function rewind(): void
    {
        $this->offset = 0;
        $this->currentIndex = 0;
        $this->fetchNextPage();
    }
    
    public function valid(): bool
    {
        return isset($this->currentPage[$this->currentIndex]);
    }
    
    private function fetchNextPage(): void
    {
        if ($this->offset >= $this->totalResults) {
            $this->currentPage = [];
            return;
        }
        
        $response = $this->api->makeRequest([
            'action' => 'GetClients',
            'limitstart' => $this->offset,
            'limitnum' => $this->limit,
        ]);
        
        $this->currentPage = $response['clients'] ?? [];
        $this->totalResults = (int) ($response['totalresults'] ?? 0);
        $this->currentIndex = 0;
        $this->offset += count($this->currentPage);
    }
}

// Usage
foreach (new WhmcsClientIterator($api, 200) as $client) {
    processClient($client);
}
```

## Best Practices

1. **Use reasonable limits** - Don't set limits exceeding 1000
2. **Implement caching** - Cache paginated results when possible
3. **Use generators** - Memory-efficient for large datasets
4. **Handle empty responses** - Check for missing data keys
5. **Track total results** - Monitor totalresults to track progress

## Related Documentation

- [WHMCS API Sorting & Filtering](/docs/whmcs-api-sorting-filtering.md)
- [WHMCS API Batch Operations](/docs/whmcs-api-batch-operations.md)