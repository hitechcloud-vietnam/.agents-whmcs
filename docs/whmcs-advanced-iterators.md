# WHMCS Iterators

Complete guide to iterator patterns.

## Overview

Implement custom iterators for data processing.

## Iterator Examples

```php
<?php
/**
 * Chunked iterator for large datasets
 */
class ChunkedIterator implements Iterator
{
    private $query;
    private int $chunkSize;
    private int $offset = 0;
    private array $currentChunk = [];
    private int $currentIndex = 0;
    
    public function __construct($query, int $chunkSize = 100)
    {
        $this->query = $query;
        $this->chunkSize = $chunkSize;
    }
    
    public function rewind(): void
    {
        $this->offset = 0;
        $this->loadChunk();
    }
    
    public function current(): mixed
    {
        return $this->currentChunk[$this->currentIndex] ?? null;
    }
    
    public function key(): int
    {
        return $this->offset + $this->currentIndex;
    }
    
    public function next(): void
    {
        $this->currentIndex++;
        
        if ($this->currentIndex >= count($this->currentChunk)) {
            $this->offset += $this->chunkSize;
            $this->loadChunk();
        }
    }
    
    public function valid(): bool
    {
        return !empty($this->currentChunk);
    }
    
    private function loadChunk(): void
    {
        $this->currentChunk = $this->query
            ->offset($this->offset)
            ->limit($this->chunkSize)
            ->get()
            ->toArray();
        
        $this->currentIndex = 0;
    }
}

/**
 * Usage
 */
$iterator = new ChunkedIterator(
    Capsule::table('tblclients')->where('status', 'Active')
);

foreach ($iterator as $client) {
    // Process client
}
```

## Best Practices

1. **Memory efficiency** - Don't load all data
2. **Lazy loading** - Load on demand
3. **Clear interface** - Implement Iterator methods
4. **Error handling** - Handle iteration errors
5. **Reset support** - Implement rewind properly
6. **Multiple passes** - Support multiple iterations

## Related Documentation

- [whmcs-advanced-database.md](whmcs-advanced-database.md)
- [whmcs-advanced-performance.md](whmcs-advanced-performance.md)
