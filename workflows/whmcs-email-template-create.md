# WHMCS Email Template Create Workflow

## Purpose
Create a new custom email template in WHMCS for automated client communications.

## Prerequisites
- WHMCS admin access
- Basic understanding of template variables
- Content ready for the template

## Steps

### Step 1: Access Email Templates
1. Log into WHMCS admin area (/admin)
2. Navigate to: Configuration > System > Email Templates
3. Click "Create New Email Template"

### Step 2: Configure Basic Settings
1. **Template Name**: Enter a descriptive name (e.g., "Welcome Email - Premium")
2. **Unique ID**: Auto-generated from name, can customize
3. **Email Subject**: Enter the subject line with variables allowed
4. **Attachments**: Optionally attach files (upload or select from library)

### Step 3: Design Email Content
1. **From Name**: Set sender name (can use variables like {$client_name})
2. **From Email**: Set sender email address
3. **Copy To**: Add CC/BCC recipients if needed
4. **Email Body**: Write content using WYSIWYG editor or raw HTML

### Step 4: Use Template Variables
Common variables for email templates:
```
{$client_name}
{$client_first_name}
{$client_last_name}
{$client_email}
{$service_id}
{$product_name}
{$domain}
{$invoice_num}
{$amount}
{$date}
{$due_date}
{$login_url}
```

### Step 5: Set Conditions (Optional)
1. Enable "Only send if" conditions
2. Add rules (e.g., product type, client group, status)

### Step 6: Configure Language & Branding
1. Select applicable languages
2. Set brand/division if using multi-brand
3. Enable/disable plain text version

### Step 7: Save and Test
1. Click "Save Changes"
2. Use "Send Test Email" to verify
3. Select a test client to receive the sample

## Testing Checklist
- [ ] Email renders correctly in webmail
- [ ] Email renders correctly in desktop clients
- [ ] All variables populate correctly
- [ ] Links are functional
- [ ] Images display properly

## Related Workflows
- whmcs-email-template-edit
- whmcs-email-template-clone
- whmcs-email-variables
- whmcs-email-test