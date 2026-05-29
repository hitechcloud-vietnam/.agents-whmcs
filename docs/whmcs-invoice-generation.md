# WHMCS Invoice Generation

## Overview

Invoice generation in WHMCS is an automated process that creates billing documents for clients based on their active services, domain registrations, add-on products, and any additional billing items.

## Generation Triggers

### Automated Generation
- **Cron-based generation**: Invoices are generated based on the configured billing cycle (daily, weekly, monthly, quarterly, annually)
- **Due date calculation**: Invoices are created X days before the service renewal date based on `GenerateInvoiceBeforeDays` setting
- **Prorated invoices**: Generated when upgrades or mid-cycle changes occur

### Manual Generation
- **Single invoice**: Generate invoice for specific client/service via Admin area
- **Batch generation**: Generate invoices for multiple services via bulk action
- **Custom invoice**: Create ad-hoc invoice with custom line items

## Invoice Number Format

Configure in `Configuration > Invoices > Invoice Number Format`

```
{year}{month}{sequence}
Example: 20240500001
```

### Available Tokens
| Token | Description |
|-------|-------------|
| `{year}` | 4-digit year |
| `{month}` | 2-digit month |
| `{day}` | 2-digit day |
| `{sequence}` | Sequential number (configurable padding) |
| `{clientid}` | Client ID number |
| `{invoiceid}` | Invoice ID number |

## Invoice Line Items

### Automatic Line Items
- Service/Product billing
- Domain registration/renewal
- Add-on products
- Configurable option additions

### Manual Line Items
- One-time charges
- Credits (negative amount)
- Custom descriptions
- Custom pricing

## Invoice Status Flow

```
Draft -> Pending -> Paid -> Overdue -> Cancelled
                  
Auto-generation creates in Pending status
```

### Status Descriptions
- **Draft**: Not yet issued, can be edited
- **Pending**: Issued, awaiting payment
- **Paid**: Payment received in full
- **Overdue**: Payment past due date
- **Cancelled**: Invoice cancelled
- **Refunded**: Payment was refunded

## Tax Calculation

Tax is calculated per line item based on:
- Client country/state tax rules
- Product tax settings
- Invoice level tax exemption

### Tax Configuration
```php
// In Configuration > Tax Configuration
Tax Enabled: Yes/No
Tax Level: Inclusive/Exclusive
Compound Tax: Yes/No
```

## Invoice Date Settings

| Setting | Location | Default |
|---------|----------|---------|
| Issue Date | Auto (generation date) | Current date |
| Due Date | Configuration | +7 days |
| Next Due Date | Service billing cycle | Varies |
| End Due Date | Grace period | +30 days |

## Automation Settings

Located in `Configuration > System > Automation Settings`

```php
// Invoice Generation Settings
GenerateInvoiceBeforeDays = 7      // Days before due date
AutoSetup = false                  // Auto provision on payment
AutoTermination = false            // Terminate on overdue
OverideSuspensionUntil =           // Suspension grace period
```

## Invoice Creation Hooks

```php
// Hook: InvoiceCreated
add_hook('InvoiceCreated', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['userid']
    // $vars['total']
    // $vars['tax']\n});
```

## Email Notifications

Invoices trigger automated emails:
- **New Invoice**: Sent when invoice generated
- **Invoice Reminder**: Configured reminder schedule
- **Invoice Overdue**: After due date passes
- **Invoice Paid**: Confirmation to client

## Related Documentation

- [Invoice Templates](./whmcs-invoice-templates.md)
- [Invoice Reminders](./whmcs-invoice-reminders.md)
- [Payment Methods](./whmcs-payment-methods.md)
- [Tax Rules](./whmcs-tax-rules.md)