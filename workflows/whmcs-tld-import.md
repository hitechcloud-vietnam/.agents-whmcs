# WHMCS TLD Import Workflow

## Purpose
Import Top Level Domains (TLDs) from domain registrars or bulk lists into WHMCS.

## Prerequisites
- WHMCS installation
- Registrar module configured
- TLD list to import

## Step-by-Step Process

### Step 1: Access TLD Import
1. Log into WHMCS admin
2. Navigate to `Configuration > Products/Services > Domain Pricing`
3. Click `Import TLDs` or `Bulk Import`

### Step 2: Prepare TLD List
1. Prepare import file:
   - CSV format: TLD, Registration, Transfer, Renewal
   - Excel format with columns
   - JSON format
   - Registry zone file format
2. Verify TLD format (.com, .net, etc.)

### Step 3: Select Import Method
1. Choose import option:
   - Bulk CSV/Excel import
   - Registrar sync import
   - Registry zone import
   - Manual entry
2. Set import source

### Step 4: Configure Import Settings
1. Set import parameters:
   - Default pricing markup
   - Billing term default
   - Auto-enable new TLDs
   - Currency selection
2. Set default configurations

### Step 5: Map Pricing Columns
1. Define column mapping:
   - TLD name column
   - Registration price column
   - Transfer price column
   - Renewal price column
   - Addon prices
2. Set default values for missing data

### Step 6: Run Import
1. Execute import:
   - Upload file
   - Preview import data
   - Confirm import
   - Monitor progress
2. Handle import errors

### Step 7: Verify Imported TLDs
1. Review imported data:
   - Check all TLDs imported
   - Verify pricing
   - Check enabled status
   - Review categorization
2. Correct any errors

### Step 8: Set TLD Configuration
1. Configure each TLD:
   - Enable/disable registration
   - Enable/disable transfer
   - Enable/disable renewal
   - Set group assignment
   - Configure DNS template
2. Set TLD-specific options

### Step 9: Configure TLD Groups
1. Organize TLDs:
   - Create TLD groups
   - Assign TLDs to groups
   - Set group display order
   - Create featured TLD group
2. Set group-based display rules

### Step 10: Test Domain Operations
1. Test each imported TLD:
   - Registration test
   - Transfer test
   - Renewal test
   - WHOIS lookup test
2. Verify pricing displays correctly

## Verification Checklist
- [ ] All TLDs imported successfully
- [ ] Pricing correct for each TLD
- [ ] Operations work (register, transfer, renew)
- [ ] TLD groups configured
- [ ] Display correct in order form

## Related Workflows
- whmcs-domain-pricing-setup
- whmcs-domain-registration-flow
- whmcs-domain-transfer-flow
- whmcs-nameserver-change

## TLD Import Best Practices
- Verify pricing before bulk import
- Test with single TLD first
- Keep backup of current pricing
- Review registry requirements per TLD
- Schedule regular sync updates