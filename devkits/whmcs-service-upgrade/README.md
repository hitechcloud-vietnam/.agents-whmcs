# WHMCS Service Upgrade Module

Handles service plan changes with proration and billing adjustments.

## Features

- Plan change calculations
- Proration support
- Immediate or end-of-period activation
- Invoice generation
- Upgrade scheduling

## Installation

Copy module to `/path/to/whmcs/modules/servers/serviceupgrade/` and activate.

## Usage

```php
// Calculate proration
$proration = serviceupgrade_CalculateProration($serviceId, $newProductId, 'monthly');
// Returns: credit, charge, difference, old_price, new_price, days_remaining

// Create upgrade request
serviceupgrade_CreateRequest($serviceId, $newProductId, array(
    'activation_type' => 'end_of_period', // or 'immediate'
    'cycle' => 'monthly'
));

// Process scheduled upgrades (run via cron)
$processed = serviceupgrade_ProcessUpgrades();

// Get pending request
$request = serviceupgrade_GetRequest($serviceId);

// Cancel request
serviceupgrade_CancelRequest($requestId);
```

## API Functions

| Function | Description |
|----------|-------------|
| `serviceupgrade_CalculateProration()` | Calculate proration amount |
| `serviceupgrade_CreateRequest()` | Create upgrade request |
| `serviceupgrade_ProcessUpgrades()` | Process pending upgrades |
| `serviceupgrade_ExecuteUpgrade()` | Execute specific upgrade |
| `serviceupgrade_GetRequest()` | Get upgrade request |
| `serviceupgrade_CancelRequest()` | Cancel upgrade request |
