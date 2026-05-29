# WHMCS Notification Digest Workflow

## Purpose
Create digest notifications that aggregate multiple events.

## Digest Types

### Daily Digest
- Summary of daily activity
- Sent at specific time
- Includes all events from past 24 hours

### Weekly Digest
- Weekly summary
- Trend analysis
- Key metrics

### Monthly Digest
- Monthly report
- Performance overview
- Recommendations

## Configuration

### Step 1: Create Digest Template
1. Navigate to: Configuration > System > Notifications
2. Click "Create Notification"
3. Select "Digest" type

### Step 2: Configure Digest Settings
```
Digest Settings:
- Frequency: Daily/Weekly/Monthly
- Time: 08:00 AM
- Timezone: Client/Admin
- Minimum items: 1
- Maximum items: 50
```

### Step 3: Set Aggregation Rules
```
Group by:
- Event type
- Client
- Product

Sort by:
- Date (newest first)
- Priority
- Amount
```

## Digest Template

### Header
```
Hi {$client_name},

Here's your daily digest for {$date}.
```

### Content Sections
```
## New Orders
{if $orders}
{foreach $orders as $order}
- Order #{$order.number}: {$order.total}
{/foreach}
{else}
No new orders today.
{/if}

## Invoices
{if $invoices}
{foreach $invoices as $invoice}
- Invoice #{$invoice.number}: {$invoice.amount}
{/foreach}
{else}
No invoices today.
{/if}
```

### Footer
```
View your account: {$client_area_url}

Manage preferences: {$notification_preferences_url}
```

## Digest Events

### Available for Digest
```
- New orders
- Invoice reminders
- Support tickets
- Service updates
- Domain expirations
- Announcements
```

### Select Events for Digest
1. Configure which events to include
2. Set minimum threshold
3. Group related events

## Formatting

### Compact Format
```
Daily Digest - {$date}

Orders: 3 new ($450 total)
Invoices: 2 pending ($200)
Support: 5 open tickets

View details: {$digest_url}
```

### Detailed Format
```
Daily Digest - {$date}

New Orders:
1. Order #123 - Shared Hosting Pro - $50
   Client: John Doe

Invoices:
1. Invoice #456 - Due Today - $100
   Status: Pending

View full report: {$report_url}
```

## Timing Options

### Send Time
- Early morning (6-8 AM)
- Midday (12 PM)
- Evening (6-8 PM)
- Based on client timezone

### Day Selection (Weekly)
```
Options:
- Monday (start of week)
- Friday (end of week)
- Saturday/Sunday (weekend)
```

### Day Selection (Monthly)
```
Options:
- 1st of month
- Last day of month
- Specific date (15th)
```

## Conditional Digests

### Only Send if...
```
{if $new_orders > 0 OR $pending_invoices > 0}
   Send digest
{else}
   Don't send (no activity)
{/if}
```

### Minimum Activity
```
Only send if:
- At least 1 order OR
- At least 1 invoice OR
- At least 1 support ticket
```

## Related Workflows
- whmcs-notification-scheduling
- whmcs-notification-summary
- whmcs-report-dashboard