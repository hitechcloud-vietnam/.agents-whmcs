# WHMCS Ticket Attachment Workflow

## Purpose
Step-by-step guide for managing ticket file attachments in WHMCS.

## Prerequisites
- Ticket exists
- File prepared
- File type allowed
- Storage available

## Workflow Steps

### Step 1: Attachment Identification
- Identify file to attach
- Verify file exists
- Check file content
- Document purpose

### Step 2: File Validation
- Verify allowed file type
- Check file size
- Scan for malware
- Validate format

### Step 3: File Type Check
Allowed file types:
- Images: JPG, PNG, GIF
- Documents: PDF, DOC, DOCX, TXT
- Archives: ZIP
- Other: TXT, CSV

### Step 4: Size Verification
- Check file size (max usually 10MB)
- Compress if needed
- Split large files
- Optimize images

### Step 5: Attachment Upload
- Upload to ticket
- Place in storage
- Set permissions
- Configure access

### Step 6: Reference Linking
- Link to ticket
- Associate with reply
- Update ticket history
- Set visibility

### Step 7: Notification
- Notify recipient
- Provide file info
- Include download link
- Set expiration (if applicable)

### Step 8: Access Control
- Set download permissions
- Configure client access
- Enable/disable access
- Manage visibility

### Step 9: Removal (if needed)
- Remove attachments
- Update ticket
- Archive files
- Update records

### Step 10: Documentation
- Log attachment
- Record file info
- Document access
- Update audit trail

## Attachment Best Practices
- Use clear filenames
- Optimize file sizes
- Remove when not needed
- Scan for security

## Verification Checklist
- [ ] File validated
- [ ] Upload complete
- [ ] Linked to ticket
- [ ] Notified
- [ ] Documented

## Related Workflows
- whmcs-ticket-reply
- whmcs-client-file-upload
- whmcs-order-file