# WHMCS Product Setup Workflow

## Purpose
Create and configure products in WHMCS for selling hosting, services, or digital goods.

## Prerequisites
- WHMCS admin access
- Product group defined
- Pricing structure planned
- Tax configuration completed

## Step-by-Step Process

### Step 1: Access Product Configuration
1. Log into WHMCS admin panel
2. Navigate to `Configuration > Products/Services`
3. Click `Create a New Product`

### Step 2: Basic Product Information
1. Select product type (Hosting Account, Reseller Account, VPS, Dedicated Server, Other)
2. Enter product name (e.g., "Starter Hosting")
3. Enter product description (use HTML formatting for rich content)
4. Select product group from dropdown
5. Enable/disable product as needed
6. Set sort order for display

### Step 3: Module Settings
1. Select module from available options (cPanel, Plesk, AutoSSL, etc.)
2. Configure module settings:
   - Server group assignment
   - Auto-setup toggle
   - Auto-configure toggle
   - Username prefix/suffix rules
3. Set import configuration if applicable

### Step 4: Pricing Configuration
1. Configure pricing tab:
   - Monthly, quarterly, semi-annual, annual pricing
   - Setup fees (one-time)
   - Currency selection
   - Enable/disable billing cycles
2. Set promotional pricing if needed
3. Configure overage pricing for metered products

### Step 5: Custom Fields
1. Navigate to Custom Fields tab
2. Add custom fields as needed:
   - Field name
   - Field type (text, dropdown, checkbox, textarea)
   - Required/optional
   - Client editable toggle
3. Set validation rules

### Step 6: Configurable Options
1. Go to Configurable Options tab
2. Assign existing configurable option groups
3. Create new option groups inline if needed

### Step 7: Settings Tab
1. Configure:
   - Allow stock control
   - Stock quantity threshold
   - Weight/dimensions for shipping
   - Download file association (for downloadable products)
2. Set hidden status if needed

### Step 8: Order Form Settings
1. Order form tab:
   - Show on order form toggle
   - Product highlighting options
   - Featured product toggle
   - Click through URL (for external products)

### Step 9: Automation Settings
1. Automation tab:
   - First payment term (affects billing cycle)
   - Termination settings
   - Auto-termination on overdue
   - Termination days threshold

### Step 10: Custom Links & Buttons
1. Add custom button links in product details
2. Configure additional action links
3. Set up upsell/cross-sell links

### Step 11: Review and Save
1. Review all settings
2. Click `Save Changes`
3. Test product creation in client area

## Verification Checklist
- [ ] Product appears in correct group
- [ ] Pricing displays correctly on order form
- [ ] Module connection works (test provisioning)
- [ ] Custom fields save and display properly
- [ ] Order form shows all configured options

## Related Workflows
- whmcs-product-bundle
- whmcs-configurable-options
- whmcs-custom-pricing

## Troubleshooting
- Module not showing: Check module is installed and enabled
- Pricing not saving: Verify currency is active
- Order form missing: Check product group settings