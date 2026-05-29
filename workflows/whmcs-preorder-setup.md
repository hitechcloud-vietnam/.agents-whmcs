# WHMCS Pre-order Setup Workflow

## Purpose
Configure pre-order functionality for products that are not yet available but can be ordered in advance of release.

## Prerequisites
- WHMCS installation
- Products planned for release
- Pre-order strategy defined

## Step-by-Step Process

### Step 1: Access Pre-order Configuration
1. Log into WHMCS admin
2. Navigate to `Configuration > Products/Services`
3. Select or create product for pre-order

### Step 2: Enable Pre-order Mode
1. Configure pre-order settings:
   - Enable pre-order capability
   - Set pre-order start date
   - Set availability date
   - Set pre-order end date
2. Configure pre-order visibility

### Step 3: Set Pre-order Pricing
1. Configure pricing:
   - Standard pricing
   - Pre-order early bird price
   - Deposit amount
   - Full payment option
   - Price after release
2. Set up pricing tiers:
   - Early bird discount
   - Pre-order bonus items
   - Release day pricing

### Step 4: Configure Pre-order Limits
1. Set quantity restrictions:
   - Maximum pre-orders allowed
   - Limit per customer
   - Limited edition quantity
   - Waitlist when limit reached
2. Configure allocation rules

### Step 5: Set Up Pre-order Display
1. Configure customer display:
   - Show "Pre-order Now" badge
   - Display release date
   - Show included extras
   - Display pre-order benefits
2. Set countdown timer

### Step 6: Configure Pre-order Payment
1. Set payment options:
   - Full payment upfront
   - Deposit to reserve
   - Payment in installments
   - Payment on release
2. Configure deposit handling

### Step 7: Set Pre-order Communications
1. Configure email notifications:
   - Pre-order confirmation
   - Payment received
   - Release date update
   - Shipping notification
   - Pre-order bonus information
2. Set up status update emails

### Step 8: Configure Release Processing
1. Set fulfillment rules:
   - Auto-process when released
   - Manual release processing
   - Staggered fulfillment
   - Priority processing for deposits
2. Configure release notifications

### Step 9: Set Up Post-release Changes
1. Configure transition:
   - Auto-convert to regular product
   - Update pricing on release
   - Change availability status
   - Disable pre-order option
2. Set release automation

### Step 10: Manage Pre-order Reporting
1. Set up tracking:
   - Pre-order volume
   - Conversion rate
   - Deposit collection
   - Cancellation rate
   - Revenue projection
2. Generate pre-order reports

## Verification Checklist
- [ ] Pre-order displays correctly
- [ ] Pre-orders place successfully
- [ ] Deposit payments process
- [ ] Release automation works
- [ ] Notifications send correctly

## Related Workflows
- whmcs-backorder-setup
- whmcs-stock-management
- whmcs-checkout-flow
- whmcs-promotional-banner

## Pre-order Best Practices
- Create urgency with early bird pricing
- Communicate release date clearly
- Offer incentives for pre-orders
- Keep customers updated on progress
- Process pre-orders promptly on release