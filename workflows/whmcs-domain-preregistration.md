# WHMCS Domain Preregistration Workflow

## Purpose
Configure and manage domain preregistration for domains that will become available in the future.

## Prerequisites
- WHMCS installation
- Registrar module with preregistration support
- Upcoming TLD launches or expiring domains identified

## Step-by-Step Process

### Step 1: Access Preregistration Settings
1. Log into WHMCS admin
2. Navigate to `Configuration > Products/Services > Domain Pricing`
3. Locate preregistration options

### Step 2: Enable Preregistration
1. Configure preregistration:
   - Enable preregistration feature
   - Set available preregistration TLDs
   - Configure preregistration limits
2. Set up preregistration campaigns

### Step 3: Configure Preregistration Details
1. Set up preregistration info:
   - Target launch date
   - Expected availability time
   - Sunrise period details
   - Landrush period details
   - General availability date
2. Set display countdown

### Step 4: Set Preregistration Pricing
1. Configure pricing:
   - Preregistration fee
   - Registration fee on launch
   - Success fee (if available)
   - Full payment vs. deposit
   - Refund policy
2. Set pricing display

### Step 5: Configure Preregistration Flow
1. Set up customer flow:
   - Search for preregistrable domains
   - Add to cart with deposit
   - Payment capture
   - Confirmation display
   - Queue position notification
2. Set up status tracking

### Step 6: Configure Priority Rules
1. Set registration priority:
   - First-come-first-served
   - Sunrise period for trademarks
   - Landrush auction
   - Premium allocation
   - Registrar lottery
2. Set up priority notifications

### Step 7: Set Up Queue Management
1. Configure queue handling:
   - Queue position display
   - Queue notification updates
   - Queue timeout handling
   - Queue cancellation policy
   - Second chance offers
2. Set queue communication

### Step 8: Configure Launch Processing
1. Set up processing:
   - Automated registration attempt
   - Registrar API integration
   - Bulk registration handling
   - Success/failure notification
   - Refund on failure
2. Set success criteria

### Step 9: Set Up Notifications
1. Configure alerts:
   - Preregistration confirmation
   - Launch approaching notification
   - Registration success notification
   - Registration failed notification
   - Refund processed notification
2. Set notification timing

### Step 10: Monitor and Report
1. Track performance:
   - Preregistrations received
   - Success rate
   - Revenue collected
   - Customer satisfaction
   - Issues encountered
2. Generate preregistration reports

## Verification Checklist
- [ ] Preregistration displays correctly
- [ ] Payment processes
- [ ] Queue position shown
- [ ] Launch registration works
- [ ] Notifications send

## Related Workflows
- whmcs-domain-registration-flow
- whmcs-domain-auction-flow
- whmcs-domain-drop-catch
- whmcs-domain-pricing-setup

## Preregistration Best Practices
- Communicate clearly on success chances
- Set proper expectations on timing
- Process registrations promptly on launch
- Handle failures gracefully
- Provide regular status updates