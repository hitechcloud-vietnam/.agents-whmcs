# WHMCS Email Trigger Workflow

## Purpose
Configure automatic email triggers based on system events.

## Available Triggers

### Client Triggers
- New client account created
- Client details updated
- Password reset requested
- Email verification

### Order Triggers
- New order placed
- Order pending
- Order accepted/completed
- Order cancelled
- Order fraud detected

### Invoice Triggers
- Invoice created
- Invoice paid
- Invoice overdue
- Invoice reminder
- Invoice cancelled

### Service Triggers
- Service activated
- Service suspended
- Service terminated
- Service upcoming renewal
- Service expired

### Domain Triggers
- Domain registration
- Domain transfer
- Domain renewal
- Domain expiration warning
- Domain deleted

### Support Triggers
- New ticket created
- Ticket replied
- Ticket escalated
- Ticket closed
- Feedback requested

## Configuring Triggers

### Step 1: Access Email Templates
1. Navigate to: Configuration > System > Email Templates
2. Each template has associated triggers

### Step 2: Enable/Disable Triggers
1. Find template (e.g., "Invoice Created")
2. Check "Active" status
3. Configure conditions

### Step 3: Set Conditions
```
{if $invoice_status == "Paid"}
   Thank you for payment!
{/if}
```

### Step 4: Test Triggers
1. Perform action that triggers email
2. Check email delivery
3. Verify content accuracy

## Custom Triggers via Hook

### Example: Custom Event Hook
```php
add_hook('AfterModuleCreate', 1, function($vars) {
    // Send custom email
});
```

## Troubleshooting
- Check cron job is running
- Verify email is active
- Check spam folders
- Review automation logs

## Related Workflows
- whmcs-email-template-create
- whmcs-email-schedule
- whmcs-notification-triggers