# WHMCS Email Attachments Workflow

## Purpose
Configure and manage file attachments in WHMCS email templates.

## Attachment Types

### 1. Static Attachments (Manual)
Files uploaded directly to templates:
1. Navigate to: Configuration > System > Email Templates
2. Edit template
3. Find "Attachments" section
4. Click "Browse" to upload
5. Save template

### 2. Dynamic Attachments (Automated)
Files generated or linked automatically:
- Invoices (PDF)
- Quotes (PDF)
- Contracts
- Agreements

## Common Attachment Scenarios

### Invoice PDF
1. Go to: Configuration > System > Email Templates
2. Find "Invoice Created" template
3. Attachments section shows auto-attached invoice PDF
4. Can add additional files

### Terms & Conditions
1. Upload PDF to /attachments/ or WHMCS media
2. In template, add attachment reference
3. Include link in email body as backup

### Welcome Kit
1. Upload welcome_guide.pdf
2. Upload getting_started.pdf
3. In welcome email template, attach both

## File Size Limits
- Default limit: 25MB
- Configurable in /configuration.php
- Check email provider limits

## Supported File Types
- PDF (recommended)
- DOC/DOCX
- Images (PNG, JPG)
- ZIP (compressed files)
- TXT

## Best Practices
1. Use PDF for universal compatibility
2. Compress images to reduce size
3. Include downloadable links as backup
4. Test attachments in multiple email clients

## Troubleshooting

### Attachments Not Sending
- Check file size limits
- Verify file permissions
- Ensure file path is correct
- Check server email logs

### Corrupted Attachments
- Re-upload file
- Check file encoding
- Try different file format

## Related Workflows
- whmcs-email-template-create
- whmcs-email-test
- whmcs-email-bulk