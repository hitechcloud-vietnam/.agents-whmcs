# WHMCS Usage-Based Billing

## Overview

Usage-based billing (metered billing) allows you to charge customers based on actual resource consumption rather than flat fees. This model is common for cloud services, bandwidth, storage, API calls, and other measurable resources.

## Enabling Usage Billing

**Configuration > System > General > Usage-Based Billing**

```php
// Enable usage billing
UsageBasedBillingEnabled: true
```

## Metered Product Configuration

### Product Setup

**Products > Create/Edit Product > Usage Billing Tab**

```php
[
    'usagemodel' => 'per_unit',
    'unitprice' => 0.05,        // $0.05 per unit
    'minimumfee' => 10.00,      // Minimum monthly charge
    'maximumfee' => 1000.00,    // Cap maximum charge
    'resetcycle' => 'monthly',   // Billing reset period
    'includeincludedunits' => true
]
```

### Included Units

Provide free allocation with each billing cycle:

```php
[
    'includedunits' => 1000,
    'includedprice' => 20.00,   // Base price for included units
    'overageprice' => 0.05      // Price per unit over included
]
```

## Usage Types

### Bandwidth
```php
[
    'type' => 'bandwidth',
    'unit' => 'GB',
    'steps' => [
        ['from' => 0, 'to' => 100, 'price' => 0.10],
        ['from' => 101, 'to' => 500, 'price' => 0.08],
        ['from' => 501, 'to' => null, 'price' => 0.05]
    ]
]
```

### Storage
```php
[
    'type' => 'storage',
    'unit' => 'GB',
    'flatrate' => 0.20,         // $0.20 per GB
    'minimum' => 10,            // Minimum 10 GB
    'maximum' => 10000
]
```

### API Calls
```php
[
    'type' => 'api_calls',
    'unit' => 'calls',
    'tiers' => [
        ['range' => '0-10000', 'price' => 0],
        ['range' => '10001-50000', 'price' => 0.001],
        ['range' => '50001+', 'price' => 0.0005]
    ]
]
```

### Compute Hours
```php
[
    'type' => 'compute',
    'unit' => 'cpu_hours',
    'price' => 0.10
]
```

## Usage Tracking

### Automatic Tracking

```php
// Log usage via module hook
$params = [
    'accountid' => $serviceId,
    'date' => date('Y-m-d H:i:s'),
    'quantity' => 150,
    'type' => 'bandwidth',
    'description' => 'Daily bandwidth consumption'
];

localAPI('LogUsage', $params);
```

### Manual Usage Entry

**Admin: Client > Service > Usage tab**

```php
// Manual entry form
[
    'date' => '2024-05-15',
    'quantity' => 500,
    'type' => 'storage',
    'notes' => 'Month-end snapshot'
]
```

### CSV Import

```csv
serviceid,date,type,quantity,description
123,2024-05-01,bandwidth,1500,Monthly transfer
123,2024-05-01,storage,250,Storage snapshot
```

## Usage Processing Cron

### Cron Command

```bash
# Process usage and generate invoices
php -q /path/to/whmcs/crons/process_usage_billing.php
```

### Schedule

```php
// Run daily to capture usage
0 0 * * * php -q /whmcs/crons/process_usage_billing.php
```

### Processing Steps

1. Aggregate usage records for billing period
2. Calculate charges based on tiered pricing
3. Generate usage invoice
4. Apply minimum/maximum caps
5. Reset usage counters (if configured)

## Usage Pricing Models

### Flat Rate Overage

```php
// Included: 1000 units (in base price)
// Overage: $0.10 per unit
$billableUnits = max(0, $actualUnits - $includedUnits);
$charge = $billableUnits * $overagePrice;
```

### Tiered Pricing

```php
// Volume-based pricing
function calculateTieredUsage($units, $tiers) {
    $total = 0;
    $remaining = $units;
    
    foreach ($tiers as $tier) {
        $tierSize = $tier['max'] - $tier['min'] + 1;
        $inTier = min($remaining, $tierSize);
        $total += $inTier * $tier['price'];
        $remaining -= $inTier;
        
        if ($remaining <= 0) break;
    }
    return $total;
}
```

### Stepped Pricing

```php
// First 100 units: $0.10
// Next 400 units: $0.08
// Additional: $0.05

$charge = 0;
if ($units <= 100) {
    $charge = $units * 0.10;
} elseif ($units <= 500) {
    $charge = (100 * 0.10) + (($units - 100) * 0.08);
} else {
    $charge = (100 * 0.10) + (400 * 0.08) + (($units - 500) * 0.05);
}
```

## Usage Invoice Generation

### Automatic Invoicing

```php
// Generated at end of billing period
[
    'invoice_items' => [
        ['description' => 'Bandwidth Usage - 2,450 GB', 'amount' => 245.00],
        ['description' => 'Storage Usage - 500 GB', 'amount' => 100.00],
        ['description' => 'API Calls - 75,000', 'amount' => 75.00]
    ],
    'subtotal' => 420.00,
    'tax' => 84.00,
    'total' => 504.00
]
```

### Invoice Display

```smarty
{foreach $invoice.lineitems as $item}
    {if $item.type == 'usage'}
        <tr>
            <td>{$item.description}</td>
            <td>{$item.quantity} units @ {$item.unitprice}</td>
            <td>{$item.amount}</td>
        </tr>
    {/if}
{/foreach}
```

## Client Usage Dashboard

### Customer Usage Display

**Client Area > Service > Usage Statistics**

```php
// Display usage metrics
[
    'current_usage' => 2450,      // GB
    'included_usage' => 1000,     // GB
    'overage_usage' => 1450,      // GB
    'projected_charge' => 145.00, // Estimated overage cost
    'usage_percentage' => 245      // 245% of included
]
```

### Usage Notifications

```php
// Threshold alerts
[
    ['threshold' => 75, 'notify' => true],
    ['threshold' => 90, 'notify' => true],
    ['threshold' => 100, 'notify' => true, 'include_upgrade' => true]
]
```

## Webhook Integration

### Real-Time Usage Events

```php
// Register webhook for usage events
// Endpoint: https://your-app.com/webhook/usage
{
    "event" => "usage.recorded",
    "serviceid" => 123,
    "type" => "bandwidth",
    "quantity" => 150,
    "timestamp" => "2024-05-15T10:30:00Z"
}
```

### External Metering Integration

```php
// External billing system integration
$usageData = [
    'provider' => 'cloud_provider',
    'serviceid' => 'ext-12345',
    'metrics' => [
        'bandwidth' => 5000,
        'storage' => 200,
        'compute' => 1200
    ],
    'period' => [
        'start' => '2024-05-01',
        'end' => '2024-05-31'
    ]
];

// Sync to WHMCS
localAPI('SyncUsageData', $usageData);
```

## Usage Reports

### Admin Reports

**Reports > Billing > Usage Summary**

```php
// Report filters
[
    'period' => 'monthly',
    'product' => 'cloud_server',
    'client' => null,
    'groupby' => 'type'
]
```

### Export Options

- CSV download
- PDF report
- Date range selection
- Product filtering
- Client filtering

## API Functions

```php
// Get usage for service
$params = [
    'serviceid' => 123,
    'startdate' => '2024-05-01',
    'enddate' => '2024-05-31',
    'type' => 'bandwidth'
];
$result = localAPI('GetUsage', $params);

// Add usage record
$params = [
    'serviceid' => 123,
    'date' => '2024-05-15',
    'type' => 'bandwidth',
    'quantity' => 500,
    'description' => 'Manual entry'
];
$result = localAPI('AddUsage', $params);
```

## Hooks

```php
// Hook: UsageCalculated
add_hook('UsageCalculated', 1, function($vars) {
    // $vars['serviceid']
    // $vars['type']
    // $vars['quantity']
    // $vars['amount']
    
    // Modify calculation or add fees
    return ['amount' => $vars['amount']];
});

// Hook: UsageInvoiceCreated
add_hook('UsageInvoiceCreated', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['serviceid']
    // $vars['usage_items']
});
```

## Best Practices

1. **Clear documentation**: Explain usage metrics to clients
2. **Reasonable thresholds**: Alert before overage charges
3. **Fair pricing**: Don't overcharge for overage
4. **Accurate tracking**: Ensure reliable measurement
5. **Regular reconciliation**: Verify usage calculations

## Related Documentation

- [Recurring Billing](./whmcs-recurring-billing.md)
- [Pro-Rated Billing](./whmcs-pro-rated-billing.md)
- [Invoice Generation](./whmcs-invoice-generation.md)
- [Service Usage Limits](./whmcs-service-usage-limits.md)