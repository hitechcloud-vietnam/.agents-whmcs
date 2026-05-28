# WHMCS Search Optimization Workflow

## Purpose

Implement comprehensive search optimization for WHMCS to improve performance, reduce database load, and enhance user experience when searching for clients, products, orders, and invoices. This workflow covers database optimization, indexing, caching, and alternative search engines.

## Prerequisites

- WHMCS v8.0+
- MySQL 8.0+ or MariaDB 10.5+
- SSH access to server
- Admin access to WHMCS

## Workflow Steps

### Step 1: Current Search Assessment

```sql
-- Analyze current search patterns
SELECT 
    action,
    COUNT(*) as usage_count,
    AVG(execution_time) as avg_time,
    MAX(execution_time) as max_time
FROM tblactivitylog
WHERE date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
AND action LIKE '%search%'
GROUP BY action
ORDER BY usage_count DESC;

-- Find slow searches in process list
SHOW PROCESSLIST;

-- Check for full table scans
EXPLAIN SELECT * FROM tblclients 
WHERE firstname LIKE '%john%' OR lastname LIKE '%john%' OR email LIKE '%john%';
```

### Step 2: Database Indexing for Search

```sql
-- Full-text indexes for client search
ALTER TABLE tblclients 
ADD FULLTEXT INDEX ft_client_search (firstname, lastname, email, companyname);

-- Full-text indexes for product search
ALTER TABLE tblproducts 
ADD FULLTEXT INDEX ft_product_search (name, description, tag);

-- Full-text indexes for domain search
ALTER TABLE tbldomains 
ADD FULLTEXT INDEX ft_domain_search (domain);

-- Full-text indexes for order search
ALTER TABLE tblorders 
ADD FULLTEXT INDEX ft_order_search (ordernum, transid);
```

```php
<?php
// /var/www/html/whmcs/includes/helpers/OptimizedSearch.php

namespace WHMCS\Helpers;

use Illuminate\Database\Capsule\Manager as Capsule;

class OptimizedSearch
{
    /**
     * Full-text client search
     */
    public static function searchClients(string $query, int $limit = 50): array
    {
        $query = trim($query);
        
        if (strlen($query) < 2) {
            return [];
        }

        // Use FULLTEXT search for better performance
        $results = Capsule::select("
            SELECT c.*, 
                   MATCH(firstname, lastname, email, companyname) 
                   AGAINST(? IN NATURAL LANGUAGE MODE) as relevance
            FROM tblclients c
            WHERE MATCH(firstname, lastname, email, companyname) 
                  AGAINST(? IN NATURAL LANGUAGE MODE)
            ORDER BY relevance DESC
            LIMIT ?
        ", [$query, $query, $limit]);

        return $results;
    }

    /**
     * Fallback LIKE search with optimized pattern
     */
    public static function searchClientsFallback(string $query, int $limit = 50): array
    {
        $search = '%' . $this->escapeLike($query) . '%';
        
        return Capsule::table('tblclients')
            ->select(['id', 'firstname', 'lastname', 'email', 'companyname', 'status'])
            ->where(function($q) use ($search) {
                $q->where('firstname', 'LIKE', $search)
                  ->orWhere('lastname', 'LIKE', $search)
                  ->orWhere('email', 'LIKE', $search)
                  ->orWhere('companyname', 'LIKE', $search);
            })
            ->limit($limit)
            ->get()
            ->toArray();
    }

    /**
     * Search products with ranking
     */
    public static function searchProducts(string $query, int $limit = 50): array
    {
        $results = Capsule::select("
            SELECT p.*, 
                   MATCH(name, description) 
                   AGAINST(? IN BOOLEAN MODE) as relevance
            FROM tblproducts p
            WHERE MATCH(name, description) AGAINST(? IN BOOLEAN MODE)
            AND hidden = '0'
            ORDER BY relevance DESC
            LIMIT ?
        ", [$query . '*', $query . '*', $limit]);

        return $results;
    }

    /**
     * Search domains
     */
    public static function searchDomains(string $query, int $limit = 50): array
    {
        return Capsule::table('tbldomains')
            ->select(['id', 'domain', 'userid', 'status', 'expirydate'])
            ->where('domain', 'LIKE', '%' . $this->escapeLike($query) . '%')
            ->limit($limit)
            ->get()
            ->toArray();
    }

    /**
     * Universal search across entities
     */
    public static function universalSearch(string $query): array
    {
        $cache = \DI::make('cache');
        $cacheKey = 'search:' . md5($query);
        
        return $cache->remember($cacheKey, 300, function() use ($query) {
            return [
                'clients' => self::searchClients($query, 10),
                'products' => self::searchProducts($query, 10),
                'domains' => self::searchDomains($query, 10),
                'orders' => self::searchOrders($query, 10),
            ];
        });
    }

    /**
     * Search orders
     */
    public static function searchOrders(string $query, int $limit = 50): array
    {
        return Capsule::table('tblorders')
            ->select(['id', 'ordernum', 'userid', 'date', 'total', 'status'])
            ->where('ordernum', 'LIKE', '%' . $this->escapeLike($query) . '%')
            ->orWhere('transid', 'LIKE', '%' . $this->escapeLike($query) . '%')
            ->limit($limit)
            ->get()
            ->toArray();
    }

    /**
     * Escape LIKE special characters
     */
    private static function escapeLike(string $value): string
    {
        return str_replace(['%', '_', '\\'], ['\\%', '\\_', '\\\\'], $value);
    }
}
```

### Step 3: Elasticsearch Integration

```php
<?php
// /var/www/html/whmcs/includes/SearchElasticsearch.php
namespace WHMCS\Search;

class ElasticsearchEngine
{
    private $client;
    private $indexPrefix = 'whmcs_';
    
    public function __construct(array $config)
    {
        $this->client = \Elastic\Elasticsearch\ClientBuilder::create()
            ->setHosts([$config['host'] . ':' . $config['port']])
            ->setBasicAuthentication($config['user'], $config['password'])
            ->build();
    }

    /**
     * Create WHMCS index with mappings
     */
    public function createIndex(): bool
    {
        $params = [
            'index' => $this->indexPrefix . 'clients',
            'body' => [
                'settings' => [
                    'number_of_shards' => 1,
                    'number_of_replicas' => 0,
                    'analysis' => [
                        'analyzer' => [
                            'whmcs_analyzer' => [
                                'type' => 'custom',
                                'tokenizer' => 'standard',
                                'filter' => ['lowercase', 'asciifolding']
                            ]
                        ]
                    ]
                ],
                'mappings' => [
                    'properties' => [
                        'id' => ['type' => 'integer'],
                        'firstname' => ['type' => 'text', 'analyzer' => 'whmcs_analyzer'],
                        'lastname' => ['type' => 'text', 'analyzer' => 'whmcs_analyzer'],
                        'fullname' => ['type' => 'text', 'analyzer' => 'whmcs_analyzer'],
                        'email' => ['type' => 'keyword', 'normalizer' => 'lowercase'],
                        'companyname' => ['type' => 'text'],
                        'status' => ['type' => 'keyword'],
                        'created_at' => ['type' => 'date']
                    ]
                ]
            ]
        ];

        try {
            $this->client->indices()->create($params);
            return true;
        } catch (\Exception $e) {
            return false;
        }
    }

    /**
     * Index a client document
     */
    public function indexClient(array $client): bool
    {
        $params = [
            'index' => $this->indexPrefix . 'clients',
            'id' => $client['id'],
            'body' => [
                'id' => $client['id'],
                'firstname' => $client['firstname'],
                'lastname' => $client['lastname'],
                'fullname' => $client['firstname'] . ' ' . $client['lastname'],
                'email' => strtolower($client['email']),
                'companyname' => $client['companyname'] ?? '',
                'status' => $client['status'],
                'created_at' => $client['createdat']
            ]
        ];

        try {
            $this->client->index($params);
            return true;
        } catch (\Exception $e) {
            logActivity('Elasticsearch index error: ' . $e->getMessage());
            return false;
        }
    }

    /**
     * Search clients
     */
    public function searchClients(string $query, int $limit = 50): array
    {
        $params = [
            'index' => $this->indexPrefix . 'clients',
            'body' => [
                'size' => $limit,
                'query' => [
                    'multi_match' => [
                        'query' => $query,
                        'fields' => ['fullname^3', 'email^2', 'firstname', 'lastname', 'companyname'],
                        'type' => 'best_fields',
                        'fuzziness' => 'AUTO'
                    ]
                ]
            ]
        ];

        try {
            $response = $this->client->search($params);
            $hits = $response['hits']['hits'] ?? [];
            
            return array_map(function($hit) {
                return $hit['_source'];
            }, $hits);
        } catch (\Exception $e) {
            return [];
        }
    }

    /**
     * Bulk index clients
     */
    public function bulkIndexClients(array $clients): int
    {
        $params = ['body' => []];
        
        foreach ($clients as $client) {
            $params['body'][] = [
                'index' => [
                    '_index' => $this->indexPrefix . 'clients',
                    '_id' => $client['id']
                ]
            ];
            $params['body'][] = [
                'id' => $client['id'],
                'firstname' => $client['firstname'],
                'lastname' => $client['lastname'],
                'fullname' => $client['firstname'] . ' ' . $client['lastname'],
                'email' => strtolower($client['email']),
                'companyname' => $client['companyname'] ?? '',
                'status' => $client['status']
            ];
        }

        $response = $this->client->bulk($params);
        return count($response['items']);
    }
}
```

### Step 4: Search Caching Implementation

```php
<?php
// /var/www/html/whmcs/includes/hooks/search_cache_hook.php

add_hook('ClientAdd', 1, function($vars) {
    $cache = \DI::make('cache');
    $cache->delete('search_index:*'); // Pattern delete if supported
});

add_hook('ClientEdit', 1, function($vars) {
    if (class_exists('\WHMCS\Search\ElasticsearchEngine')) {
        $search = new \WHMCS\Search\ElasticsearchEngine([
            'host' => '127.0.0.1',
            'port' => 9200,
            'user' => 'elastic',
            'password' => 'password'
        ]);
        $search->indexClient(\WHMCS\User\Client::find($vars['userid']));
    }
});

/**
 * Pre-warm search cache
 */
add_hook('DailyCronJob', 1, function() {
    logActivity('Pre-warming search cache');
    
    $cache = \DI::make('cache');
    
    // Warm common searches
    $commonQueries = ['pending', 'active', 'suspended', 'unpaid'];
    foreach ($commonQueries as $query) {
        \WHMCS\Helpers\OptimizedSearch::searchClients($query, 10);
    }
    
    logActivity('Search cache warm-up complete');
});
```

### Step 5: Admin Search Optimization

```php
<?php
// /var/www/html/whmcs/includes/helpers/AdminSearch.php
namespace WHMCS\Helpers;

class AdminSearch
{
    /**
     * Optimized admin client search with filters
     */
    public static function adminClientSearch(array $filters): array
    {
        $cache = \DI::make('cache');
        $cacheKey = 'admin:clients:' . md5(json_encode($filters));
        
        return $cache->remember($cacheKey, 120, function() use ($filters) {
            $query = Capsule::table('tblclients')
                ->select([
                    'id', 'firstname', 'lastname', 'email', 
                    'companyname', 'status', 'groupid', 'createdat'
                ]);

            // Apply filters
            if (!empty($filters['status'])) {
                $query->where('status', $filters['status']);
            }
            
            if (!empty($filters['group'])) {
                $query->where('groupid', $filters['group']);
            }
            
            if (!empty($filters['search'])) {
                $search = '%' . self::escapeLike($filters['search']) . '%';
                $query->where(function($q) use ($search) {
                    $q->where('firstname', 'LIKE', $search)
                      ->orWhere('lastname', 'LIKE', $search)
                      ->orWhere('email', 'LIKE', $search)
                      ->orWhere('companyname', 'LIKE', $search);
                });
            }

            // Pagination
            $page = $filters['page'] ?? 1;
            $perPage = $filters['per_page'] ?? 50;
            
            $total = $query->count();
            $results = $query
                ->orderBy('id', 'desc')
                ->offset(($page - 1) * $perPage)
                ->limit($perPage)
                ->get();

            return [
                'data' => $results,
                'total' => $total,
                'page' => $page,
                'per_page' => $perPage
            ];
        });
    }

    /**
     * Search with recent search history
     */
    public static function getRecentSearches(int $userId, int $limit = 10): array
    {
        return Capsule::table('tblactivitylog')
            ->select('description')
            ->where('userid', $userId)
            ->where('description', 'LIKE', '%Search:%')
            ->orderBy('date', 'desc')
            ->limit($limit)
            ->pluck('description')
            ->toArray();
    }
}
```

### Step 6: Search Performance Monitoring

```sql
-- Create search performance table
CREATE TABLE IF NOT EXISTS tbl_search_stats (
    id INT AUTO_INCREMENT PRIMARY KEY,
    search_type VARCHAR(50),
    query_text VARCHAR(255),
    execution_time_ms INT,
    results_count INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_search_type (search_type),
    INDEX idx_execution_time (execution_time_ms)
);

-- Log slow searches
SELECT 
    query_text,
    COUNT(*) as search_count,
    AVG(execution_time_ms) as avg_time,
    MAX(execution_time_ms) as max_time
FROM tbl_search_stats
WHERE created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY query_text
ORDER BY avg_time DESC
LIMIT 20;
```

```php
<?php
// /var/www/html/whmcs/includes/helpers/SearchMonitor.php

class SearchMonitor
{
    public static function logSearch(string $type, string $query, float $time, int $results): void
    {
        Capsule::table('tbl_search_stats')->insert([
            'search_type' => $type,
            'query_text' => substr($query, 0, 255),
            'execution_time_ms' => (int) ($time * 1000),
            'results_count' => $results
        ]);
    }

    public static function getSlowSearches(int $thresholdMs = 1000): array
    {
        return Capsule::table('tbl_search_stats')
            ->selectRaw('search_type, COUNT(*) as count, AVG(execution_time_ms) as avg_time')
            ->where('execution_time_ms', '>', $thresholdMs)
            ->groupBy('search_type')
            ->get();
    }
}
```

## Search Optimization Summary

| Technique | Performance Gain | Complexity |
|-----------|-----------------|------------|
| Database Indexing | 3-10x | Low |
| FULLTEXT Search | 5-20x | Low |
| Query Caching | 10-50x | Medium |
| Elasticsearch | 10-100x | High |
| CDN Edge Caching | 50-100x | Medium |

## Best Practices

1. **Index Appropriately**: Create indexes on frequently searched columns
2. **Use Full-Text Search**: For text searches, use MySQL FULLTEXT
3. **Cache Results**: Cache common searches for faster results
4. **Limit Results**: Always use LIMIT to prevent large result sets
5. **Escape Input**: Sanitize search queries to prevent SQL injection
6. **Monitor Performance**: Track slow searches and optimize them
7. **Consider Elasticsearch**: For large datasets, consider full-text search engines

## Common Pitfalls

- **Leading Wildcards**: Using LIKE '%term%' causes full table scans
- **No Pagination**: Loading entire result sets
- **Unindexed Columns**: Searching columns without indexes
- **Case Sensitivity**: Not handling case variations in searches
- **Special Characters**: Not escaping special characters in queries

## Verification Checklist

- [ ] Full-text indexes created
- [ ] Search performance improved
- [ ] Caching implemented
- [ ] Slow search monitoring in place
- [ ] Admin search optimized
- [ ] Monitoring dashboard created

## Related Documentation

- [WHMCS Database Tuning](whmcs-database-tuning.md)
- [WHMCS Cache Optimization](whmcs-cache-optimization.md)
- [WHMCS Performance Monitoring](whmcs-performance-monitoring.md)