# WHMCS New Order Flow Workflow

## Purpose
Step-by-step guide for processing new orders in WHMCS.

## Prerequisites
- WHMCS cart configured
- Products available
- Payment gateways configured
- Client account or guest checkout

## Workflow Steps

### Step 1: Order Initiation
- Customer selects product(s)
- Add to cart
- Review cart contents
- Proceed to checkout

### Step 2: Account Selection
- Existing client login
- New client registration
- Guest checkout
- Create new account

### Step 3: Client Information
- Collect/verify personal info
- Collect address details
- Set account preferences
- Configure notifications

### Step 4: Domain Selection (if applicable)
- Register new domain
- Transfer domain
- Existing domain
- Skip domain step

### Step 5: Configuration Options
- Select product options
- Configure addons
- Set custom values
- Apply configurations

### Step 6: Billing Cycle Selection
- Select billing term
- Monthly/Quarterly/Annual
- Calculate pricing
- Show savings

### Step 7: Payment Method Selection
- Select payment gateway
- Enter payment details
- Apply promo codes
- View totals

### Step 8: Order Review
- Review all order details
- Verify pricing
- Accept terms
- Confirm order

### Step 9: Payment Processing
- Process payment
- Verify transaction
- Handle payment errors
- Confirm payment

### Step 10: Order Completion
- Generate order
- Create services
- Send confirmation
- Trigger provisioning

## Order Flow States
- Pending (awaiting payment)
- Pending (manual review)
- Active (paid)
- Cancelled
- Fraud

## Verification Checklist
- [ ] Order placed
- [ ] Payment processed
- [ ] Order created
- [ ] Services provisioned
- [ ] Confirmation sent

## Related Workflows
- whmcs-order-payment
- whmcs-order-fulfillment
- whmcs-new-service-provisioning