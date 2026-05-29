# WHMCS Recurring Billing

## Overview

Recurring billing in WHMCS handles automatic generation of invoices for ongoing services based on configured billing cycles. This system manages subscription renewals, ensures consistent revenue collection, and automates the billing workflow.

## Billing Cycle Configuration

### Standard Cycles

| Cycle | Days | Description |
|-------|------|-------------|
| Monthly | 30 | Billed monthly |
| Quarterly | 90 | Billed every 3 months |
| Semi-Annual | 180 | Billed every 6 months |
| Annual | 365 | Billed yearly |
| Biennial | 730 | Billed every 2 years |
| Triennial | 1095 | Billed every 3 years |

### Custom Cycles

```php
// Configuration > Products > Billing Cycle
// Add custom cycle
[
    'name' => '45 Days',
    'days' => 45,
    'setupfee' => 0,
    'price' => null
]
```

## Automatic Invoice Generation

### Cron-Based Generation

WHMCS automation runs via cron job:

```bash
# crontab entry
* * * * * php -q /path/to/whmcs/crons/autocron.php
```

### Generation Timing

**Configuration > System > Automation Settings**

| Setting | Description |
|---------|-------------|
| GenerateInvoiceBeforeDays | Days before due date to generate |
| GenerateDueInvoicesDays | Days after due date to generate |
| AutoProcessPendingOrders | Auto-provision on payment |
| AutoTerminationDays | Days after due to terminate |

### Default Settings
```php
[
    'GenerateInvoiceBeforeDays' => 7,
    'AutoSuspensionDays' => 7,
    'AutoTerminationDays' => 14,
    'GenerateDueInvoicesDays' => 0
]
```

## Next Due Date Calculation

### Standard Calculation

```php
// From initial due date
$nextDue = $lastDueDate + $billingCycleDays;

// Example: Monthly from Jan 15
// Jan 15 -> Feb 15 -> Mar 15 -> etc.
```

### Prorated Next Due

When service activated mid-cycle:

```php
// Activated: Jan 20
// Billing cycle: Monthly
// First invoice: $20 (Jan 20-31)
// Next due: Feb 20
// Monthly from Mar 20 onwards
```

## Prorated Billing

### Activation Proration

```php
// Example calculation
// Monthly price: $30
// Days in month: 31
// Days remaining: 12 (Jan 20-31)
// Prorated amount: $30 * (12/31) = $11.61
```

### Mid-Cycle Upgrades

```php
// Upgrading on day 15 of 30-day cycle
// Old plan: $20/month
// New plan: $30/month
// Remaining days: 16
// Additional charge: $30 * (16/30) = $16.00
```

## First Payment vs Recurring

### First Payment Includes

- Setup fee (if applicable)
- Prorated amount (if applicable)
- First full billing cycle
- Any add-ons or configurable options

### Recurring Payments

- Standard billing cycle amount
- Price adjustments (if any)
- Add-on changes
- Configurable option changes

## Service Status During Billing

### Active Services

- Invoices generated automatically
- Services remain active
- Payment expected within due period

### Suspended Services

- No invoices generated while suspended
- Suspended date preserved
- On reactivation, catch-up billing occurs

### Terminated Services

- No further invoices
- Service removed from billing queue
- Can be restored within retention period

## Billing Automation Rules

### Auto Suspension

```php
// Configuration > Automation
AutoSuspendEnabled: true
SuspendDays: 7               // Days after due date
SuspendTerminatedProducts: true
SuspendReason: "Non-payment"
```

### Auto Termination

```php
// Configuration > Automation
AutoTerminationEnabled: true
TerminationDays: 14          // Days after due date
TerminateTerminatedProducts: true
```

### Welcome Email Sequence

```php
// Automated emails
Day 0: Invoice Created
Day 3: First Reminder
Day 7: Second Reminder (Suspension Warning)
Day 14: Final Reminder (Termination Warning)
Day 15: Auto Suspension
Day 22: Auto Termination
```

## Payment Gateway Integration

### Automatic Payment Collection

```php
// When invoice due
// 1. Attempt automatic charge
// 2. Success -> Mark paid, continue service
// 3. Fail -> Apply failed payment handling
// 4. Retry configured attempts
```

### Retry Configuration

```php
// Payment Retry Settings
MaxRetries: 3
RetryIntervals: [1, 3, 7]  // Days between retries
RetryAmount: Full invoice
```

## Billing Cycles & Revenue Recognition

### Monthly Billing
- Recognized monthly
- Cash basis or accrual basis
- Monthly reporting

### Annual Billing
- Revenue recognition over 12 months
- Deferred revenue liability
- Monthly reporting allocation

## Hooks for Recurring Billing

```php
// Hook: ServiceRecurringBilling
add_hook('ServiceRecurringBilling', 1, function($vars) {
    // $vars['serviceid']
    // $vars['userid']  
    // $vars['amount']
    // $vars['billingcycle']
    
    // Custom logic, e.g., usage-based adjustment
});

// Hook: InvoiceGenerated
add_hook('InvoiceGenerated', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['userid']
    // $vars['total']
});
```

## API Functions

```php
// Get next billing date
$params = [
    'serviceid' => 123
];
$result = localAPI('GetNextBillingDate', $params);

// Update billing cycle
$params = [
    'serviceid' => 123,
    'billingcycle' => 'annual'
];
$result = localAPI('UpdateService', $params);
```

## Troubleshooting Billing Issues

### Missing Invoices

1. Check automation cron is running
2. Verify service next due date
3. Check for invoice generation errors
4. Review suspend/termination status

### Incorrect Amounts

1. Verify product pricing
2. Check for manual overrides
3. Review configurable options
4. Confirm add-on pricing

### Double Billing

1. Check for duplicate products
2. Verify service has single billing entry
3. Review order modifications

## Best Practices

1. **Monitor automation**: Ensure cron runs successfully
2. **Set appropriate grace periods**: Don't suspend too quickly
3. **Clear payment terms**: Communicate expectations
4. **Regular reconciliation**: Match invoices to services
5. **Test proration**: Verify calculations before billing

## Related Documentation

- [Invoice Generation](./whmcs-invoice-generation.md)
- [Pro-Rated Billing](./whmcs-pro-rated-billing.md)
- [Service Suspension](./whmcs-service-suspension.md)
- [Dunning Settings](./whmcs-dunning-settings.md)