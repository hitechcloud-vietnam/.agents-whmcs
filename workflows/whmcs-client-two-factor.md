# WHMCS Client Two-Factor Authentication Workflow

## Purpose
Step-by-step guide for managing client two-factor authentication in WHMCS.

## Prerequisites
- Client account exists
- 2FA module configured
- Supported authenticator apps
- Client cooperation

## Workflow Steps

### Step 1: 2FA Enablement Request
- Receive 2FA enablement request
- Verify client identity
- Explain 2FA benefits
- Confirm intent

### Step 2: 2FA Method Selection
- Explain available methods:
  - Authenticator app (Google, Authy)
  - SMS verification
  - Email verification
  - Hardware key (YubiKey)
- Client selects method

### Step 3: Setup Preparation
- Generate setup data
- Create QR code (for app)
- Prepare verification codes
- Document setup process

### Step 4: Authenticator App Setup
- Generate secret key
- Create QR code
- Client scans with app
- Verify initial code

### Step 5: SMS/Email Setup
- Collect phone number/email
- Send verification code
- Client enters code
- Verify and confirm

### Step 6: Verification Process
- Verify first code entry
- Confirm 2FA working
- Generate backup codes
- Store backup codes securely

### Step 7: Backup Codes Generation
- Generate backup codes (usually 10)
- Set code usage limit
- Provide codes to client
- Explain backup usage

### Step 8: Configuration Save
- Save 2FA configuration
- Link to client account
- Enable 2FA enforcement
- Update security status

### Step 9: Client Training
- Train client on 2FA use
- Explain backup codes
- Provide recovery options
- Share troubleshooting tips

### Step 10: Documentation and Monitoring
- Log 2FA enablement
- Record 2FA method
- Set up monitoring
- Schedule regular review

## 2FA Methods
- TOTP (Time-based One-Time Password)
- SMS (text message codes)
- Email codes
- Hardware security keys

## Backup Code Policy
- Generate 10 backup codes
- Single-use codes
- Regenerate when depleted
- Secure storage required

## Verification Checklist
- [ ] Method selected
- [ ] Setup completed
- [ ] Verified working
- [ ] Backup codes generated
- [ ] Client trained
- [ ] Documented

## Related Workflows
- whmcs-client-security
- whmcs-client-login-reset
- whmcs-client-password-change