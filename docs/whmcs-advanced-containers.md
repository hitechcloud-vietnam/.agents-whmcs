# WHMCS Dependency Injection

Complete guide to DI patterns.

## Overview

Implement dependency injection for WHMCS.

## Service Container

```php
<?php
/**
 * Service container
 */
class Container
{
    private array $services = [];
    private array $instances = [];
    
    /**
     * Register service
     */
    public function register(string $id, callable $factory): void
    {
        $this->services[$id] = $factory;
    }
    
    /**
     * Get service
     */
    public function get(string $id): mixed
    {
        if (isset($this->instances[$id])) {
            return $this->instances[$id];
        }
        
        if (!isset($this->services[$id])) {
            throw new Exception("Service not found: {$id}");
        }
        
        $instance = $this->services[$id]($this);
        $this->instances[$id] = $instance;
        
        return $instance;
    }
    
    /**
     * Check if service exists
     */
    public function has(string $id): bool
    {
        return isset($this->services[$id]);
    }
}
```

## Best Practices

1. **Single responsibility** - One service per class
2. **Interface binding** - Depend on abstractions
3. **Lazy loading** - Only instantiate when needed
4. **Clear naming** - Use descriptive IDs
5. **Testing** - Easy to mock dependencies
6. **Documentation** - Document service dependencies

## Related Documentation

- [whmcs-advanced-api.md](whmcs-advanced-api.md)
- [whmcs-advanced-testing.md](whmcs-advanced-testing.md)
