# WHMCS Discount Rules Workflow

## Purpose
Create and manage discount rules for products, categories, customers, and time-based promotions.

## Prerequisites
- WHMCS installation
- Products/services configured
- Discount strategy planned

## Step-by-Step Process

### Step 1: Access Discount Configuration
1. Log into WHMCS admin
2. Navigate to `Configuration > Orders > Promotions & Discounts`
3. Review available discount types

### Step 2: Create Percentage Discounts
1. Add new discount rule:
   - Rule name
   - Discount type: Percentage
   - Discount value (e.g., 20%)
2. Set applicable scope:
   - All products
   - Specific products
   - Product categories
   - Specific clients/groups

### Step 3: Create Fixed Amount Discounts
1. Configure flat discounts:
   - Rule name
   - Discount type: Fixed Amount
   - Discount value (e.g., $10)
2. Set applicable products/services

### Step 4: Configure Quantity Discounts
1. Set up volume-based pricing:
   - Minimum quantity threshold
   - Discount per unit
   - Tiered pricing levels
2. Configure maximum quantity limits

### Step 5: Set Up Time-Based Discounts
1. Configure promotional periods:
   - Start date/time
   - End date/time
   - Recurring schedule (e.g., weekly)
2. Set discount duration:
   - First term only
   - All terms during promotion
   - Limited number of uses

### Step 6: Configure Customer-Specific Discounts
1. Set client group discounts:
   - VIP client pricing
   - Loyalty discounts
   - New customer incentives
2. Configure individual client discounts

### Step 7: Create Category Discounts
1. Set category-wide discounts:
   - All products in category
   - Specific subcategory
   - Combined category discounts
2. Configure category exclusions

### Step 8: Set Discount Conditions
1. Configure eligibility rules:
   - First order only
   - Minimum order amount
   - Specific payment methods
   - Geographic restrictions
2. Set combination rules:
   - Can combine with other discounts
   - Exclusive discounts
   - Maximum discount cap

### Step 9: Configure Auto-Application
1. Set automatic discount application:
   - Automatic vs. code-required
   - Priority order when multiple apply
   - Best price selection
2. Configure display of applied discounts

### Step 10: Monitor Discount Performance
1. Track discount usage:
   - Usage count
   - Revenue impact
   - Conversion lift
2. Generate discount reports
3. Adjust rules based on results

## Verification Checklist
- [ ] Discounts apply to correct products
- [ ] Percentage/fixed amounts calculate correctly
- [ ] Time-based discounts activate/deactivate properly
- [ ] Customer-specific discounts apply to right clients
- [ ] Discount reports accurate

## Related Workflows
- whmcs-coupon-creation
- whmcs-custom-pricing
- whmcs-promotional-banner
- whmcs-abandoned-cart-recovery

## Discount Strategy Tips
- Set clear discount limits
- Monitor profit margin impact
- Use time-limited offers strategically
- Track effectiveness regularly