# WHMCS Coupon Creation Workflow

## Purpose
Create and manage coupon codes for discounts, promotions, and special offers.

## Prerequisites
- WHMCS installation
- Products/discounts configured
- Coupon strategy planned

## Step-by-Step Process

### Step 1: Access Coupon Configuration
1. Log into WHMCS admin
2. Navigate to `Configuration > Orders > Promotions & Discounts`
3. Click `Create New Promotion` or `Add Coupon`

### Step 2: Configure Basic Coupon Details
1. Enter coupon information:
   - Coupon code (e.g., SAVE20)
   - Coupon name for admin reference
   - Description/purpose
   - Enable/disable status

### Step 3: Set Discount Type
1. Choose discount type:
   - Percentage discount (e.g., 20% off)
   - Fixed amount discount (e.g., $10 off)
   - Free setup
   - Free product
   - Free shipping
2. Set discount value

### Step 4: Define Applicability
1. Configure what coupon applies to:
   - All products
   - Specific products
   - Product categories
   - Specific services
2. Set exclusions

### Step 5: Set Usage Limits
1. Configure usage restrictions:
   - Maximum total uses
   - Maximum uses per client
   - One use per client
   - Unlimited uses
2. Set current usage tracking

### Step 6: Configure Date/Time Restrictions
1. Set validity period:
   - Start date and time
   - End date and time
   - Specific days of week
   - Time of day restrictions
2. Set recurring validity (e.g., first 3 days of each month)

### Step 7: Set Client Eligibility
1. Configure client restrictions:
   - All clients
   - Specific client IDs
   - Client groups
   - New clients only
   - Existing clients only
2. Set minimum order requirements

### Step 8: Configure Stacking Rules
1. Set combination rules:
   - Can combine with other coupons
   - Cannot combine with other offers
   - Exclusive coupon
2. Set priority when multiple apply

### Step 9: Set Up Auto-Apply Rules
1. Configure automatic application:
   - Apply automatically to eligible orders
   - Require code entry
   - Show available auto-coupons
2. Set auto-coupon eligibility rules

### Step 10: Create Coupon Campaigns
1. Organize coupons into campaigns:
   - Email campaign codes
   - Social media codes
   - Partner/reseller codes
2. Track campaign performance
3. Generate unique codes for tracking

## Verification Checklist
- [ ] Coupon code validates correctly
- [ ] Discount applies to correct products
- [ ] Usage limits enforce properly
- [ ] Expiration dates work
- [ ] Client restrictions apply correctly

## Related Workflows
- whmcs-discount-rules
- whmcs-promotional-banner
- whmcs-abandoned-cart-recovery
- whmcs-custom-pricing

## Coupon Best Practices
- Use unique codes for tracking
- Set reasonable expiration dates
- Monitor usage to prevent abuse
- Test codes before sending to customers
- Track ROI per coupon campaign