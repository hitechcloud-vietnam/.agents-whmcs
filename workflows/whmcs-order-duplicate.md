# WHMCS Order Duplicate Workflow

## Purpose
Step-by-step guide for duplicating orders in WHMCS.

## Prerequisites
- Original order exists
- Duplicate reason identified
- Client account exists
- Authorization verified

## Workflow Steps

### Step 1: Duplicate Request
- Receive duplicate request
- Identify original order
- Verify requestor
- document duplicate reason

### Step 2: Original Order Review
- Review order details
- Identify items to duplicate
- Check configurations
- Assess duplicate needs

### Step 3: Item Selection
- Select items to duplicate
- Include/exclude addons
- Preserve configurations
- Set new quantities

### Step 4: Pricing Adjustment
- Apply current pricing
- Update for new cycle
- Apply any new discounts
- Calculate totals

### Step 5: New Order Creation
- Create new order
- Copy items from original
- Apply configurations
- Set appropriate status

### Step 6: Configuration Update
- Update dates
- Set new term
- Apply current settings
- Configure new billing

### Step 7: Invoice Generation
- Generate new invoice
- Apply current pricing
- Set due date
- Send to client

### Step 8: Payment Processing
- Process payment
- Handle payment issues
- Update invoice
- Record transaction

### Step 9: Notification
- Notify client
- Explain duplicate order
- Provide order details
- Include next steps

### Step 10: Documentation
- Log duplicate creation
- Link to original order
- Document changes
- Update audit trail

## Duplicate Scenarios
- Repeat orders
- Renewal orders
- Similar configurations
- Customer request

## Verification Checklist
- [ ] Original reviewed
- [ ] Items selected
- [ ] Order created
- [ ] Invoice generated
- [ ] Documented

## Related Workflows
- whmcs-order-merge
- whmcs-order-split
- whmcs-new-order-flow