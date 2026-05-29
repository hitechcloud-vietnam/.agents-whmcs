# WHMCS Email Test Workflow

## Purpose
Test email templates to ensure proper rendering and delivery.

## Testing Methods

### Method 1: Built-in Test Feature
1. Navigate to: Configuration > System > Email Templates
2. Edit template
3. Click "Send Test Email"
4. Enter recipient email
5. Click "Send"

### Method 2: Preview Feature
1. Edit template
2. Click "Preview"
3. Select client for sample data
4. Review rendered output

### Method 3: SMTP Test
1. Navigate to: Configuration > System > Settings > Mail
2. Click "Send Test Email"
3. Enter recipient
4. Verify delivery

## Test Checklist

### Visual Testing
- [ ] Header displays correctly
- [ ] Logo/images load
- [ ] Text formatting is correct
- [ ] Links work properly
- [ ] Footer displays
- [ ] Mobile responsiveness

### Variable Testing
- [ ] Client name populates
- [ ] Email address is correct
- [ ] Product/service details accurate
- [ ] Dates formatted correctly
- [ ] Amounts calculate properly

### Email Client Testing
- [ ] Gmail (web)
- [ ] Outlook (desktop)
- [ ] Apple Mail
- [ ] Mobile email apps
- [ ] Spam folder check

### Spam Score Testing
Use tools like:
- Mail Tester
- MXToolbox
- SpamAssassin checker

## Test Scenarios

### Scenario 1: New Invoice
1. Create test invoice
2. Trigger invoice email
3. Verify PDF attachment
4. Check all variables

### Scenario 2: Welcome Email
1. Create test client
2. Trigger welcome email
3. Verify all client data
4. Test login link

### Scenario 3: Renewal Notice
1. Set service to near expiration
2. Trigger renewal email
3. Verify countdown
4. Check CTA button

## Common Issues
- Variables not replacing
- Broken images
- Spam triggers
- HTML rendering issues

## Related Workflows
- whmcs-email-template-create
- whmcs-email-smtp-debug
- whmcs-email-tracking