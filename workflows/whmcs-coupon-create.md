# WHMCS Coupon Create Workflow

## Purpose
Create and manage coupon codes for promotional offers.

## Coupon Types

### Percentage Coupon
```
Example: 25% off
- Code: SAVE25
- Discount: 25% off order
- Usage: Unlimited
```

### Fixed Amount Coupon
```
Example: $50 off
- Code: FLAT50
- Discount: $50 off
- Usage: 100 uses
```

### Free Shipping Coupon
```
Example: Free shipping
- Code: FREESHIP
- Discount: Shipping cost
- Usage: 50 uses
```

## Creating Coupons

### Step 1: Access Coupons
1. Navigate to: Configuration > Promotions
2. Click "Create New"
3. Select "Coupon"

### Step 2: Basic Information
```
Coupon Details:
- Name: Summer Sale 2024
- Code: SUMMER2024 (unique)
- Type: Percentage
- Value: 25
```

### Step 3: Configure Settings
```
Settings:
- Maximum Uses: 100
- Uses Per Client: 1
- Minimum Order: $50
- Valid From: 2024-06-01
- Valid Until: 2024-08-31
```

### Step 4: Set Applies To
```
Applies To:
- All products
- Specific categories
- Specific products
- Exclude products
```

### Step 5: Save Coupon
1. Review settings
2. Save coupon
3. Test with order

## Code Generation

### Manual Entry
```
Enter code:
- SUMMER2024
- PARTNER50
- WELCOME25
```

### Auto-Generate
```
Generate options:
- Random alphanumeric
- Sequential codes
- Prefix + random
```

### Bulk Generation
```
Create multiple:
- Prefix: SUMMER
- Quantity: 100
- Format: SUMMER-XXXX
```

## Coupon Validation

### Checking Code
```
Valid when:
- Code exists
- Not expired
- Uses remaining
- Conditions met
```

### Error Messages
```
Invalid code:
- "Code not found"
- "Code expired"
- "Limit reached"
- "Minimum not met"
```

## Testing Coupons

### Test Scenarios
1. Apply valid code - should work
2. Apply expired code - should fail
3. Apply when limit reached - should fail
4. Apply below minimum - should fail

### Test Mode
```
Enable:
- Test mode for new coupons
- Verify before public
- Check calculations
```

## Distribution

### Distribution Methods
```
- Email to customers
- Social media posts
- Partner links
- Invoices/receipts
- Physical (if applicable)
```

### Tracking
```
Track:
- Code usage
- Client usage
- Order value
- Revenue impact
```

## Managing Coupons

### Edit
1. Find coupon
2. Modify settings
3. Save changes

### Disable
1. Toggle active status
2. Set end date
3. Delete coupon

### Delete
1. Archive (keep history)
2. Delete (permanent)

## Best Practices

### Guidelines
```
- Unique, easy-to-remember codes
- Clear expiration dates
- Set reasonable limits
- Track usage closely
```

## Related Workflows
- whmcs-promotion-create
- whmcs-coupon-manage
- whmcs-coupon-tracking