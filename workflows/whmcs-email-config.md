# WHMCS Email Configuration Workflow

## Purpose
Configure email settings for WHMCS transactional emails

## Prerequisites
- WHMCS installed
- SMTP server access or mail service
- Admin access

## Step 1: Navigate to Email Settings

Navigate to: Setup > Email > Mail Configuration

## Step 2: Choose Mail Method

### Method A: PHP Mail (Default)
```
Mail Type: PHP Mail
```
Simple but less reliable.

### Method B: SMTP (Recommended)
```
Mail Type: SMTP
SMTP Host: smtp.example.com
SMTP Port: 587
SMTP Username: your-email@example.com
SMTP Password: [your-password]
SMTP Security: TLS
```

### Method C: Amazon SES
```
Mail Type: Amazon SES
Access Key: [your-key]
Secret Key: [your-key]
Region: us-east-1
```

## Step 3: Configure SMTP (Gmail Example)

```
SMTP Host: smtp.gmail.com
SMTP Port: 587
SMTP Username: your-email@gmail.com
SMTP Password: [App Password]
SMTP Security: TLS
```

## Step 4: Configure SMTP (Office 365 Example)

```
SMTP Host: smtp.office365.com
SMTP Port: 587
SMTP Username: your-email@domain.com
SMTP Password: [your-password]
SMTP Security: TLS
```

## Step 5: Configure SMTP (Mailgun Example)

```
SMTP Host: smtp.mailgun.org
SMTP Port: 587
SMTP Username: postmaster@domain.com
SMTP Password: [your-api-key]
SMTP Security: TLS
```

## Step 6: Test Email Configuration

Navigate to: Utilities > System > Send Test Email

1. Enter recipient email
2. Click "Send Test Email"
3. Verify delivery

## Step 7: Configure Email Branding

Navigate to: Setup > Email > Email Templates

### Customize Templates
1. Select template
2. Edit content
3. Add company logo
4. Save changes

### Key Templates
- Client Welcome Email
- Password Reset
- Invoice Created
- Invoice Paid
- Support Ticket Opened
- Domain Renewal Notice

## Step 8: Configure Email Preferences

Navigate to: Setup > Email > Email Preferences

```
Default Email Format: HTML
Email Character Set: UTF-8
Bounce Setting: [As needed]
```

## Step 9: Set Up Bounce Handling (Optional)

Navigate to: Setup > Email > Bounce Settings

1. Create dedicated bounce email
2. Configure IMAP connection
3. Set bounce rules

## Step 10: Configure Domain-Level Email

Navigate to: Setup > Email > Domain-Based Email Routing

For multi-tenant setups, configure email routing per domain.

## Email Troubleshooting

### Emails Not Sending
- Verify SMTP credentials
- Check firewall rules
- Review error logs in Utilities > Logs

### Emails Going to Spam
- Use proper SPF, DKIM, DMARC
- Enable tracking judiciously
- Maintain sender reputation

### SMTP Connection Failed
```
Error: Connection refused
Solution: Check port 587/465 is open
```
