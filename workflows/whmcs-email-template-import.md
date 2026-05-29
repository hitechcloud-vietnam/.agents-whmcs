# WHMCS Email Template Import Workflow

## Purpose
Import email templates from backup or external sources.

## Import Methods

### Method 1: Single Template Import
1. Navigate to: Configuration > System > Email Templates
2. Click "Import Template"
3. Upload file (.txt, .html)
4. Map fields (subject, body)
5. Save template

### Method 2: Bulk Import
1. Prepare ZIP file with templates
2. Navigate to: Configuration > System > Email Templates
3. Click "Import Templates"
4. Upload ZIP file
5. Review and confirm

### Method 3: Template Migration
1. Export from source WHMCS
2. Download export file
3. Import to target WHMCS
4. Map to correct trigger events

## Import Process

### Step 1: Prepare Files
1. Ensure proper format (.txt, .html)
2. Validate template syntax
3. Check for required fields
4. Include metadata if available

### Step 2: Upload and Map
1. Select import file
2. Map variables to fields:
   - Subject line
   - Email body
   - Plain text version
   - Attachments
3. Review mapping

### Step 3: Configure Template
1. Set template name
2. Assign trigger event
3. Set status (active/inactive)
4. Configure conditions

### Step 4: Test Import
1. Send test email
2. Verify variables
3. Check formatting

## Import Validation
- Valid HTML structure
- No broken variables
- Proper encoding (UTF-8)
- Reasonable file sizes

## Troubleshooting
- Failed import: Check file format
- Missing variables: Update syntax
- Encoding issues: Convert to UTF-8

## Related Workflows
- whmcs-email-template-export
- whmcs-email-template-create