# WHMCS Notification Scheduling Workflow

## Purpose
Schedule notifications for future delivery at specific times.

## Scheduling Options

### Immediate
- Send immediately on trigger
- Default behavior

### Delayed
- Send X hours/days after trigger
- Use for follow-ups

### Scheduled
- Send at specific date/time
- Use for announcements

### Recurring
- Send daily/weekly/monthly
- Use for reports, digests

## Configure Scheduling

### Step 1: Edit Notification
1. Navigate to: Configuration > System > Notifications
2. Select notification
3. Click "Schedule" tab

### Step 2: Set Timing
```
Timing Options:
- Send immediately
- Send after X hours
- Send on specific date/time
- Send recurring: Daily/Weekly/Monthly
```

### Step 3: Configure Recurrence
```
Daily:
- Time: 09:00 AM
- Timezone: Client local / Admin timezone

Weekly:
- Day: Monday
- Time: 09:00 AM

Monthly:
- Day: 1st of month
- Time: 09:00 AM
```

## Common Use Cases

### Follow-up Sequence
```
Day 0: Initial notification
Day 1: Reminder
Day 3: Second reminder
Day 7: Escalation
Day 14: Final notice
```

### Announcement Scheduling
```
- Schedule for business hours
- Consider time zones
- Avoid weekends
- Peak engagement times
```

### Digest Notifications
```
Daily Digest:
- Time: 08:00 AM
- Aggregate: Previous day events

Weekly Summary:
- Day: Monday
- Time: 09:00 AM
- Include: Weekly metrics
```

## Time Zone Handling

### Options
```
- Admin timezone
- Client timezone
- UTC
- Specific timezone
```

### Example
```
Client in US East Coast:
- Schedule for 9 AM local
- System converts to send at correct time
```

## Queue Management

### Batch Processing
1. Schedule multiple notifications
2. Cron processes queue
3. Send in batches

### Rate Limiting
- Limit per hour
- Spread over time
- Prevent overwhelming

## Monitoring Scheduled

### Check Schedule
1. Navigate to: Utilities > Scheduled Tasks
2. View pending notifications
3. Check next run time

### Reschedule
1. Find scheduled notification
2. Update time
3. Save changes

## Cancel Scheduled

### Option 1: Delete
1. Find scheduled notification
2. Delete from queue
3. Notification won't send

### Option 2: Edit
1. Find scheduled notification
2. Modify content
3. Keep same schedule

## Testing Scheduled

### Test Mode
1. Enable test mode
2. Trigger notification
3. Verify schedule logic
4. Check timing

## Cron Requirements

### Ensure Cron Running
```
/usr/bin/php -q /path/to/whmcs/admin/cron.php
```

### Frequency
- Run every 5-15 minutes
- Process scheduled notifications
- Update queue

## Related Workflows
- whmcs-notification-create
- whmcs-notification-digest
- whmcs-email-schedule