# WHMCS Configurable Options Workflow

## Purpose
Create configurable options (product customizations, add-ons, selections) that customers can choose during ordering.

## Prerequisites
- Products created in WHMCS
- Understanding of option types needed
- Pricing structure for options defined

## Step-by-Step Process

### Step 1: Access Configurable Options
1. Log into WHMCS admin
2. Navigate to `Configuration > Products/Services > Configurable Options`
3. Click `Create New Group`

### Step 2: Create Option Group
1. Enter group name (e.g., "Disk Space Options")
2. Select display order
3. Choose option type:
   - Dropdown (single selection)
   - Checkbox (multiple selection)
   - Radio (single selection with visual)
   - Quantity (numeric selection)

### Step 3: Add Options to Group
1. Click `Add Option` for each choice:
   - Option name (e.g., "5GB Storage")
   - Option type (addon, selectable)
   - Sort order
2. Configure pricing for each option:
   - Recurring price per cycle
   - One-time setup fee
   - Monthly/quarterly/annual pricing

### Step 4: Configure Option Pricing
1. Set pricing for each option:
   - Monthly recurring
   - Quarterly recurring
   - Annual recurring
   - One-time fees
2. Enable/disable specific billing cycles per option
3. Set modifier type (flat fee or percentage)

### Step 5: Link Options to Products
1. Product assignment section:
   - Select specific products
   - Select product groups
   - Apply to all products option
2. Set default selection

### Step 6: Advanced Option Settings
1. Configure option dependencies:
   - Require option A if option B selected
   - Hide/show based on selections
2. Set minimum/maximum quantities
3. Configure bulk pricing tiers

### Step 7: Display Settings
1. Show on order form toggle
2. Show in client area toggle
3. Show in invoices toggle
4. Required/optional selection
5. Multi-select limits

### Step 8: Option Validation Rules
1. Set validation:
   - Minimum selection count
   - Maximum selection count
   - Required combinations
2. Configure error messages

### Step 9: Bundle-Style Option Groups
1. Create "pick one" option groups:
   - Hosting tier selection
   - Operating system choice
   - Control panel selection
2. Set default recommended option

### Step 10: Nested Option Groups
1. Create dependent options:
   - Primary option selection
   - Secondary options filtered by primary
   - Configuration complete on all levels

## Verification Checklist
- [ ] Options display on product page
- [ ] Pricing updates in cart
- [ ] Option dependencies work correctly
- [ ] Invoice shows selected options
- [ ] Module receives option data

## Related Workflows
- whmcs-product-setup
- whmcs-checkout-flow
- whmcs-upsell-checkout

## Common Configurable Option Patterns
- Disk space/bandwidth upgrades
- Operating system selection
- Control panel choice
- Add-on features (SSL, backups, etc.)
- Licensing options