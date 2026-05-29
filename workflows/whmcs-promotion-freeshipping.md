# WHMCS Promotion Free Shipping Workflow

## Purpose
Create free shipping promotions to drive sales.

## Free Shipping Types

### Automatic Free Shipping
```
Applied when:
- Order exceeds threshold
- Specific products
- Client group
```

### Code-Based Free Shipping
```
Requires code:
- Enter at checkout
- Time-limited
- Single or multiple use
```

### Conditional Free Shipping
```
Based on:
- Geographic location
- Shipping method
- Order combination
```

## Creating Free Shipping Promotions

### Step 1: Configure Shipping
1. Navigate to: Configuration > System > Automations
2. Enable shipping modules
3. Set base shipping rates

### Step 2: Create Promotion
1. Go to: Configuration > Promotions
2. Click "Create New"
3. Select "Free Shipping"

### Step 3: Set Conditions
```
Options:
- Minimum order amount: $50
- Specific products only
- Specific countries
- Specific shipping methods
```

### Step 4: Configure Duration
```
Timing:
- Limited time offer
- Ongoing
- First X customers
```

## Free Shipping Strategies

### Threshold-Based
```
Encourage higher orders:
- Free shipping over $50
- Free shipping over $100 (faster)
- Free shipping over $200 (express)
```

### Product-Based
```
特定产品包邮:
- Virtual products
- Downloadable items
- Service products
```

### Event-Based
```
Promotional:
- Holiday free shipping
- Flash sale shipping
- Member-only shipping
```

## Displaying Free Shipping

### Progress Indicator
```
Cart page display:
Your cart: $35
[====        ] Free shipping at $50!
Add $15 more for free shipping
```

### Promotional Banner
```
Homepage/Product page:
FREE SHIPPING on orders over $50!
Ends: [countdown]
```

## Combining with Other Discounts

### Allow Stacking
```
Options:
- Free shipping + percentage off
- Free shipping + fixed discount
- Free shipping + buy X get Y
```

### Exclude Stacking
```
Limitations:
- One shipping discount per order
- Shipping cannot be discounted further
```

## Testing Free Shipping

### Scenarios
1. Order below threshold - no free shipping
2. Order above threshold - free shipping applied
3. Mixed products - verify eligibility
4. With other discounts - verify stacking

## Cost Management

### Profit Impact
```
Consider:
- Product margins
- Shipping costs
- Threshold amounts
- Customer lifetime value
```

### Limits
```
Set maximum:
- Maximum shipping cost covered
- Number of free shipments
- Dollar amount budget
```

## Best Practices

### Guidelines
```
- Set reasonable thresholds
- Monitor profitability
- Promote clearly
- Combine with other offers
```

## Related Workflows
- whmcs-promotion-create
- whmcs-coupon-create
- whmcs-promotion-discount