# WHMCS Addon Product Workflow

## Purpose
Create and manage addon products that complement main products (upgrades, extras, add-ons).

## Prerequisites
- Main products created
- Addon pricing structure planned
- Product relationships defined

## Step-by-Step Process

### Step 1: Access Addon Configuration
1. Log into WHMCS admin panel
2. Navigate to `Configuration > Products/Services > Addon Modules`
3. Click `Create New Addon`

### Step 2: Basic Addon Information
1. Enter addon name (e.g., "Extra Storage 10GB")
2. Enter description and long description
3. Select module if applicable (for automated addons)
4. Set billing type (recurring, one-time, or both)

### Step 3: Pricing Configuration
1. Set pricing for each billing cycle:
   - Monthly price
   - Quarterly price
   - Annual price
2. Configure setup fee
3. Set pricing per billing cycle options

### Step 4: Product Relationships
1. Define which products this addon applies to:
   - Link to specific products
   - Link to product groups
   - Set as available with all products
2. Configure compatibility rules

### Step 5: Addon Display Settings
1. Order form display options:
   - Show on product page toggle
   - Show in shopping cart
   - Show in checkout summary
2. Set display order priority
3. Configure show/hide based on existing products

### Step 6: Auto-Provisioning Setup
1. If addon requires module action:
   - Select provisioning module
   - Configure module settings
   - Set action on order
2. Configure action on cancellation
3. Set action on upgrade/downgrade

### Step 7: Custom Fields for Addons
1. Add custom fields if needed:
   - Selection fields for addon variants
   - Text inputs for customization
2. Set field visibility rules

### Step 8: Addon Categories
1. Create addon categories for organization:
   - "Storage Addons"
   - "Security Addons"
   - "Performance Addons"
2. Assign addons to categories
3. Set category display order

### Step 9: Featured Addons
1. Mark premium addons as featured
2. Configure featured addons display
3. Set promotional pricing periods

### Step 10: Addon Workflows
1. Create automation for addon upsells:
   - Post-purchase upsell emails
   - In-cart recommendations
   - Checkout add-on prompts
2. Configure upgrade paths

## Verification Checklist
- [ ] Addon appears with linked products
- [ ] Pricing displays correctly
- [ ] Addon selectable in cart
- [ ] Module provisions addon correctly
- [ ] Upgrade path works

## Related Workflows
- whmcs-product-setup
- whmcs-upsell-checkout
- whmcs-checkout-flow

## Best Practices
- Group similar addons in categories
- Use clear naming conventions
- Set appropriate pricing for perceived value
- Test provisioning flow for each addon