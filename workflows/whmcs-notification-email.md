# WHMCS Email Notifications Workflow

## Purpose
Configure email as a notification channel in WHMCS.

## Overview
Email notifications use WHMCS's email system to deliver alerts to clients and admins.

## Configuration

### Step 1: Configure Email Settings
1. Navigate to: Configuration > System > Settings > Mail
2. Verify email configuration
3. Test SMTP connection

### Step 2: Enable Email Notifications
1. Go to: Configuration > System > Notifications
2. Select "Providers" tab
3. Enable "Email" provider
4. Configure default sender

### Step 3: Set Up Admin Notifications
1. Enable admin email notifications
2. Set admin email addresses
3. Configure notification categories

## Notification Types

### Client Notifications
```
- Welcome emails
- Order confirmations
- Invoice notifications
- Payment confirmations
- Service activation
- Renewal reminders
- Support updates
```

### Admin Notifications
```
- New orders (high value)
- Fraud alerts
- Payment failures
- Service cancellations
- Support escalations
- System errors
```

## Email Template Configuration

### Step 1: Create Template
1. Navigate to: Configuration > System > Email Templates
2. Create new or edit existing
3. Enable for notifications

### Step 2: Template Variables
```
{$client_name}
{$client_email}
{$alert_title}
{$alert_message}
{$action_url}
{$priority}
{$event_type}
```

### Step 3: Configure Trigger
1. Select event type
2. Assign email template
3. Set conditions
4. Test notification

## Notification Settings

### Frequency Options
- Immediate: Send right away
- Digest: Collect and send daily/weekly
- Batch: Send every X hours

### Priority Levels
- Low: Non-urgent updates
- Medium: Standard notifications
- High: Important alerts
- Critical: Urgent actions required

## Advanced Configuration

### Separate Recipients
1. Client email: Enabled/Disabled
2. Admin email: Enabled/Disabled
3. Custom emails: Add specific addresses

### Conditional Emails
```
{if $invoice_amount > 1000}
   Send to: manager@company.com
{/if}
```

## Best Practices

### Subject Lines
- Keep under 50 characters
- Include key information
- Use clear urgency indicators

### Content
- Start with action required
- Include all relevant details
- Add clear CTA button
- Provide contact information

## Testing

### Test Email Notification
1. Select notification
2. Click "Send Test"
3. Enter test email
4. Verify delivery
5. Check spam folder

## Troubleshooting

### Emails Not Sending
- Check email configuration
- Verify cron job running
- Check notification is active
- Review email logs

### Wrong Template
- Verify template assignment
- Check condition logic
- Test with sample data

## Related Workflows
- whmcs-notification-create
- whmcs-notification-channels
- whmcs-email-template-create