# WHMCS Email Queue Management Workflow

## Purpose
Manage and monitor the email queue for reliable delivery.

## Access Email Queue

### Step 1: View Queue Status
1. Navigate to: Utilities > System > Email Queue
2. View pending emails
3. Check queue statistics

### Step 2: Queue Information
- Total emails in queue
- Average processing time
- Failed email count
- Next scheduled send

## Queue Configuration

### Step 1: Configure Queue Settings
1. Navigate to: Configuration > System > Settings > Mail
2. Find "Email Queue" section

### Settings:
```
- Queue Enabled: Yes/No
- Max emails per run: 100
- Run interval: 5 minutes
- Retry failed: Yes
- Max retries: 3
```

### Step 2: Adjust for Volume
For high-volume sites:
- Increase batch size
- Reduce interval
- Add server resources

## Managing Queue

### View Pending Emails
1. Email list shows:
   - Recipient
   - Subject
   - Queued time
   - Retry count

### Manual Queue Processing
1. Click "Process Queue Now"
2. Monitor processing
3. Check results

### Clear Queue
- Clear all failed emails
- Clear entire queue
- Use caution - may lose important emails

## Troubleshooting Queue Issues

### Emails Stuck in Queue
1. Check cron job status
2. Verify SMTP configuration
3. Check server resources
4. Review error logs

### High Failure Rate
1. Check email addresses
2. Verify SMTP credentials
3. Review server reputation
4. Check firewall settings

## Queue Optimization

### Best Practices
1. Use reliable SMTP service
2. Set appropriate batch sizes
3. Monitor queue health
4. Regular maintenance

### Performance Tuning
- Small sites: Default settings
- Medium sites: Increase batch to 200
- Large sites: Consider dedicated email service

## Related Workflows
- whmcs-email-sending
- whmcs-email-log
- whmcs-email-smtp-debug