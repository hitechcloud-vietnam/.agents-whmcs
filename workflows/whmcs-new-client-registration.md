# WHMCS New Client Registration Workflow

## Purpose
Step-by-step guide for processing new client registrations in WHMCS.

## Prerequisites
- WHMCS registration enabled
- Product/service selected
- Payment method configured
- Required information collected

## Workflow Steps

### Step 1: Registration Initiation
- Client starts registration
- Selects desired product/service
- Begins checkout process
- System creates prospect record

### Step 2: Personal Information
- Collect first/last name
- Gather company name (if applicable)
- Collect email address
- Verify email format

### Step 3: Contact Information
- Collect phone number
- Gather address details
- Collect city/state/province
- Collect postal code
- Collect country

### Step 4: Account Security
- Set password
- Configure security question
- Set email preferences
- Accept terms of service

### Step 5: Client Account Creation
- Create WHMCS client account
- Generate client ID
- Set account status
- Configure email preferences

### Step 6: Identity Verification
- Send verification email
- Verify phone if required
- Check for duplicate accounts
- Apply verification status

### Step 7: Order Processing
- Generate order
- Apply pricing
- Calculate totals
- Create pending order

### Step 8: Payment Processing
- Collect payment details
- Process initial payment
- Apply promo codes
- Generate invoice

### Step 9: Welcome Communication
- Send welcome email
- Include account details
- Provide login instructions
- Share support information

### Step 10: Onboarding Follow-up
- Send onboarding guide
- Schedule follow-up
- Create welcome ticket
- Document registration

## Registration Fields
- First name, Last name
- Email address
- Phone number
- Address details
- Company name
- VAT number (EU)

## Verification Checklist
- [ ] All fields completed
- [ ] Email verified
- [ ] Account created
- [ ] Order processed
- [ ] Welcome sent
- [ ] Onboarding complete

## Related Workflows
- whmcs-client-onboarding
- whmcs-client-verification
- whmcs-new-order-flow