# WHMCS Factories

Complete guide to factory patterns.

## Overview

Implement factory patterns for object creation.

## Factory Pattern

```php
<?php
/**
 * Service factory
 */
class ServiceFactory
{
    /**
     * Create service instance
     */
    public function create(array $productData): array
    {
        $service = [
            'id' => Capsule::table('tblhosting')->insertGetId([
                'userid' => $productData['userid'],
                'packageid' => $productData['packageid'],
                'domain' => $productData['domain'],
                'regdate' => date('Y-m-d'),
                'domainstatus' => 'Pending',
            ]),
            'domain' => $productData['domain'],
            'status' => 'Pending',
        ];
        
        // Create invoice
        $invoiceId = (new InvoiceFactory())->createForService($service);
        
        return $service;
    }
}

/**
 * Invoice factory
 */
class InvoiceFactory
{
    public function createForService(array $service): int
    {
        return Capsule::table('tblinvoices')->insertGetId([
            'userid' => $service['userid'] ?? 0,
            'date' => date('Y-m-d'),
            'duedate' => date('Y-m-d', strtotime('+7 days')),
            'status' => 'Unpaid',
            'total' => 0,
        ]);
    }
}
```

## Best Practices

1. **Single responsibility** - Each factory creates one type
2. **Configuration** - Accept configuration arrays
3. **Validation** - Validate before creation
4. **Dependencies** - Inject dependencies
5. **Testing** - Easy to mock factories
6. **Documentation** - Document creation steps

## Related Documentation

- [whmcs-advanced-containers.md](whmcs-advanced-containers.md)
- [whmcs-advanced-testing.md](whmcs-advanced-testing.md)
