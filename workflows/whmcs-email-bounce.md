# WHMCS Email Bounce Handling Workflow

## Purpose
Configure and manage email bounce handling for better deliverability.

## Bounce Types

### Hard Bounces
- Invalid email address
- Domain doesn't exist
- Mailbox full
- Action: Remove from list immediately

### Soft Bounces
- Server temporarily unavailable
- Message too large
- Action: Retry later (3-5 times)

## Configuration

### Step 1: Enable Bounce Handling
1. Navigate to: Configuration > System > Settings > Mail
2. Enable "Bounce Handling"
3. Configure POP3/IMAP inbox for bounces

### Step 2: Set Up Bounce Inbox
```
Inbox Configuration:
- Email: bounces@yourdomain.com
- Protocol: POP3 or IMAP
- Server: mail.yourdomain.com
- Port: 995 (POP3 SSL) or 993 (IMAP SSL)
- Username: bounces@yourdomain.com
- Password: xxxxxxxx
```

### Step 3: Define Bounce Rules
Default rules:
- 550: Invalid recipient -> Hard bounce
- 551: User not local -> Hard bounce
- 421: Service not available -> Soft bounce
- 450: Requested action not taken -> Soft bounce

### Step 4: Configure Actions
```
On Hard Bounce:
- Mark client email as invalid
- Disable email notifications
- Log for review

On Soft Bounce:
- Retry sending
- Track failure count
- Escalate after X retries
```

## Automatic Processing

### Step 1: Cron Configuration
Add to cron:
```
/usr/bin/php -q /path/to/whmcs/admin/cron.php -EmailBounceHandler
```

### Step 2: Processing Schedule
- Run every 15 minutes
- Process in batches of 100
- Log all actions

## Bounce Management

### Review Bounce Log
1. Navigate to: Utilities > Logs > Bounce Log
2. Review bounce events
3. Identify patterns
4. Clean invalid addresses

### Manual Intervention
1. Export bounce list
2. Review hard bounces
3. Update client records
4. Consider email verification service

## Best Practices

### Prevention
1. Use double opt-in
2. Validate email at signup
3. Regular list cleaning
4. Monitor sender reputation

### Maintenance
1. Weekly bounce review
2. Monthly list cleaning
3. Quarterly health check

## Related Workflows
- whmcs-email-tracking
- whmcs-email-verification
- whmcs-email-log