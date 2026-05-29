# WHMCS Email Sending Configuration Workflow

## Purpose
Configure WHMCS email sending settings for reliable delivery.

## Steps

### Step 1: Access Email Settings
1. Navigate to: Configuration > System > Settings > Mail
2. Review current configuration

### Step 2: Choose Sending Method

#### Option A: PHP Mail (Default)
- Simple setup
- Uses server mail transfer agent
- May have deliverability issues

#### Option B: SMTP
1. Select "SMTP" as mail type
2. Enter SMTP credentials:
   - Host: smtp.example.com
   - Port: 587 (TLS) or 465 (SSL)
   - Username: your-email@domain.com
   - Password: SMTP password
   - Security: TLS/SSL
3. Test connection

#### Option C: Amazon SES
1. Set up SES in AWS
2. Configure in WHMCS:
   - SMTP Host: email-smtp.us-east-1.amazonaws.com
   - Port: 587
   - Enable TLS
   - Use SES credentials

#### Option D: Mailgun/SendGrid/etc.
1. Set up account with provider
2. Enter SMTP credentials from provider
3. Verify domain/sender

### Step 3: Configure Additional Options
1. **Email Charset**: UTF-8 (default)
2. **Word Wrap**: 80 characters recommended
3. **Enable HTML**: Enable for rich emails
4. **Strip HTML**: For plain text version

### Step 4: Test Configuration
1. Click "Send Test Email"
2. Verify delivery
3. Check spam folder
4. Review email headers

### Step 5: Monitor & Optimize
1. Check admin email logs
2. Review bounce rates
3. Track deliverability

## Related Workflows
- whmcs-email-smtp
- whmcs-email-test
- whmcs-email-tracking