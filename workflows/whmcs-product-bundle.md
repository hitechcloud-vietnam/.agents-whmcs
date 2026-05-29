# WHMCS Product Bundle Workflow

## Purpose
Create bundled product offerings that combine multiple products at a discounted rate.

## Prerequisites
- Existing products in WHMCS
- Pricing structure defined for bundle
- Bundle display strategy planned

## Step-by-Step Process

### Step 1: Access Bundle Configuration
1. Log into WHMCS admin
2. Navigate to `Configuration > Products/Services > Products`
3. Create a new product of type "Other" or select existing bundle product

### Step 2: Define Bundle Components
1. Create or select bundle product
2. Navigate to `Configurable Options` tab
3. Create a new configurable option group:
   - Name: "Bundle Selection"
   - Option Type: Dropdown

### Step 3: Add Bundle Options
1. Add individual bundle combinations as options:
   - Option 1: "Starter Bundle (Hosting + Email + Domain)"
   - Option 2: "Business Bundle (Hosting + Email + Domain + SSL)"
   - Option 3: "Enterprise Bundle (Full Suite)"
2. Set pricing for each bundle option

### Step 4: Configure Bundle Pricing
1. Pricing tab:
   - Set base bundle price
   - Configure setup fees
   - Enable multiple billing cycles
2. Set promotional pricing periods

### Step 5: Create Bundle Product Groups
1. Go to `Products > Product Groups`
2. Create bundle product group (e.g., "Value Bundles")
3. Add bundle products to group
4. Set display order and styling

### Step 6: Bundle-Specific Configurable Options
1. Create configurable option groups for each bundle variant:
   - Hosting tier selection
   - Email storage allocation
   - Domain extension choice
   - SSL certificate level
2. Link options to bundle product

### Step 7: Bundle Discount Configuration
1. Configure percentage discounts per bundle tier
2. Set absolute discount amounts
3. Enable free items in higher tiers

### Step 8: Order Form Bundle Display
1. Configure bundle display settings:
   - Show component savings
   - Display comparison pricing
   - Highlight best value bundle
2. Set bundle card styling

### Step 9: Automated Bundle Component Setup
1. Create product links for auto-provisioning
2. Configure automated service creation for each bundle component
3. Set dependency rules (domain requires hosting, etc.)

### Step 10: Bundle Testing
1. Test bundle purchase flow
2. Verify all components provision correctly
3. Confirm pricing calculations
4. Check invoice generation

## Verification Checklist
- [ ] Bundle displays correctly on order form
- [ ] All component products include in bundle
- [ ] Discount calculations accurate
- [ ] Invoice shows bundled pricing
- [ ] Provisioning triggers all components

## Related Workflows
- whmcs-product-setup
- whmcs-configurable-options
- whmcs-upsell-checkout

## Advanced Bundle Configurations
- Bundle upgrades/downgrades between tiers
- Prorated pricing for mid-cycle upgrades
- Bundle-only add-ons (premium support, etc.)