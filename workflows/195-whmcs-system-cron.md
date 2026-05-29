---
name: whmcs-system-cron
description: Configure system cron jobs in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, cron, automation, system]
---

# WHMCS Cron Configuration Workflow

## Purpose
Step-by-step guide for configuring WHMCS cron jobs.

## Prerequisites
- WHMCS admin access
- Cron access (command line or Cpanel)
- Automation requirements defined

## Step 1: Access Cron Settings
- Navigate to Configuration > System
- Click "Automation"
- View cron configuration
- Check current cron URL

## Step 2: Configure Cron Command
Default cron command:
```
php -q /path/to/whmcs/crons/cron.php
```

Alternative:
```
 wget -q -O /dev/null https://domain.com/crons/cron.php
```

## Step 3: Set Cron Schedule
Recommended schedule:
- Every 5 minutes (recommended)
- Every 15 minutes (minimum)
- Every hour (basic)

## Step 4: Configure Automation Tasks
In WHMCS Automation settings:
- Invoice generation
- Payment reminders
- Domain sync
- Cancellation processing
- Overdue handling
- Credit card processing

## Step 5: Configure Individual Tasks
Set timing for:
- Daily tasks
- Weekly tasks
- Monthly tasks
- Quarterly tasks
- Custom schedules

## Step 6: Test Cron Execution
- Run cron manually
- Check output
- Verify task completion
- Review cron log
- Fix any errors

## Step 7: Monitor Cron Health
- Check cron execution log
- Monitor for failures
- Verify task completion
- Set up cron monitoring
- Create alerts for failures

## Common Cron Tasks
- Generate invoices
- Send reminders
- Process payments
- Sync domains
- Update pricing
- Clean old data

## Related Workflows
- whmcs-cron-configuration
- whmcs-cron-setup
- whmcs-cron-automation