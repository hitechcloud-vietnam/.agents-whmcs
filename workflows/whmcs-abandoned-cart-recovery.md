# WHMCS Abandoned Cart Recovery Workflow

## Purpose
Implement automated cart recovery to re-engage customers who left checkout without completing their purchase.

## Prerequisites
- WHMCS installation with email configuration
- Cron jobs configured
- Email templates available

## Step-by-Step Process

### Step 1: Access Cart Recovery Settings
1. Log into WHMCS admin
2. Navigate to `Configuration > System > Automation Settings`
3. Locate abandoned cart settings

### Step 2: Enable Cart Recovery
1. Turn on abandoned cart recovery:
   - Enable automated emails
   - Set trigger conditions
   - Configure timing
2. Set recovery percentage goal

### Step 3: Configure Recovery Timing
1. Set email sequence:
   - Email 1: 1 hour after abandonment
   - Email 2: 24 hours after abandonment
   - Email 3: 72 hours after abandonment
2. Set day/time preferences
3. Configure timezone handling

### Step 4: Create Recovery Email Templates
1. Access `Configuration > Emails > Email Templates`
2. Create abandoned cart emails:
   - Subject line optimization
   - Email body with cart contents
   - Clear call-to-action button
   - Include discount code option
3. Personalize with customer name
4. Add urgency elements

### Step 5: Set Up Discount Offers
1. Configure recovery discounts:
   - Percentage off next order
   - Fixed amount discount
   - Free shipping offer
   - No incentive (just reminder)
2. Create unique coupon codes for tracking
3. Set discount expiration dates

### Step 6: Define Recovery Rules
1. Set cart value thresholds:
   - Minimum cart value for recovery
   - Maximum discount value
   - Product exclusions
2. Configure customer eligibility:
   - New customers only
   - All customers
   - Exclude specific groups

### Step 7: Implement Tracking
1. Set up analytics:
   - Email open tracking
   - Click tracking
   - Conversion tracking
   - Revenue attribution
2. Create recovery reports

### Step 8: Configure Cart State Detection
1. Set detection rules:
   - Cart created but not ordered
   - Checkout started but not completed
   - Payment failed
   - Session timeout
2. Exclude test carts

### Step 9: Set Up Multi-Channel Recovery
1. Configure additional channels:
   - Email recovery (primary)
   - SMS recovery option
   - Browser push notifications
2. Coordinate messaging across channels

### Step 10: Optimize Recovery Performance
1. Review recovery metrics:
   - Recovery rate
   - Revenue recovered
   - Discount cost vs. revenue
   - Email engagement rates
2. Test different email content
3. Adjust timing and offers
4. A/B test subject lines

## Verification Checklist
- [ ] Abandoned carts detected correctly
- [ ] Recovery emails send at scheduled times
- [ ] Email content displays properly
- [ ] Discount codes apply correctly
- [ ] Recovery conversion tracked

## Related Workflows
- whmcs-coupon-creation
- whmcs-checkout-flow
- whmcs-discount-rules
- whmcs-promotional-banner

## Cart Recovery Best Practices
- Send first email quickly (within 1 hour)
- Include clear CTA to return to cart
- Add discount to increase conversion
- Test different email sequences
- Track and optimize continuously