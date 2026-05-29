# WHMCS Client Password Change Workflow

## Purpose
Step-by-step guide for changing client passwords in WHMCS.

## Prerequisites
- Client account exists
- Authentication verified
- Password reset requested
- Security protocol active

## Workflow Steps

### Step 1: Request Verification
- Identify password change request
- Verify requestor identity
- Check request method
- Document request details

### Step 2: Identity Verification
- For self-service: email link verification
- For admin: verify admin identity
- For support: verify client via questions
- Document verification method

### Step 3: Password Requirements Review
- Check password policy requirements
- Minimum length: 8 characters
- Require uppercase/lowercase
- Require numbers
- Require special characters

### Step 4: New Password Generation
- Generate strong password
- Apply policy requirements
- Avoid common patterns
- Ensure randomness

### Step 5: Password Update
- Apply new password to account
- Update password hash
- Clear any temp passwords
- Update session tokens

### Step 6: Two-Factor Update (if enabled)
- Preserve 2FA settings
- Update backup codes
- Send new verification codes
- Document 2FA status

### Step 7: Security Notifications
- Send password change notification
- Alert if suspicious activity
- Include change timestamp
- Provide security tips

### Step 8: Related System Updates
- Update SSO if applicable
- Sync with external auth
- Update API credentials
- Clear remembered devices

### Step 9: Client Confirmation
- Confirm password changed
- Provide login instructions
- Test access (optional)
- Share support contact

### Step 10: Documentation
- Log password change
- Record timestamp
- Document requestor
- Update audit trail

## Security Protocols
- Email verification required
- Admin verification for support
- Security questions
- IP validation

## Verification Checklist
- [ ] Identity verified
- [ ] Password changed
- [ ] Notifications sent
- [ ] Access confirmed
- [ ] Documented

## Related Workflows
- whmcs-client-login-reset
- whmcs-client-security
- whmcs-client-two-factor