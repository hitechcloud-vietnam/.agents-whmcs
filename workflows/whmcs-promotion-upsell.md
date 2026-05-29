# WHMCS Promotion Upsell Workflow

## Purpose
Create upsell promotions to increase order value.

## Upsell Types

### Cart Upsell
```
At checkout:
- "Add domain privacy - $2/mo"
- "Add SiteLock - $5/mo"
- "Add backup service - $3/mo"
```

### Order Upsell
```
After initial order:
- Upgrade to higher tier
- Add complementary products
- Extend service period
```

### Product Addon Upsell
```
After product purchase:
- "Add SSL to your hosting"
- "Add extra storage"
- "Add priority support"
```

## Creating Upsell Promotions

### Step 1: Identify Upsell Opportunities
```
Common upsells:
- Domain registration
- Privacy protection
- SSL certificates
- Extended support
- Additional services
```

### Step 2: Configure Upsell
1. Navigate to: Configuration > Promotions
2. Create new promotion
3. Set as "Upsell"

### Step 3: Set Trigger
```
Triggers:
- Product type
- Cart value
- Specific products
- Client segments
```

### Step 4: Configure Timing
```
Display at:
- Cart page
- Checkout page
- Order confirmation
- Post-purchase email
```

## Upsell Page Design

### Effective Elements
```
- Clear benefit statement
- Original price vs upsell price
- One-click add
- Limited time offer
```

### Example Display
```
EXCLUSIVE OFFER
Add Domain Privacy for just $2/mo
(Regular price: $5/mo)
You're saving 60%!

[X] Yes, add Domain Privacy (+$2/mo)
[No thanks]
```

## Upsell Pricing

### Strategy
```
Pricing models:
- Percentage off regular price
- Flat discount
- Bundled rate
- Free first period
```

### Positioning
```
Offer structure:
- Entry product: Low price
- Upsell: High perceived value
- Cross-sell: Complementary
```

## Post-Purchase Upsell

### Email Upsell
```
After order:
- Thank you email
- Suggest related products
- Limited time offer
- Personal recommendation
```

### Client Area Upsell
```
In client dashboard:
- Upgrade banner
- Add-on suggestions
- Feature highlights
```

## Testing Upsells

### Metrics
```
- Show rate: Shown / Orders
- Acceptance rate: Accepted / Shown
- Revenue per upsell
- Upsell ROI
```

### Optimization
```
A/B test:
- Different pricing
- Different positioning
- Different timing
- Different copy
```

## Best Practices

### Guidelines
```
- Relevant products only
- Clear pricing
- Easy to decline
- Don't oversell
```

## Related Workflows
- whmcs-promotion-create
- whmcs-promotion-bundle
- whmcs-coupon-create