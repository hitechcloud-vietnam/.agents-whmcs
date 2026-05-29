# WHMCS Custom Pricing Workflow

## Purpose
Implement custom pricing rules including tiered pricing, volume discounts, customer-specific pricing, and promotional pricing.

## Prerequisites
- Products created
- Pricing structure planned
- Customer segments identified
- Understanding of pricing rules needed

## Step-by-Step Process

### Step 1: Access Pricing Rules
1. Log into WHMCS admin
2. Navigate to `Configuration > Products/Services > Products`
3. Select product and go to `Pricing` tab

### Step 2: Configure Tiered Pricing
1. Enable tiered pricing feature
2. Add pricing tiers:
   - Tier 1: 1-10 units (base price)
   - Tier 2: 11-50 units (5% discount)
   - Tier 3: 51-100 units (10% discount)
   - Tier 4: 100+ units (15% discount)
3. Set minimum quantity for tier activation

### Step 3: Set Up Volume Discounts
1. Configure volume discount rules:
   - Discount percentage per tier
   - Flat discount amounts
   - Maximum discount caps
2. Set discount display text

### Step 4: Customer-Specific Pricing
1. Navigate to `Clients > Clients`
2. Select specific client
3. Go to `Products > Pricing` tab
4. Override pricing for specific products

### Step 5: Group-Based Pricing
1. Create client groups with pricing tiers:
   - Gold clients (15% discount)
   - Silver clients (10% discount)
   - Bronze clients (5% discount)
2. Configure group pricing rules

### Step 6: Promotional Pricing
1. Set up promotional pricing:
   - Start/end date range
   - Promotional price amount
   - Original price display
2. Configure auto-expiration

### Step 7: Quantity-Based Pricing
1. Enable quantity-based pricing in product
2. Configure pricing matrix:
   - Units | Monthly | Annual
   - 1 | $10 | $100
   - 5 | $9 | $90
   - 10 | $8 | $80
3. Set minimum quantity

### Step 8: Currency-Specific Pricing
1. Configure pricing per currency:
   - USD base pricing
   - EUR conversion with margin
   - GBP pricing adjustment
2. Set automatic currency conversion

### Step 9: Affiliate Pricing Rules
1. Configure affiliate-specific pricing:
   - Affiliate partner discounts
   - Reseller pricing tiers
   - Commission-based pricing

### Step 10: Pricing Rule Automation
1. Set up automatic pricing updates:
   - Date-based pricing changes
   - Inventory-based pricing
   - Competitor price matching
2. Configure notification triggers

## Verification Checklist
- [ ] Tiered pricing calculates correctly
- [ ] Volume discounts apply at threshold
- [ ] Customer-specific pricing overrides display
- [ ] Group pricing applies to all members
- [ ] Promotional pricing expires correctly

## Related Workflows
- whmcs-product-setup
- whmcs-discount-rules
- whmcs-coupon-creation

## Advanced Pricing Configurations
- Time-based pricing (day/night rates)
- Seasonal pricing adjustments
- Geographic pricing zones
- Bundle pricing calculations