# WHMCS Notification Triggers Workflow

## Purpose
Configure event triggers for automated notifications.

## Available Event Categories

### Client Events
```
- Client Created
- Client Updated
- Client Login
- Password Changed
- Email Changed
- Phone Changed
```

### Order Events
```
- Order Submitted
- Order Pending
- Order Accepted
- Order Cancelled
- Order Fraud Detected
- Order Completed
```

### Invoice Events
```
- Invoice Created
- Invoice Paid
- Invoice Overdue
- Invoice Reminder Sent
- Invoice Cancelled
- Invoice Refund Processed
```

### Service Events
```
- Service Created
- Service Activated
- Service Suspended
- Service Terminated
- Service Upcoming Renewal
- Service Expired
```

### Domain Events
```
- Domain Registered
- Domain Transfer Initiated
- Domain Transfer Completed
- Domain Renewed
- Domain Expiring Soon
- Domain Expired
```

### Support Events
```
- Ticket Opened
- Ticket Replied
- Ticket Escalated
- Ticket Closed
- Ticket Feedback Received
```

## Configuring Triggers

### Step 1: Select Event Type
1. Navigate to: Configuration > System > Notifications
2. Click "Add Notification"
3. Select event from dropdown

### Step 2: Set Trigger Conditions
```
Filter Options:
- Status: Active only, Inactive only, All
- Client Group: Specific groups
- Product Type: Specific products
- Amount Range: Min/Max values
- Date Range: Time-based triggers
```

### Step 3: Configure Timing
```
Timing Options:
- Immediate: Send right away
- Delayed: Send X hours after event
- Scheduled: Send at specific time
- Batch: Collect and send daily/weekly
```

### Step 4: Set Recipients
```
Recipient Options:
- Client: Always, Sometimes, Never
- Admin: All admins, Specific admin, Admin group
- Other: Custom email addresses
```

## Custom Triggers via API

### Register Custom Event
```php
add_hook('CustomEvent', 1, function($vars) {
    // Trigger notification
});
```

### Trigger from Module
```php
sendNotification('CustomEvent', [
    'client_id' => 123,
    'message' => 'Custom event occurred'
]);
```

## Trigger Testing

### Step 1: Test Mode
1. Enable "Test Mode" for trigger
2. Perform action that triggers event
3. Check notification delivery

### Step 2: Verify
1. Check notification log
2. Verify correct template used
3. Check all variables populated

## Troubleshooting
- Trigger not firing: Check conditions
- Wrong template: Verify template assignment
- Variables not working: Check syntax

## Related Workflows
- whmcs-notification-create
- whmcs-notification-rules
- whmcs-notification-channels