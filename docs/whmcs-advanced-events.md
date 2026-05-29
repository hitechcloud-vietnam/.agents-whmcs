# WHMCS Event System

Complete guide to event-driven architecture.

## Overview

Implement event-driven patterns for WHMCS.

## Event Emitter

```php
<?php
/**
 * Event emitter
 */
class EventEmitter
{
    private array $listeners = [];
    
    /**
     * Register listener
     */
    public function on(string $event, callable $listener): void
    {
        $this->listeners[$event][] = $listener;
    }
    
    /**
     * Emit event
     */
    public function emit(string $event, array $data = []): void
    {
        if (!isset($this->listeners[$event])) {
            return;
        }
        
        foreach ($this->listeners[$event] as $listener) {
            $listener($data);
        }
    }
    
    /**
     * Remove listener
     */
    public function off(string $event, callable $listener): void
    {
        if (!isset($this->listeners[$event])) {
            return;
        }
        
        $this->listeners[$event] = array_filter(
            $this->listeners[$event],
            fn($l) => $l !== $listener
        );
    }
}
```

## Best Practices

1. **Single responsibility** - Events do one thing
2. **Async processing** - Don't block main flow
3. **Error handling** - Handle listener failures
4. **Documentation** - Document events clearly
5. **Versioning** - Version event schemas
6. **Testing** - Test event flows

## Related Documentation

- [whmcs-advanced-hooks.md](whmcs-advanced-hooks.md)
- [whmcs-advanced-automation.md](whmcs-advanced-automation.md)
