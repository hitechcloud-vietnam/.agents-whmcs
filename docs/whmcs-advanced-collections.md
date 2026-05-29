# WHMCS Collections

Complete guide to collection patterns.

## Overview

Build utility collections for data manipulation.

## Collection Class

```php
<?php
/**
 * Collection helper
 */
class Collection
{
    private array $items = [];
    
    public function __construct(array $items = [])
    {
        $this->items = $items;
    }
    
    /**
     * Filter items
     */
    public function filter(callable $callback): self
    {
        return new self(array_filter($this->items, $callback));
    }
    
    /**
     * Map items
     */
    public function map(callable $callback): self
    {
        return new self(array_map($callback, $this->items));
    }
    
    /**
     * Get first item
     */
    public function first(): mixed
    {
        return $this->items[0] ?? null;
    }
    
    /**
     * Get last item
     */
    public function last(): mixed
    {
        return end($this->items) ?: null;
    }
    
    /**
     * Get count
     */
    public function count(): int
    {
        return count($this->items);
    }
    
    /**
     * Group by key
     */
    public function groupBy(string $key): array
    {
        $groups = [];
        
        foreach ($this->items as $item) {
            $value = $item[$key] ?? null;
            $groups[$value][] = $item;
        }
        
        return $groups;
    }
    
    /**
     * Sort by key
     */
    public function sortBy(string $key, string $direction = 'asc'): self
    {
        usort($this->items, function($a, $b) use ($key, $direction) {
            $cmp = ($a[$key] ?? '') <=> ($b[$key] ?? '');
            return $direction === 'desc' ? -$cmp : $cmp;
        });
        
        return $this;
    }
}
```

## Best Practices

1. **Immutability** - Return new collections
2. **Chainable** - Enable method chaining
3. **Type safety** - Validate item types
4. **Performance** - Use native functions
5. **Memory** - Handle large collections
6. **Testing** - Test collection operations

## Related Documentation

- [whmcs-advanced-database.md](whmcs-advanced-database.md)
- [whmcs-advanced-iterators.md](whmcs-advanced-iterators.md)
