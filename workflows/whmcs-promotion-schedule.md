# WHMCS Promotion Schedule Workflow

## Purpose
Schedule promotional offers to run at specific times.

## Scheduling Options

### Fixed Date Range
```
Start: 2024-01-01
End: 2024-01-31
Promotion runs entire month
```

### Recurring Schedule
```
Weekly: Every Monday
Monthly: 1st of month
Quarterly: Every 3 months
```

### Event-Triggered
```
Run on:
- New year
- Black Friday
- Customer anniversary
```

## Creating Scheduled Promotions

### Step 1: Create Promotion
1. Navigate to: Configuration > Promotions
2. Create new promotion
3. Configure base settings

### Step 2: Set Schedule
```
Schedule Configuration:
- Start Date: 2024-02-01
- End Date: 2024-02-28
- Apply time: 00:00 (midnight)
- Timezone: UTC/Local
```

### Step 3: Configure During Period
```
Behavior:
- Auto-apply: Yes/No
- Code required: Yes/No
- Visible to clients: Yes/No
```

### Step 4: Set Post-Expiration
```
After promotion ends:
- Deactivate promotion
- Show next promotion
- Revert to regular pricing
```

## Recurring Promotions

### Weekly Recurring
```
Example: Weekend Sale
- Every Saturday 9 AM
- Every Sunday 11:59 PM
- Uses per week: Unlimited
```

### Monthly Recurring
```
Example: Monthly Special
- 1st of every month
- Lasts 3 days
- Same promotion monthly
```

### Annual Recurring
```
Example: Anniversary Sale
- Same date each year
- Customer's signup anniversary
- Personalized timing
```

## Pre-Launch

### Preview Mode
```
Before launch:
- Create promotion
- Set to "Preview"
- Test internally
- Schedule activation
```

### Announcement
```
Pre-launch activities:
- Email announcement
- Homepage banner
- Social media teaser
- Countdown timer
```

## During Promotion

### Monitoring
```
Track:
- Uses per hour
- Conversion rate
- Revenue generated
- Remaining inventory
```

### Adjustments
```
During promotion:
- Extend end date
- Increase limits
- Add new conditions
- Adjust pricing
```

## Post-Promotion

### Analysis
```
Review metrics:
- Total uses
- Revenue impact
- Conversion lift
- Customer feedback
```

### Follow-up
```
Post-promotion:
- Thank customers
- Survey for feedback
- Plan next promotion
- Document learnings
```

## Automation

### Cron Job Integration
```
Automatic processing:
- Enable promotion at start
- Disable at end
- Update inventory
- Send notifications
```

### Smart Scheduling
```
Based on:
- Customer behavior
- Inventory levels
- Competitor activity
- Historical data
```

## Related Workflows
- whmcs-promotion-create
- whmcs-promotion-limit
- whmcs-marketing-campaign