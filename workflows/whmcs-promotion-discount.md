# WHMCS Promotion Discount Workflow

## Purpose
Configure discount-based promotions in WHMCS.

## Discount Types

### Percentage Discount
```
Example: 25% off
- Applied to subtotal
- Calculated automatically
- Can exceed certain amount
```

### Fixed Amount Discount
```
Example: $20 off
- Subtracts fixed amount
- Cannot exceed subtotal
- Applied directly
```

### Configurable Discount
```
Set specific amounts:
- 10% off for monthly
- 15% off for quarterly
- 20% off for annual
```

## Configuring Discounts

### Step 1: Choose Type
1. Navigate to: Configuration > System > Promotions
2. Select "Create New"
3. Choose discount type

### Step 2: Set Value
```
Percentage:
- Enter percentage (e.g., 25)
- Max: 100%

Fixed Amount:
- Enter amount (e.g., 50.00)
- Currency: Based on WHMCS setting
```

### Step 3: Configure Applies To
```
Options:
- Entire order
- Specific products
- Specific categories
- First order only
- Recurring billing
```

### Step 4: Set Conditions
```
Example conditions:
- Minimum order: $100
- Client group: VIP
- Products: Shared Hosting
- Billing cycle: Annual
```

## Discount Stacking

### Allow Stacking
```
Enable multiple discounts:
- Promotion A: 10%
- Promotion B: $20 off
- Can apply both if enabled
```

### Prevent Stacking
```
Conflict rules:
- Only one discount per order
- Category vs product discounts
- Auto vs code discounts
```

## Tiered Discounts

### Volume Discounts
```
Order Quantity:
- 1-5 items: 0% off
- 6-10 items: 5% off
- 11-25 items: 10% off
- 25+ items: 15% off
```

### Spend-Based Discounts
```
Order Value:
- $0-$100: No discount
- $101-$500: 5% off
- $501-$1000: 10% off
- $1000+: 15% off
```

## Discount Testing

### Test Scenarios
1. Apply to qualifying order
2. Apply with conditions
3. Verify exclusions
4. Check stacking behavior
5. Test expiration

### Debugging
```
Common issues:
- Discount not appearing: Check conditions
- Wrong amount: Check calculation settings
- Not applying: Check product restrictions
```

## Best Practices

### Offering Discounts
```
Guidelines:
- Limit duration
- Set clear conditions
- Test thoroughly
- Monitor usage
```

## Related Workflows
- whmcs-promotion-create
- whmcs-coupon-create
- whmcs-promotion-limit