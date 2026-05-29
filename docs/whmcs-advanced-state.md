# WHMCS State Management

Complete guide to state management patterns.

## Overview

Manage application state effectively.

## State Machine

```php
<?php
/**
 * State machine for service status
 */
class ServiceStateMachine
{
    private array $states = [
        'pending' => ['active', 'cancelled'],
        'active' => ['suspended', 'terminated'],
        'suspended' => ['active', 'terminated'],
        'terminated' => [],
        'cancelled' => [],
    ];
    
    /**
     * Check if transition is valid
     */
    public function canTransition(string $from, string $to): bool
    {
        return in_array($to, $this->states[$from] ?? []);
    }
    
    /**
     * Get available transitions
     */
    public function getAvailableTransitions(string $from): array
    {
        return $this->states[$from] ?? [];
    }
    
    /**
     * Transition state
     */
    public function transition(int $serviceId, string $to): array
    {
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();
        
        $from = $service->domainstatus;
        
        if (!$this->canTransition($from, $to)) {
            return [
                'success' => false,
                'error' => "Cannot transition from {$from} to {$to}",
            ];
        }
        
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update(['domainstatus' => $to]);
        
        return [
            'success' => true,
            'from' => $from,
            'to' => $to,
        ];
    }
}
```

## Best Practices

1. **Define states** - Clear state definitions
2. **Validate transitions** - Only allow valid transitions
3. **Log changes** - Track state history
4. **Atomic updates** - Ensure consistency
5. **Side effects** - Handle state change effects
6. **Testing** - Test all transitions

## Related Documentation

- [whmcs-advanced-hooks.md](whmcs-advanced-hooks.md)
- [whmcs-advanced-database.md](whmcs-advanced-database.md)
