# WHMCS Cron Automation Workflow

## Overview
This workflow sets up comprehensive cron-based automation for WHMCS using the native cron system.

## Prerequisites
- WHMCS installation with root/admin cron access
- SSH access to server
- Basic understanding of cron syntax

## Step-by-Step Process

### Step 1: Configure WHMCS Cron Command
1. Access WHMCS admin area
2. Go to **Configuration > System Settings > Automation**
3. Copy the cron command shown
4. Default command format:
   ```bash
   php -q /path/to/whmcs/admin/cron.php
   ```

### Step 2: Set Up System Crontab
1. SSH to server:
   ```bash
   ssh admin@your-server.com
   ```

2. Open crontab editor:
   ```bash
   crontab -e
   ```

3. Add WHMCS cron entries:
   ```bash
   # Main WHMCS cron - every 5 minutes
   */5 * * * * php -q /var/www/html/whmcs/admin/cron.php

   # Additional automation cron - every hour
   0 * * * * php -q /var/www/html/whmcs/admin/cron.php with:misc/cleanup

   # Daily maintenance - early morning
   0 3 * * * php -q /var/www/html/whmcs/admin/cron.php with:maint/daily
   ```

### Step 3: Configure Module-Specific Cron Tasks

#### Domain Sync Cron
```bash
# Domain registrar sync - every 6 hours
0 */6 * * * php -q /var/www/html/whmcs/admin/cron.php with:domain/sync
```

#### License Check Cron
```bash
# License validation - twice daily
0 6,18 * * * php -q /var/www/html/whmcs/admin/cron.php with:license/check
```

### Step 4: Create Custom Cron Modules

#### Module Structure
```
/whmcs/includes/clients/
└── cron_tasks/
    ├── MyCustomCron.php
    └── MyCustomCronTrait.php
```

#### Cron Module Class
```php
<?php

namespace WHMCS\Cron\Tasks;

class DailyReportGeneration extends Task
{
    protected $name = 'Daily Report Generation';
    protected $description = 'Generates daily business reports';
    protected $frequency = [1, 0, 0]; // Run once per day at hour 1

    public function __invoke()
    {
        $this->generateRevenueReport();
        $this->generateNewClientsReport();
        $this->generateServiceHealthReport();
        $this->sendAdminSummary();

        return "Daily reports generated successfully";
    }

    private function generateRevenueReport()
    {
        $today = date('Y-m-d');
        $yesterday = date('Y-m-d', strtotime('-1 day'));

        $stats = Capsule::select("
            SELECT
                SUM(total) as total_revenue,
                COUNT(*) as transaction_count,
                AVG(total) as average_transaction
            FROM tblinvoices
            WHERE datepaid >= ?
            AND datepaid < ?
            AND status = 'Paid'
        ", [$yesterday, $today]);

        Capsule::table('mod_daily_reports')->insert([
            'report_type' => 'revenue',
            'report_date' => $yesterday,
            'data' => json_encode($stats[0]),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    private function sendAdminSummary()
    {
        sendEmail(
            'admin',
            'daily_summary',
            [
                'date' => date('Y-m-d'),
                'stats' => $this->compileStats()
            ]
        );
    }
}
```

### Step 5: Implement Hook-Based Cron Extensions
```php
<?php
// /includes/hooks/cron_extensions.php

// Custom hook to run with cron
add_hook('CronJobDaily', 1, function($vars) {
    logActivity('Daily cron extension executing');

    // Sync with external API
    syncExternalContacts();

    // Process queued emails
    processEmailQueue();

    // Update service metrics
    updateServiceMetrics();
});

add_hook('CronJobHourly', 1, function($vars) {
    // Quick health check tasks
    checkServiceAvailability();
    monitorQueueDepth();
});
```

### Step 6: Configure WHMCS Built-in Cron Modules

#### Invoice Automation
Navigate to **Configuration > System Settings > Automation**:
```
Invoice Generation Settings:
- Generate Invoices: [X] Enabled
- Invoice Day: [1] (1st of month)
- Days Before Due Date: [7]

Payment Reminders:
- Send Reminder: [X] Enabled
- First Reminder: [1] days after due
- Second Reminder: [7] days after due
- Third Reminder: [14] days after due

Service Suspension:
- Suspend Services: [X] Enabled
- Days After Due Date: [14]

Termination:
- Terminate Services: [ ] Enabled
- Days After Due Date: [30]
```

#### Domain Automation
Navigate to **Configuration > System Settings > Automation**:
```
Domain Sync Settings:
- Auto Lock Domains: [X] Enabled
- Auto Renewal: [X] Enabled
- Sync Interval: [6] hours
```

### Step 7: Create Cron Wrapper Script
```bash
#!/bin/bash
# /opt/whmcs-cron/cron-wrapper.sh

LOG_FILE="/var/log/whmcs-cron.log"
WHMCS_PATH="/var/www/html/whmcs"
CRON_USER="apache"

log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> $LOG_FILE
}

# Acquire lock to prevent concurrent runs
LOCK_FILE="/var/run/whmcs-cron.lock"
if [ -f $LOCK_FILE ]; then
    LOCK_AGE=$(($(date +%s) - $(stat -c %Y $LOCK_FILE)))
    if [ $LOCK_AGE -lt 600 ]; then
        log_message "Previous cron still running, skipping"
        exit 1
    fi
    log_message "Stale lock file found, removing"
    rm -f $LOCK_FILE
fi

touch $LOCK_FILE
trap "rm -f $LOCK_FILE" EXIT

log_message "Starting WHMCS cron"

cd $WHMCS_PATH
/usr/bin/php -q admin/cron.php >> $LOG_FILE 2>&1
EXIT_CODE=$?

if [ $EXIT_CODE -eq 0 ]; then
    log_message "WHMCS cron completed successfully"
else
    log_message "WHMCS cron failed with exit code $EXIT_CODE"
fi

exit $EXIT_CODE
```

### Step 8: Monitor Cron Health
```bash
# Check if cron is running
ps aux | grep "cron.php"

# Check last run time
tail -20 /var/log/whmcs-cron.log

# Verify cron schedule
crontab -l
```

### Step 9: Set Up Cron Monitoring

#### Monitoring Script
```bash
#!/bin/bash
# /opt/whmcs-cron/monitor.sh

LAST_RUN_FILE="/var/run/whmcs-last-run"
ALERT_EMAIL="admin@example.com"

# Update last run timestamp
date +%s > $LAST_RUN_FILE

# Check cron health
CURRENT_TIME=$(date +%s)
LAST_RUN=$(cat $LAST_RUN_FILE)
TIME_DIFF=$((CURRENT_TIME - LAST_RUN))

# Alert if cron hasn't run in 10 minutes
if [ $TIME_DIFF -gt 600 ]; then
    echo "WHMCS cron may not be running. Last run: ${TIME_DIFF} seconds ago" \
        | mail -s "WHMCS Cron Alert" $ALERT_EMAIL
fi
```

Add to crontab:
```bash
*/5 * * * * /opt/whmcs-cron/monitor.sh
```

### Step 10: Troubleshooting Cron Issues

#### Common Issues

1. **Cron not executing**:
   - Verify cron is installed: `service crond status`
   - Check crontab: `crontab -l`
   - Test command manually: `php -q /path/to/cron.php`

2. **Permission errors**:
   - Ensure PHP has execute permissions
   - Check file ownership: `chown -R www-data:www-data /path/to/whmcs`

3. **Timeout issues**:
   - Increase PHP max execution time
   - Split long tasks into smaller chunks

#### Debug Mode
```php
// Add to cron.php temporarily for debugging
define('CRON_DEBUG', true);
ini_set('display_errors', 1);
error_reporting(E_ALL);
```

## Cron Schedule Best Practices

| Task Type | Schedule | Rationale |
|-----------|----------|-----------|
| Main cron | */5 * * * * | WHMCS recommended |
| Health checks | */15 * * * * | Frequent monitoring |
| Email queue | */10 * * * * | Responsive delivery |
| Reports | 0 5 * * * | Early morning before business |
| Backups | 0 2 * * * | Off-peak hours |
| Large syncs | 0 */6 * * * | Every 6 hours |
| Cleanup | 0 3 * * * | Late night maintenance |

## Related Workflows
- [WHMCS Scheduled Tasks](./whmcs-scheduled-tasks.md)
- [WHMCS Event-Driven Automation](./whmcs-event-driven-automation.md)
- [WHMCS Queue Processing](./whmcs-queue-processing.md)