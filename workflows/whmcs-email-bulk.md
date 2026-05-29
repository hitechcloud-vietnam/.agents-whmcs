# WHMCS Bulk Email Workflow

## Purpose
Send bulk emails to multiple clients for marketing, notifications, or announcements.

## Options in WHMCS

### Option 1: Admin Mass Mail
1. Navigate to: Utilities > Mass Mail
2. Click "Send Message"
3. Select criteria:
   - All clients
   - Specific client group
   - Specific product
   - Specific status
   - Custom filter

4. Compose message
5. Schedule or send immediately
6. Track results

### Option 2: Newsletter Module
1. Install newsletter module (e.g., Mailchimp, Campaign Monitor)
2. Sync client list
3. Create campaign in external tool
4. Send from external provider

### Option 3: Custom Module
1. Use API to create custom solution
2. Third-party email service integration

## Mass Mail Configuration

### Step 1: Access Mass Mail
1. Login to admin
2. Navigate to: Utilities > Mass Mail > Send Message

### Step 2: Select Recipients
```
Filters:
- Status: Active, Inactive, Suspended
- Client Group: VIP, Free Trial, etc.
- Products: Specific products owned
- Date Range: Registration date, etc.
- Country: Specific countries
```

### Step 3: Compose Email
1. Select or create email template
2. Use personalization variables
3. Preview with sample data
4. Set subject line with merge tags

### Step 4: Configure Sending
- Rate limit (emails per hour)
- Schedule delivery time
- Enable tracking (if supported)

### Step 5: Send & Monitor
1. Send test email first
2. Start bulk send
3. Monitor progress
4. Check delivery reports

## Best Practices

### Prevent Spam Issues
- Warm up sending reputation
- Clean email list first
- Avoid spam trigger words
- Include unsubscribe link
- Authenticate with SPF/DKIM/DMARC

### Content Guidelines
- Clear subject line
- Single call-to-action
- Mobile-friendly design
- Short paragraphs

## Related Workflows
- whmcs-marketing-email
- whmcs-email-schedule
- whmcs-email-tracking