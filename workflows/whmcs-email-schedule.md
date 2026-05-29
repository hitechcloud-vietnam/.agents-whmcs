# WHMCS Scheduled Email Workflow

## Purpose
Set up automated scheduled emails to send at specific times or intervals.

## Scheduled Email Triggers

### Built-in Scheduled Emails
- Invoice Reminder (X days before/after due date)
- Overdue Invoice Notice
- Domain Renewal Notice
- Service Renewal Notice
- Welcome Email
- Password Reset

### Custom Scheduled Emails
Create via Automation Settings:

### Step 1: Configure Automation
1. Navigate to: Configuration > System > Automation
2. Review scheduled tasks

### Step 2: Email Template Scheduling
1. Go to: Configuration > System > Email Templates
2. Edit template
3. Set "Scheduled for" option
4. Configure trigger conditions

### Step 3: Cron Job Setup
1. Ensure cron is configured in WHMCS
2. Cron command location:
```
/usr/bin/php -q /path/to/whmcs/admin/cron.php
```

### Step 4: Verify Schedule
1. Check "Automation Log" in admin
2. Review next scheduled run times
3. Test with sample data

## Custom Scheduled Emails via API

### Create Scheduled Email
```php
// Using WHMCS API
$postData = array(
    "action" => "SendEmail",
    "messagename" => "Custom Template",
    "customtype" => "scheduled",
    "scheduledate" => "2024-01-15 09:00",
    "clientid" => 123
);
```

## Best Practices
1. Schedule during business hours for support emails
2. Avoid sending during night hours (spam)
3. Space out bulk emails
4. Monitor delivery rates

## Related Workflows
- whmcs-email-trigger
- whmcs-email-schedule
- whmcs-notification-scheduling