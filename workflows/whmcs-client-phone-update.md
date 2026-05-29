# WHMCS Client Phone Update Workflow

## Purpose
Step-by-step guide for updating client phone numbers in WHMCS.

## Prerequisites
- Client account exists
- Current phone on file
- New phone number provided
- Verification capability

## Workflow Steps

### Step 1: Phone Update Request
- Receive phone update request
- Identify current phone
- Collect new phone number
- Document request reason

### Step 2: Phone Format Validation
- Validate phone format
- Check country code
- Verify length
- Format consistently

### Step 3: Phone Number Verification
- Select verification method
- Send SMS verification
- Generate verification code
- Document verification sent

### Step 4: Verification Completion
- Receive verification code
- Validate code entry
- Confirm phone ownership
- Update verification status

### Step 5: Duplicate Check
- Check for duplicate numbers
- Verify no conflicting accounts
- Flag if issues found
- Document check results

### Step 6: Phone Update Execution
- Update phone number field
- Update secondary phone (if needed)
- Sync across all fields
- Update notification settings

### Step 7: Two-Factor Authentication Update
- Update 2FA phone if configured
- Send new verification codes
- Update backup codes
- Document 2FA status

### Step 8: Service Notifications
- Update SMS notifications
- Review notification preferences
- Update voice notifications
- Sync with support integration

### Step 9: Client Notification
- Confirm phone updated
- Verify receiving texts
- Update notification preferences
- Provide security tips

### Step 10: Documentation
- Log phone update
- Record verification method
- Document timestamp
- Update client notes

## Verification Methods
- SMS code verification
- Voice call verification
- WhatsApp verification
- Admin verification

## Phone Format Standards
- Include country code
- Use E.164 format
- Remove formatting characters
- Store consistently

## Verification Checklist
- [ ] Format validated
- [ ] Verification completed
- [ ] Phone updated
- [ ] 2FA synced (if applicable)
- [ ] Documented

## Related Workflows
- whmcs-client-profile-edit
- whmcs-client-two-factor
- whmcs-client-verification