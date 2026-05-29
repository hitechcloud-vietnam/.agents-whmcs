# WHMCS Coupon Restrict Workflow

## Purpose
 configure restrictions on coupon codes.

## Restriction Types

### Product Restrictions
```
Limit to:
- Specific products
- Specific categories
- Exclude products
- Require certain products
```

### Client Restrictions
```
Limit to:
- New clients only
- Specific client groups
- Specific countries
- First order only
```

### Order Restrictions
```
Limit to:
- Minimum order value
- Maximum discount
- Specific payment methods
- Specific billing cycles
```

## Configuring Restrictions

### Step 1: Edit Coupon
1. Navigate to: Configuration > Promotions
2. Select coupon
3. Click "Edit"

### Step 2: Set Product Restrictions
```
Applies To:
- All products: No restriction
- Selected products: Choose specific
- Categories: Choose categories
```

### Step 3: Set Client Restrictions
```
Client Requirements:
- All clients: No restriction
- New clients: First order only
- Client groups: Select groups
- Countries: Select countries
```

### Step 4: Set Order Restrictions
```
Order Requirements:
- Minimum order: $50
- Maximum discount: $100
- Payment method: PayPal only
- Billing cycle: Annual only
```

## Common Restrictions

### New Customer Only
```
Restrictions:
- Client: New only
- Uses per client: 1
- Products: All
```

### Product-Specific
```
Restrictions:
- Products: Hosting only
- Categories: None
- Exclude: Add-ons
```

### Minimum Spend
```
Restrictions:
- Minimum order: $100
- Exclude: Shipping
- Includes: Products only
```

## Multiple Restrictions

### Combining Rules
```
Example:
- Products: Shared Hosting, VPS
- Minimum order: $50
- Client groups: Standard, Premium
- Countries: US, UK, CA

ALL conditions must be met
```

### OR Logic
```
Example:
- Product A OR Product B
- Client group: VIP OR Partner

ANY condition can be met
```

## Restriction Testing

### Test Scenarios
1. Qualifying order - should work
2. Non-qualifying product - should fail
3. Below minimum - should fail
4. Wrong client group - should fail

### Debug Mode
```
Enable:
- Show restriction details
- Display why failed
- Log all attempts
```

## Restriction Messages

### User-Friendly Messages
```
Display:
- "This coupon is for hosting only"
- "Minimum order of $50 required"
- "This coupon is for new customers only"
```

### Custom Messages
```
Set in coupon:
- Error message when invalid
- Help text on coupon field
```

## Advanced Restrictions

### Time-Based
```
Restrictions:
- Day of week: Weekdays only
- Time of day: Business hours
- Date range: Blackout dates
```

### Device-Based
```
Restrictions:
- Mobile only
- Desktop only
- First device only
```

## Best Practices

### Guidelines
```
- Clear restriction messaging
- Test all scenarios
- Document restrictions
- Monitor for issues
```

## Related Workflows
- whmcs-coupon-create
- whmcs-coupon-limit
- whmcs-promotion-target