# WHMCS Product Bundles Setup Workflow

## Purpose
Create product bundles and packages

## Prerequisites
- WHMCS installed
- Products already created
- Admin access

## Step 1: Navigate to Product Bundles

Navigate to: Setup > Products/Services > Products/Services

Click "Create a New Product"

## Step 2: Create Bundle Product

1. Select "Product Bundle"
2. Configure basic info:
   ```
   Product Name: Starter Package
   Product Group: Packages
   Bundle Description: Everything you need to get started
   ```
3. Save

## Step 3: Configure Bundle Contents

Navigate to: Edit Bundle > Bundle Items

### Add Items
```
Item 1: Starter Hosting
  - Billing Cycle: Monthly
  - Quantity: 1

Item 2: Domain Registration
  - Billing Cycle: Annual
  - Quantity: 1

Item 3: SSL Certificate
  - Billing Cycle: Annual
  - Quantity: 1
```

## Step 4: Set Bundle Pricing

Navigate to: Edit Bundle > Pricing

### Pricing Strategy
```
Individual Total: $25.00/month
Bundle Price: $19.99/month
Savings: $5.01/month (20%)

Setup Fee: $0.00
```

## Step 5: Configure Bundle Discounts

Navigate to: Edit Bundle > Bundle Settings

```
Bundle Discount: 20%
Discount Type: Percentage / Fixed Amount
Apply to All Items: Yes
```

## Step 6: Set Up Bundle-Specific Options

Navigate to: Edit Bundle > Configuration Options

### Add Configurable Options
```
Addons Included:
  - Site Backup (Free)
  - Site Builder (Add $3/mo)
```

## Step 7: Configure Bundle Automation

Navigate to: Edit Bundle > Module Settings

```
Module: [as needed]
AutoSetup: Products Only
Prorata: Yes
```

## Step 8: Create Multiple Bundle Tiers

### Bundle Tiers
```
Starter Package:
  - Basic Hosting
  - Free Domain
  - No SSL

Professional Package:
  - Premium Hosting
  - Free Domain
  - Free SSL
  - Free Backup

Enterprise Package:
  - Business Hosting
  - Free Domain
  - Premium SSL
  - All Addons
```

## Step 9: Set Up Bundle Dependencies

Navigate to: Edit Bundle > Dependencies

```
Prerequisite Product: [if needed]
Required Addons: [optional]
```

## Step 10: Configure Bundle Display

Navigate to: Edit Bundle > Order Form Settings

```
Show Individual Prices: Yes
Show Savings: Yes
Highlight Savings: Yes
Bundle Image: [upload image]
```

## Bundle Checklist

- [ ] Bundle product created
- [ ] Contents configured
- [ ] Pricing set
- [ ] Discounts configured
- [ ] Options added
- [ ] Automation set
- [ ] Multiple tiers created
- [ ] Dependencies configured
- [ ] Display settings customized
