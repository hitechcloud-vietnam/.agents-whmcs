# WHMCS Client Email Change Workflow

## Purpose
Step-by-step guide for changing client email addresses in WHMCS.

## Prerequisites
- Client account exists
- Current email verified
- New email provided
- Change authority confirmed

## Workflow Steps

### Step 1: Email Change Request
- Receive email change request
- Identify current email
- Collect new email
- Document request details

### Step 2: Current Email Verification
- Send verification to current email
- Confirm access to current email
- Verify client identity
- Document verification

### Step 3: New Email Validation
- Verify email format
- Check email deliverability
- Validate email domain
- Detect disposable emails

### Step 4: Duplicate Check
- Check for existing account
- Verify no duplicate accounts
- Check for pending registrations
- Flag conflicts if found

### Step 5: Verification Email
- Send verification to new email
- Include verification link
- Set expiration time
- Document sent status

### Step 6: Email Verification
- Client clicks verification link
- Verify token validity
- Complete verification
- Update verification status

### Step 7: Email Update Execution
- Update primary email
- Update billing email (if applicable)
- Sync all email fields
- Update notification preferences

### Step 8: Service Impact Check
- Review services linked to old email
- Update service notifications
- Check API integrations
- Update ticket notifications

### Step 9: Notifications
- Send confirmation to new email
- Alert about email change
- Include security notice
- Provide support contact

### Step 10: Documentation
- Log email change
- Record verification method
- Document date/time
- Update audit trail

## Email Change Security
- Verify old email access
- Verify new email
- Check for fraud indicators
- Require admin approval for high-risk

## Verification Checklist
- [ ] Current email verified
- [ ] New email validated
- [ ] Verification complete
- [ ] Email updated
- [ ] Services synced
- [ ] Documented

## Related Workflows
- whmcs-client-profile-edit
- whmcs-client-password-change
- whmcs-client-security