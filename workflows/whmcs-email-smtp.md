# WHMCS SMTP Setup Workflow

## Purpose
Configure SMTP for improved email deliverability and tracking.

## Steps

### Step 1: Choose SMTP Provider
Popular options:
- Gmail (limited for bulk)
- SendGrid
- Mailgun
- Amazon SES
- Postmark
- SMTP.com

### Step 2: Prepare SMTP Credentials
1. Create account with provider
2. Verify domain/email
3. Generate API key or app password
4. Note SMTP host and port

### Step 3: Configure WHMCS
1. Navigate to: Configuration > System > Settings > Mail
2. Set "Mail Type" to "SMTP"
3. Enter credentials:

```
SMTP Host: smtp.provider.com
SMTP Port: 587 (TLS) or 465 (SSL)
SMTP Username: your@email.com
SMTP Password: your-password
SMTP Security: TLS
```

### Step 4: Test Connection
1. Click "Send Test Email"
2. Enter test recipient email
3. Verify successful delivery

### Step 5: Verify Deliverability
1. Send test to multiple email providers
2. Check spam folders
3. Verify SPF/DKIM/DMARC records
4. Use email testing tools (Mail Tester, MXToolbox)

## Troubleshooting SMTP Issues

### Authentication Failed
- Double-check username/password
- Use app-specific password if 2FA enabled
- Verify SMTP credentials

### Connection Timeout
- Check firewall settings
- Verify correct port (587/465)
- Ensure TLS/SSL settings match

### Emails Going to Spam
- Set up SPF record
- Configure DKIM signature
- Add DMARC policy
- Warm up sending reputation

## Related Workflows
- whmcs-email-sending
- whmcs-email-smtp-debug
- whmcs-email-tracking