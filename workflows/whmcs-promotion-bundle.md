# WHMCS Promotion Bundle Workflow

## Purpose
Create bundle promotions that offer products together at a discounted rate.

## Bundle Types

### Product Bundle
```
Example: Starter Kit Bundle
- Product A: $30/mo
- Product B: $20/mo
- Product C: $15/mo
- Total: $65/mo
- Bundle price: $50/mo
- Savings: $15/mo
```

### Add-on Bundle
```
Example: Security Suite
- SSL Certificate: $10/mo
- SiteLock: $5/mo
- Site Backup: $8/mo
- Bundle: $18/mo (save $5)
```

## Creating Bundles

### Step 1: Create Bundle Products
1. Navigate to: Configuration > Products/Services
2. Create new product: "Bundle Name"
3. Set as bundle type

### Step 2: Add Bundle Items
```
Bundle Configuration:
- Product 1: Shared Hosting Basic
- Product 2: Email Hosting
- Product 3: SSL Certificate
```

### Step 3: Set Pricing
```
Pricing:
- Regular price: Sum of all products
- Bundle price: Discounted total
- Setup fee: Optional
```

### Step 4: Configure Display
```
Show on order:
- Display individual products
- Show savings amount
- Show bundle benefits
```

## Bundle Strategy

### Use Cases
```
Common bundles:
- Starter packages
- Enterprise packages
- Industry-specific
- Tier-based offerings
```

### Pricing Models
```
Flat bundle:
- Fixed price regardless

Percentage savings:
- Show percentage off

Tiered bundles:
- Basic, Standard, Premium
```

## Bundle Display

### Order Page
```
Bundle Display:
[Bundles]
-----------------------------------------
Starter Kit - $29/mo
Includes:
  - Basic Hosting ($15 value)
  - Email ($8 value)
  - SSL ($6 value)
Total Value: $29 - You Save: $8
[Select Bundle]
-----------------------------------------
```

## Bundle Management

### Upgrades
```
Bundle upgrades:
- Upgrade within bundle
- Add items to bundle
- Change bundle tier
```

### Cancellations
```
Handling:
- Cancel entire bundle
- Remove individual items
- Downgrade options
```

## Testing Bundles

### Test Scenarios
1. Add bundle to cart
2. Verify included products
3. Check pricing
4. Test upgrades
5. Test cancellations

## Related Workflows
- whmcs-promotion-create
- whmcs-promotion-upsell
- whmcs-promotion-discount