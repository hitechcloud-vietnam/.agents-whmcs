# WHMCS Performance Optimization Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for optimizing WHMCS module performance.

## When to Use

- Reducing API call frequency
- Optimizing database queries
- Implementing caching

## Optimization Patterns

### API Caching

```php
class CachedApiClient {
    private int $cacheTtl = 300; // 5 minutes

    public function getServerList(): array {
        $cacheKey = 'server_list_' . md5($this->serverId);

        $cached = Capsule::cache()->get($cacheKey);
        if ($cached !== null) {
            return $cached;
        }

        $data = $this->api->getServerList();

        Capsule::cache()->put($cacheKey, $data, $this->cacheTtl);

        return $data;
    }

    public function invalidateCache(): void {
        $cacheKey = 'server_list_' . md5($this->serverId);
        Capsule::cache()->forget($cacheKey);
    }
}
```

### Database Optimization

```php
// Use indexes
Capsule::schema()->table('mod_{module}_data', function($t) {
    $t->index(['user_id', 'type', 'created_at'], 'idx_user_type_date');
});

// Batch operations
function batchImport(array $items): void {
    $chunks = array_chunk($items, 100);

    foreach ($chunks as $chunk) {
        Capsule::table('mod_{module}_data')->insert($chunk);
    }
}

// Efficient queries
$results = Capsule::table('mod_{module}_data')
    ->select(['id', 'user_id', 'type', 'created_at'])  // Select only needed columns
    ->where('user_id', $userId)
    ->where('status', 'active')
    ->orderBy('created_at', 'desc')
    ->limit(50)
    ->get();
```

### Lazy Loading

```php
class LazyLoader {
    private ?array $serverInfo = null;

    public function getServerInfo(): array {
        if ($this->serverInfo === null) {
            $api = new ApiClient($this->params);
            $this->serverInfo = $api->getServerInfo($this->serverId);
        }
        return $this->serverInfo;
    }
}
```

### Async Processing

```php
function queueAsyncTask(string $task, array $data): void {
    Capsule::table('mod_{module}_queue')->insert([
        'task' => $task,
        'data' => json_encode($data),
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

function processQueue(): void {
    $tasks = Capsule::table('mod_{module}_queue')
        ->where('status', 'pending')
        ->limit(10)
        ->get();

    foreach ($tasks as $task) {
        try {
            $data = json_decode($task->data, true);
            processTask($task->task, $data);

            Capsule::table('mod_{module}_queue')
                ->where('id', $task->id)
                ->update(['status' => 'completed', 'processed_at' => date('Y-m-d H:i:s')]);
        } catch (\Exception $e) {
            Capsule::table('mod_{module}_queue')
                ->where('id', $task->id)
                ->update(['status' => 'failed', 'error' => $e->getMessage()]);
        }
    }
}
```

## Checklist

- [ ] API response caching
- [ ] Database indexes
- [ ] Batch operations for bulk data
- [ ] Lazy loading for heavy resources
- [ ] Async queue for non-critical tasks

---

**Related Skills:**
- whmcs-database-design
- whmcs-api-integration
- whmcs-cron-automation