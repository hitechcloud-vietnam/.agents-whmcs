# WHMCS Promotion Create Workflow

## Purpose
Create promotional offers and discounts in WHMCS.

## Promotion Types

### Discount Types
```
- Percentage off
- Fixed amount off
- Free item
- Free shipping
- Buy X get Y
```

### Application Types
```
- Automatic (applied at checkout)
- Code-based (manual entry)
- First order only
- Recurring discount
```

## Creating Promotions

### Step 1: Access Promotions
1. Navigate to: Configuration > System > Promotions
2. Click "Create New Promotion"

### Step 2: Basic Information
```
Promotion Details:
- Name: Enter descriptive name
- Code: Unique promotional code (if applicable)
- Type: Percentage/Fixed Amount
- Value: Discount amount
```

### Step 3: Configure Conditions
```
Applies To:
- All products
- Specific products
- Specific categories
- Specific client groups
```

### Step 4: Set Limits
```
Restrictions:
- Maximum uses
- Uses per client
- Minimum order value
- Valid dates
```

### Step 5: Save & Test
1. Save promotion
2. Test with sample order
3. Verify calculation

## Promotion Examples

### 20% Off First Order
```
Type: Percentage
Value: 20%
Condition: First order only
Limit: One use per client
```

### $50 Off Annual Plans
```
Type: Fixed Amount
Value: $50
Condition: Annual billing cycle
Applies to: Selected plans
```

### Free Setup
```
Type: Credit/Discount
Value: Setup fee waived
Condition: All products
Auto-apply: Yes
```

## Testing Promotions

### Test Checklist
- [ ] Discount calculates correctly
- [ ] Limits enforced properly
- [ ] Expiration works
- [ ] Excludes apply
- [ ] Stacking rules work

## Related Workflows
- whmcs-promotion-discount
- whmcs-coupon-create
- whmcs-promotion-limit