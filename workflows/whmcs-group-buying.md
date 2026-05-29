# WHMCS Group Buying Workflow

## Purpose
Implement group buying/collective purchasing functionality where customers can join groups to unlock discounted pricing.

## Prerequisites
- WHMCS installation
- Products for group buying
- Group buying strategy defined

## Step-by-Step Process

### Step 1: Access Group Buying Settings
1. Log into WHMCS admin
2. Navigate to `Configuration > Products/Services`
3. Select product for group buying

### Step 2: Enable Group Buying
1. Configure group buying:
   - Enable group buy option
   - Set minimum group size
   - Set maximum group size
   - Set time limit for group
2. Configure group pricing

### Step 3: Configure Group Pricing
1. Set up pricing tiers:
   - Single price (no discount)
   - Group price (discounted)
   - Tiered pricing based on group size
   - Maximum discount level
2. Set price display:
   - Show original vs. group price
   - Display savings amount/percentage
   - Show tier progress

### Step 4: Set Group Size Requirements
1. Configure group rules:
   - Minimum participants to proceed
   - Maximum participants allowed
   - Auto-fill minimum requirement
   - Group leader benefits
2. Set group formation rules

### Step 5: Configure Group Duration
1. Set time limits:
   - Group duration (24h, 48h, 72h, etc.)
   - Extend time if close to goal
   - End early if maximum reached
   - Time extension limits
2. Set countdown display

### Step 6: Set Up Group Management
1. Configure group handling:
   - Allow creating new groups
   - Allow joining existing groups
   - Set group creator privileges
   - Configure group status display
2. Set up group sharing options

### Step 7: Configure Payment Processing
1. Set payment rules:
   - Payment on group completion
   - Payment upfront with refund if fail
   - Deposit to join group
   - Installment payments
2. Configure refund handling

### Step 8: Configure Group Completion
1. Set completion rules:
   - Auto-complete when full
   - Require manual confirmation
   - Partial fulfillment option
   - Group cancellation handling
2. Set notification on completion

### Step 9: Set Up Notifications
1. Configure alerts:
   - Group created notification
   - New member joined notification
   - Group nearly full alert
   - Group completed notification
   - Group failed notification
2. Set reminder emails

### Step 10: Track Group Buying Performance
1. Set up reporting:
   - Groups created
   - Conversion rate
   - Average group size
   - Revenue from group buying
2. Optimize group buying settings

## Verification Checklist
- [ ] Group buying option displays correctly
- [ ] Groups form and fill correctly
- [ ] Pricing applies at group size
- [ ] Notifications send properly
- [ ] Completion triggers correctly

## Related Workflows
- whmcs-product-setup
- whmcs-promotional-banner
- whmcs-checkout-flow
- whmcs-discount-rules

## Group Buying Best Practices
- Set achievable group sizes
- Give clear time limits
- Offer meaningful discounts
- Send progress updates
- Make sharing easy