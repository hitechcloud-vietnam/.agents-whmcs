# WHMCS Email Template Edit Workflow

## Purpose
Modify an existing email template to update content, styling, or configuration.

## Prerequisites
- WHMCS admin access
- Existing email template to edit

## Steps

### Step 1: Locate Template
1. Navigate to: Configuration > System > Email Templates
2. Use search/filter to find the template
3. Click on template name to edit

### Step 2: Edit Subject Line
1. Modify the email subject line
2. Consider adding urgency/time-sensitive elements
3. Test variable usage in subject

### Step 3: Update Email Content
1. Edit body content in WYSIWYG or HTML
2. Preserve existing variable placeholders
3. Update branding elements (logo, colors)
4. Check responsive design for mobile

### Step 4: Update Sender Information
1. Modify From Name if needed
2. Update From Email address
3. Adjust CC/BCC recipients

### Step 5: Review Conditions
1. Check "Only send if" conditions
2. Update trigger conditions if needed
3. Ensure conditions still make sense

### Step 6: Save Changes
1. Click "Save Changes"
2. Check "Changelog" tab for history
3. Run test email to verify changes

## Best Practices
- Make incremental changes
- Test after each significant edit
- Maintain version backup
- Document changes for team reference

## Testing Checklist
- [ ] All variables work correctly
- [ ] Formatting preserved
- [ ] Links functional
- [ ] Mobile responsive

## Related Workflows
- whmcs-email-template-create
- whmcs-email-template-clone
- whmcs-email-test