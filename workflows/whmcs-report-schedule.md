# WHMCS Report Schedule Workflow

## Purpose
Schedule automated report generation and delivery.

## Schedule Setup

### Access
1. Navigate to: Reports > Scheduled
2. Click "Create Schedule"
3. Select report

### Schedule Options
```
Frequencies:
- Daily
- Weekly
- Monthly
- Custom (cron)
```

## Report Selection

### Choose Report
```
Options:
- Pre-built reports
- Custom reports
- Multiple reports
```

### Filter Configuration
```
Set:
- Date range
- Filters
- Groups
- Sort order
```

## Frequency Options

### Daily Schedule
```
Options:
- Every day
- Weekdays only
- Specific time
```

### Weekly Schedule
```
Options:
- Day of week
- Time
- Week start (Sunday/Monday)
```

### Monthly Schedule
```
Options:
- Day of month
- Time
- First/last day
```

### Custom Schedule
```
Cron expression:
- 0 9 * * * (Daily at 9 AM)
- 0 9 * * 1 (Weekly Monday)
- 0 9 1 * * (Monthly 1st)
```

## Delivery Options

### Email Delivery
```
Configure:
- Recipients
- Subject line
- Body text
- Attachments
```

### Multiple Recipients
```
Add:
- Admin emails
- Team members
- External stakeholders
```

### File Delivery
```
Options:
- Save to server
- FTP upload
- Cloud storage
- Email
```

## Format Options

### File Formats
```
Choose:
- CSV
- Excel
- PDF
- Multiple formats
```

### Compression
```
Options:
- No compression
- ZIP archive
- GZ compress
```

## Execution History

### Track Schedule
```
Shows:
- Last run time
- Next run time
- Status
- Errors
```

### Logs
```
View:
- Execution history
- Success/failure
- Runtime
- Output size
```

## Managing Schedules

### Edit Schedule
```
Modify:
- Frequency
- Recipients
- Filters
- Format
```

### Pause/Resume
```
Options:
- Pause temporarily
- Resume when needed
- Modify schedule
```

### Delete Schedule
```
Options:
- Delete completely
- Archive before delete
- Confirm deletion
```

## Conditional Execution

### Smart Scheduling
```
Options:
- Only if data exists
- Skip weekends
- Skip holidays
```

### Triggers
```
Set:
- Run after other reports
- Run after automation
- Conditional based on data
```

## Notifications

### Success Notification
```
Notify:
- Report generated
- Delivered
- Recipient count
```

### Failure Notification
```
Alert:
- Generation failed
- Delivery failed
- Errors details
```

## Best Practices

### Scheduling
```
Guidelines:
- Off-peak hours
- Appropriate frequency
- Right recipients
- Manage size
```

### Maintenance
```
Regular:
- Review schedules
- Remove unused
- Update recipients
```

## Related Workflows
- whmcs-report-export
- whmcs-report-email
- whmcs-report-dashboard