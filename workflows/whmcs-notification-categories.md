# WHMCS Notification Categories Workflow

## Purpose
Organize notifications into categories for better management.

## Available Categories

### Billing
- Invoice notifications
- Payment confirmations
- Overdue alerts
- Refund notices

### Orders
- New orders
- Order status changes
- Order cancellations
- Fraud alerts

### Support
- Ticket notifications
- Escalations
- Response alerts
- Feedback requests

### Services
- Activation notices
- Renewal reminders
- Suspension alerts
- Termination notices

### Domains
- Registration confirmations
- Transfer notifications
- Renewal warnings
- Expiration alerts

### Marketing
- Promotional notifications
- Newsletter updates
- Special offers
- Announcements

## Creating Categories

### Step 1: Define Category
1. Navigate to: Configuration > System > Notifications
2. Select "Categories" tab
3. Click "Add Category"
4. Enter name and description

### Step 2: Assign Notifications
1. Select category
2. Add notifications
3. Set priority
4. Configure channels

### Step 3: Set Display Order
1. Drag to reorder
2. Set default view
3. Configure visibility

## Category Settings

### Appearance
- Icon selection
- Color coding
- Display name

### Permissions
- Admin access
- Client access
- Group restrictions

### Behavior
- Notification grouping
- Sorting options
- Default filters

## Category-Based Filtering

### Client Filtering
```
Client sees:
- Billing (invoices, payments)
- Services (their products)
- Support (their tickets)
- Announcements
```

### Admin Filtering
```
Admin sees:
- All categories
- Department-specific
- Role-based view
```

## Best Practices

### Naming Conventions
- Use descriptive names
- Consistent naming pattern
- Avoid too many categories

### Organization
- Group by type
- Prioritize critical
- Keep manageable count

## Related Workflows
- whmcs-notification-create
- whmcs-notification-groups
- whmcs-notification-priority