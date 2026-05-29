# WHMCS Domain Auto-Renewal Setup Workflow

## Purpose
Configure automatic domain renewal to prevent expiration and maintain continuous service.

## Prerequisites
- WHMCS installation
- Domain registrar module configured
- Payment methods configured
- Domain pricing set

## Step-by-Step Process

### Step 1: Access Renewal Settings
1. Log into WHMCS admin
2. Navigate to `Configuration > System > Automation Settings`
3. Locate domain renewal settings

### Step 2: Enable Auto-Renewal
1. Configure global setting:
   - Enable auto-renewal toggle
   - Default for new domains
   - Existing domains opt-in
2. Set renewal behavior rules

### Step 3: Configure Renewal Timing
1. Set renewal schedule:
   - Days before expiration to renew
   - Number of renewal attempts
   - Retry interval
   - Final reminder before renew
2. Set grace period handling

### Step 4: Configure Renewal Billing
1. Set billing parameters:
   - Payment method selection
   - Charge method (credit card, PayPal)
   - Backup payment method
   - Billing threshold (max amount)
   - Charge attempt schedule
2. Set billing failure handling

### Step 5: Set Renewal Notifications
1. Configure notifications:
   - Renewal reminder (30 days)
   - Renewal reminder (14 days)
   - Renewal reminder (7 days)
   - Auto-renewal confirmation
   - Renewal failed notification
2. Set notification templates

### Step 6: Configure Domain-Specific Rules
1. Set per-domain options:
   - Enable/disable per domain
   - Renewal years selection
   - Registrar-specific rules
   - TLD-specific rules
   - Premium domain handling
2. Set bulk renewal options

### Step 7: Configure Renewal Exclusions
1. Set exclusion rules:
   - Exclude transferred domains (30-day rule)
   - Exclude recently registered domains
   - Manual-only domains
   - Suspended domains
   - Cancelled domains
2. Set up exemption list

### Step 8: Configure Registrar Integration
1. Set registrar automation:
   - Auto-renew via registrar API
   - Registrar-specific methods
   - Sync renewal status
   - Handle registrar errors
   - Fallback procedures

### Step 9: Set Up Post-Renewal
1. Configure follow-up:
   - Renewal confirmation email
   - Updated expiry date
   - Invoice generation
   - Service continuation
   - DNS continuity
2. Set up renewal records

### Step 10: Monitor Renewal Performance
1. Set up tracking:
   - Renewal success rate
   - Failed renewals
   - Expired domains
   - Revenue from renewals
   - Cost analysis
2. Generate renewal reports

## Verification Checklist
- [ ] Auto-renewal triggers correctly
- [ ] Payments process successfully
- [ ] Notifications send
- [ ] Expiry extended properly
- [ ] Exclusions work correctly

## Related Workflows
- whmcs-domain-pricing-setup
- whmcs-domain-sync-automation
- whmcs-domain-transfer-flow
- whmcs-domain-registration-flow

## Auto-Renewal Best Practices
- Set renewal 7-14 days before expiry
- Use reliable payment method
- Send multiple reminders
- Monitor failed renewals closely
- Keep sufficient account balance