# WHMCS Email Debug Workflow

## Overview
This workflow guides you through debugging email delivery issues.

## Prerequisites
- WHMCS email logs
- Mail server access

## Step-by-Step Guide

### Step 1: Check Email Queue
```sql
SELECT * FROM tblemails WHERE id = (SELECT MAX(id) FROM tblemails)
ORDER BY date DESC LIMIT 10;
```

### Step 2: Check Email Log
```bash
# View WHMCS email log
tail -100 /var/www/whmcs/storage/logs/emaillog.log

# Check mail server log
tail -100 /var/log/mail.log

# Check failed emails
grep -i "failed\|error" /var/www/whmcs/storage/logs/emaillog.log
```

### Step 3: Enable Email Logging
```php
// In configuration.php
define('LOG_EMAIL_PERMANENT', true);
```

### Step 4: Test Email Sending
```bash
# Test with WHMCS API
php -r "
require '/var/www/whmcs/init.php';
use WHMCS\Mail\Queue;
\$queue = new Queue();
var_dump(\$queue->getPending(1));
"
```

### Step 5: Common Email Issues
```php
// Issue: Emails not sending
// Fix: Check SMTP settings, verify from address

// Issue: Emails in queue
// Fix: Run cron job for email queue

// Issue: SPF/DKIM failures
// Fix: Update DNS records
```

## Email Debug Checklist

### Investigation
- [ ] Email queue checked
- [ ] Email log reviewed
- [ ] SMTP tested
- [ ] DNS records verified

### Resolution
- [ ] Cron job running
- [ ] SMTP configured
- [ ] DNS records correct
- [ ] Emails sending
