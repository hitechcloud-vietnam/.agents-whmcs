# WHMCS Notification Summary Workflow

## Purpose
Create summary notifications that condense multiple events into concise overviews.

## Summary Types

### Real-time Summary
- Immediately after event
- Condensed format
- Key information only

### Period Summary
- End of day/week/month
- Aggregated data
- Metrics included

### Event Summary
- After specific events
- Related information
- Action items

## Creating Summaries

### Step 1: Choose Summary Type
1. Navigate to: Configuration > System > Notifications
2. Select "Create Notification"
3. Choose "Summary" type

### Step 2: Configure Content
```
Summary Content:
- Title: Brief heading
- Summary: Key points (3-5 bullets)
- Actions: Required actions
- Links: Relevant URLs
```

### Step 3: Set Triggers
```
Triggers:
- Single event
- Multiple related events
- Time-based aggregation
```

## Summary Format

### Quick Summary (Email)
```
Subject: Order Summary - {$date}

Hi {$client_name},

Your recent activity:
- 2 new orders totaling $150
- 1 invoice due in 3 days
- 1 support ticket open

Quick actions:
[Pay Invoice] [View Orders] [View Tickets]
```

### Detailed Summary (Slack)
```
*Activity Summary*
Orders: 3 ($450)
Invoices: 2 ($200) - 1 overdue
Support: 5 open - 1 urgent

*Action Required*
- Pay overdue invoice #123
- Respond to urgent ticket

[View Dashboard] [Manage Account]
```

## Components

### Header
- Date/time
- Account name
- Summary type

### Body
- Key metrics
- Important items
- Status updates

### Actions
- Primary CTA
- Secondary links
- Quick links

### Footer
- Help information
- Preferences link
- Support contact

## Summary Templates

### Weekly Sales Summary
```
*Weekly Sales Summary*

This Week: {$orders_count} orders - {$orders_total}
Last Week: {$last_week_orders} orders - {$last_week_total}
Change: {$change_percentage}

Top Products:
1. {$top_product_1}
2. {$top_product_2}

View Full Report: {$report_url}
```

### Client Health Summary
```
*Client Health - {$client_name}*

Services: {$service_count} active
Invoices: {$invoice_count} pending ({$invoice_total})
Tickets: {$ticket_count} open

Status: {$health_score}

[View Details] [Manage Services]
```

## Timing

### Send Frequency
```
- Real-time: Immediate
- Daily: Once per day
- Weekly: Once per week
- Monthly: Once per month
```

### Grouping
- Group by type
- Group by priority
- Group by time

## Best Practices

### Keep It Brief
- Maximum 5-7 key points
- Use bullet points
- Clear hierarchy

### Focus on Action
- What needs attention
- What requires action
- Deadlines and dates

### Include Links
- Quick access to details
- One-click actions
- Deep links to relevant pages

## Related Workflows
- whmcs-notification-digest
- whmcs-notification-scheduling
- whmcs-report-dashboard