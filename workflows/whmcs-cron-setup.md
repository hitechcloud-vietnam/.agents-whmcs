# WHMCS Cron Setup Workflow

## Purpose
Configure and verify WHMCS automated cron jobs

## Prerequisites
- WHMCS installed
- SSH/cPanel access
- Basic cron knowledge

## Step 1: Understand WHMCS Cron

The WHMCS cron handles:
- Invoice generation
- Payment reminders
- Domain renewals
- Service suspensions
- Ticket automation
- Affiliate commissions
- Report generation

## Step 2: Locate Cron Command

Navigate to: Setup > Automation Settings

Copy the cron command shown:
```bash
/usr/bin/php /var/www/whmcs/crons/cron.php
```

## Step 3: Create Cron Job (SSH)

```bash
crontab -e
```

Add the cron job:
```cron
# WHMCS Cron - Every 5 minutes (recommended)
*/5 * * * * /usr/bin/php /var/www/whmcs/crons/cron.php

# Alternative: Every 30 minutes
0,30 * * * * /usr/bin/php /var/www/whmcs/crons/cron.php

# Alternative: Every hour
0 * * * * /usr/bin/php /var/www/whmcs/crons/cron.php
```

## Step 4: Create Cron Job (cPanel)

1. Log in to cPanel
2. Navigate to: Cron Jobs
3. Select common settings: "Every 5 minutes"
4. Enter command:
   ```
   /usr/bin/php /var/www/whmcs/crons/cron.php
   ```
5. Click "Add New Cron Job"

## Step 5: Configure Cron Security

### Restrict Cron to Localhost
```cron
# Only run from localhost
0,30 * * * * curl -s http://localhost/crons/cron.php >/dev/null 2>&1
```

### Use Cron Lock File
```bash
nano /var/www/whmcs/crons/cron.php
```

Add at the beginning:
```php
<?php
// Cron lock file to prevent overlapping executions
$lockFile = __DIR__ . '/cron.lock';
if (file_exists($lockFile) && (filemtime($lockFile) > (time() - 600))) {
    exit("Cron already running");
}
touch($lockFile);
register_shutdown_function('unlink', $lockFile);
?>
```

## Step 6: Configure Automation Settings

Navigate to: Setup > Automation Settings

### General Settings
```
Enable Automated Tasks: Yes
Automation Interval: Every 5 Minutes
```

### Invoice Settings
```
Auto Generate Invoices: Yes
Invoice Due After Days: 7
Invoice Generation Notice: 3
```

### Suspension Settings
```
Auto Suspend Accounts: Yes
Suspend After Days: 14
Auto Terminate: No
Termination After Days: 30
```

### Domain Settings
```
Auto Renew Domains: No
Domain Renewal Reminder Days: 30, 14, 7, 1
```

### Ticket Settings
```
Auto Close Tickets: Yes
Close After Days: 7
Auto-Lock Tickets: Yes
Lock After Days: 30
```

### Affiliate Settings
```
Auto Register Affiliate: No
Affiliate Commission: 5%
Affiliate Payout: Monthly
```

## Step 7: Verify Cron Execution

### Check Cron Logs
Navigate to: Utilities > System Health Status

Look for "Last Cron Run" - should be recent.

### Manual Cron Test
```bash
cd /var/www/whmcs
/usr/bin/php crons/cron.php
```

### Check Activity Log
Navigate to: Utilities > Logs > Activity Log

Look for cron-related entries.

## Step 8: Configure Email Cron (Optional)

For bulk email sending:
```cron
# Send queued emails every minute
* * * * * /usr/bin/php /var/www/whmcs/crons/emailqueue.php
```

## Step 9: Configure Domain Cron (Optional)

```cron
# Run domain sync every hour
0 * * * * /usr/bin/php /var/www/whmcs/crons/domainssync.php
```

## Step 10: Set Up Monitoring

### Create Monitoring Script
```bash
nano /var/www/whmcs/monitor-cron.sh
```

```bash
#!/bin/bash
LOG_FILE="/var/www/whmcs/logs/cron-monitor.log"
LAST_RUN_FILE="/var/www/whmcs/crons/lastrun"

if [ -f "$LAST_RUN_FILE" ]; then
    LAST_RUN=$(cat "$LAST_RUN_FILE")
    CURRENT=$(date +%s)
    DIFF=$((CURRENT - LAST_RUN))
    
    if [ $DIFF -gt 600 ]; then
        echo "$(date): Cron not running for $DIFF seconds" >> $LOG_FILE
        # Send alert email
        echo "WHMCS Cron not running for $DIFF seconds" | mail -s "WHMCS Cron Alert" admin@domain.com
    fi
fi
```

```bash
chmod +x /var/www/whmcs/monitor-cron.sh
```

### Add Monitoring to Crontab
```cron
# Check cron every 15 minutes
*/15 * * * * /var/www/whmcs/monitor-cron.sh
```

## Step 11: Cron Email Notifications

Prevent cron output emails by redirecting:
```cron
# Send output to /dev/null
*/5 * * * * /usr/bin/php /var/www/whmcs/crons/cron.php >/dev/null 2>&1

# Or redirect to log file
*/5 * * * * /usr/bin/php /var/www/whmcs/crons/cron.php >> /var/www/whmcs/logs/cron.log 2>&1
```

## Step 12: Test All Automation Tasks

Navigate to: Setup > Automation Settings

Click "Run Automation Now" to test all tasks.

## Cron Tasks Reference

| Task | Frequency | Description |
|------|-----------|-------------|
| Invoice Generation | Every 5 min | Creates invoices for due services |
| Payment Reminders | Every 5 min | Sends payment reminder emails |
| Domain Renewals | Every hour | Updates domain status |
| Suspension | Every hour | Suspends overdue accounts |
| Ticket Auto-Close | Every hour | Closes old tickets |
| Affiliate | Daily | Calculates commissions |
| Statistics | Daily | Updates reports |

## Troubleshooting

### Cron Not Running
```bash
# Check if cron is installed
systemctl status cron

# Check cron logs
grep CRON /var/log/syslog

# Verify PHP path
which php
```

### Permission Denied
```bash
chmod 755 /var/www/whmcs/crons/cron.php
chown -R www-data:www-data /var/www/whmcs
```

### Overlapping Executions
Add lock file as shown in Step 5.

## Verification Checklist

- [ ] Cron command configured
- [ ] Automation settings saved
- [ ] Cron runs successfully
- [ ] No overlapping executions
- [ ] Monitoring configured
- [ ] Email notifications working
- [ ] All automation tasks tested
