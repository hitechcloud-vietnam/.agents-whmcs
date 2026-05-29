# WHMCS Email Template Export Workflow

## Purpose
Export email templates for backup, migration, or sharing.

## Export Methods

### Method 1: Single Template Export
1. Navigate to: Configuration > System > Email Templates
2. Find template to export
3. Click template name
4. Click "Export" button
5. Download .txt or .html file

### Method 2: Bulk Export
1. Navigate to: Configuration > System > Email Templates
2. Select multiple templates (checkbox)
3. Click "Bulk Actions"
4. Select "Export Selected"
5. Download ZIP file

### Method 3: Full System Export
1. Navigate to: Configuration > System > Email Templates
2. Click "Export All"
3. Download complete backup

## Export File Formats
- .txt (Plain text)
- .html (HTML content)
- .json (Metadata + content)
- .zip (Multiple files)

## Export Contents
```
Template Export:
├── subject
├── body
├── plain_text
├── attachments
├── variables
├── conditions
└── metadata
```

## Use Cases

### Backup
1. Regular template backups
2. Before major updates
3. Document current state

### Migration
1. Export from staging
2. Import to production
3. Move between WHMCS instances

### Version Control
1. Export templates to Git
2. Track changes
3. Collaborate with team

## Related Workflows
- whmcs-email-template-import
- whmcs-email-template-clone