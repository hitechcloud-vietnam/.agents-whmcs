# WHMCS Registrant Verification Workflow

## Purpose
Configure and manage domain registrant verification to ensure accurate contact information and comply with ICANN requirements.

## Prerequisites
- WHMCS installation
- Email verification system
- Verification policy defined

## Step-by-Step Process

### Step 1: Access Verification Settings
1. Log into WHMCS admin
2. Navigate to `Configuration > System > Registrant Verification`
3. Review verification options

### Step 2: Enable Verification
1. Configure verification:
   - Enable verification feature
   - Set verification requirements
   - Configure verification timing
   - Set up verification scope
2. Set global verification settings

### Step 3: Configure Verification Methods
1. Set up verification types:
   - Email verification
   - Phone verification (SMS)
   - Identity verification
   - Document verification
   - Annual verification
2. Set method priority

### Step 4: Configure Verification Workflow
1. Set up workflow:
   - Verification trigger (new registration, transfer, update)
   - Verification email/notification
   - Verification link/code
   - Verification deadline
   - Reminder notifications
   - Expiration handling
2. Set up step sequence

### Step 5: Configure Verification Requirements
1. Set requirements:
   - Required fields for verification
   - Registrant data accuracy check
   - Contact validation
   - Address verification
   - Organization verification
2. Set validation rules

### Step 6: Configure Verification Exemptions
1. Set exemptions:
   - Verified registrar accounts
   - Premium domains
   - Specific TLDs
   - Existing verified contacts
   - Bulk registration exemptions
2. Set exemption criteria

### Step 7: Configure Verification Notifications
1. Set up notifications:
   - Verification request email
   - Reminder emails
   - Deadline warning
   - Verification success
   - Verification failed
   - Action required alerts
2. Set notification timing

### Step 8: Configure Verification Actions
1. Set up actions:
   - Domain suspension on failure
   - Renewal denial
   - Transfer hold
   - Verification reminder
   - Admin notification
   - Auto-verification for updates
2. Set action timing

### Step 9: Configure Bulk Verification
1. Set up bulk process:
   - Batch verification
   - Verification campaign
   - Annual verification schedule
   - Verification report
   - Exception handling
2. Set up verification tracking

### Step 10: Monitor Verification Performance
1. Track metrics:
   - Verification success rate
   - Pending verifications
   - Failed verifications
   - Verification time
   - Customer compliance
2. Generate reports

## Verification Checklist
- [ ] Verification emails send
- [ ] Verification process works
- [ ] Reminders trigger correctly
- [ ] Actions enforce properly
- [ ] Reports accurate

## Related Workflows
- whmcs-domain-registration-flow
- whmcs-domain-transfer-flow
- whmcs-epp-code-request
- whmcs-whois-privacy-setup

## Registrant Verification Best Practices
- Make verification simple
- Send clear instructions
- Set reasonable deadlines
- Follow up on reminders
- Track compliance closely