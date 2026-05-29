# WHMCS Pro-Rated Billing

## Overview

Pro-rated billing calculates charges based on the actual time a service is active during a billing cycle. This ensures fair billing when clients sign up mid-cycle, upgrade, downgrade, or make changes during their billing period.

## When Pro-Rating Occurs

### Mid-Signup Billing
- Client signs up on day 15 of a 30-day month
- Only pays for remaining days of billing cycle
- First full cycle begins next month

### Upgrade Mid-Cycle
- Client upgrades from Basic to Pro plan
- Difference charged for remaining days
- Both plans prorated appropriately

### Downgrade Mid-Cycle
- Client downgrades plans
- Credit applied for remaining days
- May result in credit on next invoice

### Add-on Addition
- New add-on added mid-cycle
- Prorated from addition date to cycle end

## Pro-Rate Calculation Formula

### Standard Formula

```
Pro-Rated Amount = (Monthly Price / Days in Period) * Remaining Days
```

### Example Calculation

```
Billing cycle: Monthly (30 days)
Monthly price: $30
Days remaining: 18

Pro-Rated Amount = ($30 / 30) * 18 = $18.00
```

### Days in Period Calculation

```php
// Standard month (30 days)
$daysInPeriod = date('t', strtotime($billingDate));

// Actual calendar days
$startDate = strtotime('2024-01-20');
$endDate = strtotime('2024-02-01');
$daysRemaining = ($endDate - $startDate) / 86400;  // seconds per day
```

## Upgrade Pro-Ration

### Basic to Pro Upgrade

```
Old Plan: Basic - $20/month
New Plan: Pro - $30/month

Day of upgrade: Day 15 of 30-day cycle
Days remaining: 16

Price difference: $30 - $20 = $10
Pro-Rated charge: ($10 / 30) * 16 = $5.33
```

### Multiple Components

```php
// Upgrade with add-ons
$serviceUpgrade = 5.33;   // Service proration
$addonUpgrade = 2.67;     // Add-on proration
$totalUpgrade = 8.00;     // Combined charge
```

### Upgrade with Setup Fee

```php
// New setup fee on upgrade
$proration = 5.33;       // Difference for remaining days
$setupFee = 10.00;        // New setup fee
$totalCharge = 15.33;
```

## Downgrade Pro-Ration

### Credit Calculation

```
Old Plan: Pro - $30/month
New Plan: Basic - $20/month

Day of downgrade: Day 20 of 30-day cycle
Days remaining: 11

Price difference: $30 - $20 = $10
Credit amount: ($10 / 30) * 11 = $3.67
```

### Credit Application

```php
// Credit options
Option 1: Apply to next invoice
Option 2: Add to client credit balance
Option 3: Refund to payment method
```

## Add-on Pro-Ration

### Adding Add-on Mid-Cycle

```
Add-on price: $5/month
Day added: Day 25 of 30-day cycle
Days remaining: 6

Pro-Rated charge: ($5 / 30) * 6 = $1.00
```

### Removing Add-on

```
Add-on price: $5/month
Day removed: Day 10 of 30-day cycle
Days remaining: 21

Credit: ($5 / 30) * 21 = $3.50
```

## Prorate Display Settings

### Configuration

**Configuration > Invoices > Prorate Display**

```php
[
    'show_prorate_warning' => true,
    'show_next_due_date' => true,
    'prorate_text' => 'Pro-rated charge for remaining days',
    'show_upgrade_credit' => true
]
```

### Invoice Line Item

```
Description: Pro-rated upgrade from Basic to Pro (16 days)
Amount: $5.33
Tax: $0.53
Total: $5.86
```

## Configuration Options

### Prorate Behavior Settings

**Configuration > Billing > Prorate Settings**

| Setting | Description | Options |
|---------|-------------|---------|
| Prorate Upgrades | Charge difference for upgrades | Yes/No |
| Credit Downgrades | Apply credit for downgrades | Yes/No |
| Prorate Billing | Use prorated amounts | Always/First Only/Never |
| Billing Cycle | Pro-rate to next cycle | Natural/Billing Date |
| Prorate Method | Calculation method | Daily/Hourly |

### Pro-Rate Calculations

```php
// Daily calculation (most common)
$dailyRate = $monthlyPrice / 30;
$prorateAmount = $dailyRate * $remainingDays;

// Hourly calculation (for precision)
$hourlyRate = $monthlyPrice / 720;  // 30 days * 24 hours
$prorateAmount = $hourlyRate * $remainingHours;

// Actual calendar days
$daysInMonth = cal_days_in_month($month, $year);
$prorateAmount = ($monthlyPrice / $daysInMonth) * $remainingDays;
```

## Configuration Settings

### Pro-Rate Display

**Configuration > Invoices > Display Settings**

```php
// Invoice line item format
{
    "type" => "prorate",
    "description" => "Pro-rated charge for upgrade",
    "amount" => 5.33,
    "from" => "Basic",
    "to" => "Pro",
    "days" => 16,
    "percentage" => 53.33
}
```

### Upgrade/Downgrade Messaging

```php
// Customer notification
{
    "title" => "Plan Upgrade",
    "message" => "You are upgrading from Basic to Pro. 
                  A prorated charge of $5.33 for the 
                  remaining 16 days will be applied.
                  Your next invoice will be $30.00."
}
```

## Invoice Line Items

### Pro-Rated Entry Format

```php
// Invoice line item
[
    'type' => 'prorate',
    'description' => 'Pro-rated upgrade (Basic to Pro)',
    'relid' => 123,           // Service ID
    'amount' => 5.33,
    'taxable' => 1
]
```

### Separate Line Items

```
+--------------------------------------------------+
| Description                          | Amount    |
+--------------------------------------------------+
| Pro-rated upgrade (Basic to Pro)     |           |
| 16 days remaining                   |           |
+--------------------------------------------------+
|                                     | $5.33     |
+--------------------------------------------------+
| Prorated from: Jan 15 to Jan 31     |           |
| Pro-rated to: Feb 15                |           |
+--------------------------------------------------+
```

## Hooks and Customization

### Prorate Calculation Hook

```php
// Hook: PreCalculateProrate
add_hook('PreCalculateProrate', 1, function($vars) {
    // $vars['amount']
    // $vars['days']
    // $vars['serviceid']
    
    // Modify calculation
    return [
        'amount' => $vars['amount'],
        'round' => true,
        'decimals' => 2
    ];
});
```

### Prorate Display Hook

```php
// Hook: ProrateDisplay
add_hook('ProrateDisplay', 1, function($vars) {
    // Customize display text
    return [
        'title' => 'Mid-cycle upgrade charge',
        'description' => 'Difference for remaining ' . $vars['days'] . ' days'
    ];
});
```

## API Functions

```php
// Calculate prorate amount
$params = [
    'serviceid' => 123,
    'newprice' => 30.00,
    'oldprice' => 20.00,
    'cycle' => 'monthly',
    'daysremaining' => 16
];
$result = localAPI('CalculateProrate', $params);

// Get prorate settings
$params = [
    'userid' => 1
];
$result = localAPI('GetProrateSettings', $params);
```

## Edge Cases

### Leap Year Months

```php
// February (29 days in 2024)
$daysInFeb = 29;
$dailyRate = $monthlyPrice / $daysInFeb;
```

### Short Months

```php
// April (30 days)
$daysInApril = 30;
$dailyRate = $monthlyPrice / $daysInApril;

// End of month handling
// Day 31 in February = Day 28/29
// Day 30 in February = Day 28/29
```

### Year Boundary

```php
// Prorate spanning year end
// Dec 25 to Jan 5
$decAmount = ($monthlyPrice / 31) * 7;
$janAmount = ($monthlyPrice / 31) * 5;
$totalProrate = $decAmount + $janAmount;
```

## Common Scenarios

### Scenario 1: Mid-Month Signup

```
Client signs up: January 15
Billing cycle: Monthly
Monthly price: $30

Days in January: 31
Days remaining: 17 (Jan 15-31)
Prorate: ($30 / 31) * 17 = $16.45

First invoice: $16.45
Next invoice: $30.00 (Feb 15)
```

### Scenario 2: Upgrade Day 1

```
Client upgrades: February 1
Days remaining: 28 (Feb 1-28)
Price difference: $10

Prorate: ($10 / 28) * 28 = $10.00
Full first cycle charge
```

### Scenario 3: Multiple Changes

```
Day 5: Upgrade Service (+$5.33)
Day 15: Add Add-on (+$2.50)
Day 25: Downgrade Service (-$3.00)

Total adjustments on next invoice
```

## Best Practices

1. **Clear communication**: Show exact prorated amounts
2. **Accurate calculations**: Use consistent formula
3. **Document changes**: Log all proration events
4. **Round appropriately**: Use 2 decimal places
5. **Handle edge cases**: Account for all month lengths

## Related Documentation

- [Recurring Billing](./whmcs-recurring-billing.md)
- [Service Upgrade](./whmcs-service-upgrade.md)
- [Service Downgrade](./whmcs-service-downgrade.md)
- [Invoice Generation](./whmcs-invoice-generation.md)