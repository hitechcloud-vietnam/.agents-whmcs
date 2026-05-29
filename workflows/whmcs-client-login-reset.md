# WHMCS Client Login Reset Workflow

## Purpose
Step-by-step guide for resetting client login access in WHMCS.

## Prerequisites
- Client account exists
- Identity verified
- Reset request authorized
- Communication channel available

## Workflow Steps

### Step 1: Reset Request Received
- Receive login reset request
- Identify client account
- Document request reason
- Verify requester identity

### Step 2: Identity Verification
- Verify client identity
- Check verification method
- Confirm account ownership
- Document verification

### Step 3: Verification Method Selection
- Email link reset
- Security questions
- Admin verification
- Phone verification

### Step 4: Password Reset
- Generate reset link
- Send to verified email
- Set link expiration
- Document link sent

### Step 5: Email Delivery
- Send reset email
- Verify email deliverability
- Check spam folder
- Monitor for issues

### Step 6: Reset Link Expiration
- Set expiration (usually 1 hour)
- Track link usage
- Invalidate after use
- Handle expired links

### Step 7: New Password Setup
- Client sets new password
- Enforce password policy
- Confirm password
- Apply password change

### Step 8: Session Invalidation
- Invalidate existing sessions
- Clear remembered logins
- Reset security tokens
- Force re-authentication

### Step 9: Two-Factor Reset (if enabled)
- Verify 2FA reset request
- Disable 2FA temporarily
- Require re-setup
- Document 2FA change

### Step 10: Confirmation and Documentation
- Confirm login reset complete
- Send confirmation email
- Log reset event
- Update security audit

## Security Protocols
- Require identity verification
- Use secure reset channels
- Log all reset attempts
- Monitor for abuse

## Verification Checklist
- [ ] Identity verified
- [ ] Reset processed
- [ ] New password set
- [ ] Sessions cleared
- [ ] Confirmation sent
- [ ] Documented

## Related Workflows
- whmcs-client-password-change
- whmcs-client-security
- whmcs-client-two-factor