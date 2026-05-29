# WHMCS Client Address Update Workflow

## Purpose
Step-by-step guide for updating client addresses in WHMCS.

## Prerequisites
- Client account exists
- Current address on file
- New address provided
- Address verification capability

## Workflow Steps

### Step 1: Address Update Request
- Receive address update request
- Identify current address
- Collect new address details
- Document request reason

### Step 2: Address Component Collection
- Collect address line 1
- Collect address line 2
- Collect city
- Collect state/province
- Collect postal code
- Collect country

### Step 3: Address Validation
- Validate address format
- Run address verification
- Check for deliverability
- Suggest corrections

### Step 4: Address Standardization
- Format address properly
- Apply country standards
- Standardize abbreviations
- Ensure completeness

### Step 5: Verification Check
- Verify against databases
- Check for typos
- Validate postal code
- Confirm country

### Step 6: Address Update Execution
- Update address fields
- Sync billing address
- Sync service address
- Update all contact records

### Step 7: Tax Implications
- Review tax ID requirements
- Check VAT number validity
- Update tax settings
- Calculate tax impact

### Step 8: Invoice Review
- Review pending invoices
- Update invoice addresses
- Consider credit implications
- Document changes

### Step 9: Client Notification
- Confirm address updated
- Provide updated address
- Explain changes
- Include effective date

### Step 10: Documentation
- Log address update
- Record validation results
- Document reason
- Update audit trail

## Address Components
- Address Line 1 (street address)
- Address Line 2 (apt, suite, floor)
- City
- State/Province/Region
- Postal Code
- Country

## Address Validation Services
- Google Address Validation
- USPS Address Verification
- International address validation

## Verification Checklist
- [ ] Address validated
- [ ] Format standardized
- [ ] Updated in system
- [ ] Tax settings reviewed
- [ ] Documented

## Related Workflows
- whmcs-client-profile-edit
- whmcs-client-verification
- whmcs-order-invoice