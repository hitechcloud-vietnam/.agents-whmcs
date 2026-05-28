# WHMCS Usage-Based Billing Module

Tracks resource usage and generates usage-based invoices with tiered pricing.

## Features

- Track multiple resource types
- Tiered pricing support
- Included amounts
- Usage history
- Cost calculation
- Invoice generation

## Installation

Copy module to `/path/to/whmcs/modules/servers/usagebilling/` and activate.

## Usage

```php
// Setup resource for service
usagebilling_SetupResource($serviceId, 'bandwidth', 1000); // 1GB included

// Record usage
usagebilling_RecordUsage($serviceId, 'bandwidth', 0.5); // 500MB used

// Set pricing tiers
usagebilling_SetPricingTier($productId, 'bandwidth', array(
    array('min' => 0, 'max' => 1000, 'price' => 0.00),      // First 1GB free
    array('min' => 1000, 'max' => 10000, 'price' => 0.10), // $0.10 per GB
    array('min' => 10000, 'max' => null, 'price' => 0.05), // $0.05 per GB over 10GB
));

// Get usage
$usage = usagebilling_GetUsage($serviceId);
$history = usagebilling_GetUsageHistory($serviceId, 'bandwidth', 30);

// Calculate cost
$cost = usagebilling_CalculateCost($serviceId, 'bandwidth');

// Generate invoice items
$invoice = usagebilling_GenerateInvoice($serviceId);

// Reset usage (after billing)
usagebilling_ResetUsage($serviceId, 'bandwidth');
```

## API Functions

| Function | Description |
|----------|-------------|
| `usagebilling_SetupResource()` | Setup resource tracking |
| `usagebilling_RecordUsage()` | Record usage amount |
| `usagebilling_GetUsage()` | Get current usage |
| `usagebilling_GetUsageHistory()` | Get usage history |
| `usagebilling_SetPricingTier()` | Set tiered pricing |
| `usagebilling_CalculateCost()` | Calculate usage cost |
| `usagebilling_GenerateInvoice()` | Generate invoice items |
| `usagebilling_ResetUsage()` | Reset usage counters |
