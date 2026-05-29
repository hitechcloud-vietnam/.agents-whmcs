# WHMCS Affiliate Commission Workflow

## Purpose
Configure and manage affiliate commission structures.

## Commission Types

### Percentage Commission
```
Example:
- 10% of order total
- Applied to subtotal
- Calculated automatically
```

### Fixed Amount Commission
```
Example:
- $10 per sale
- Same regardless of order value
- Set per product/category
```

### Recurring Commission
```
Example:
- 5% of monthly payment
- While customer stays active
- For X months duration
```

## Setting Commission Rates

### Step 1: Configure General Rate
1. Navigate to: Configuration > Affiliates
2. Set default commission rate
3. Choose commission type

### Step 2: Product-Specific Rates
```
Configure:
- Different rates per product
- Different rates per category
- Override general rates
```

### Step 3: Tiered Rates
```
Set up:
- Level 1: 0-10 referrals = 5%
- Level 2: 11-25 referrals = 10%
- Level 3: 26+ referrals = 15%
```

## Commission Rules

### Order-Based
```
Rules:
- New orders only
- Include renewals
- Include upgrades
```

### Time-Based
```
Rules:
- Cookie duration: 30 days
- Attribution: First-click/Last-click
- Hold period: 14 days
```

### Exclusions
```
Exclude:
- Refunded orders
- Fraudulent orders
- Specific products
- Certain client groups
```

## Commission Calculation

### Standard Calculation
```
Order Total: $100
Commission Rate: 10%
Commission: $10
```

### With Tier
```
Order Total: $100
Tier: 10%
Commission: $10
```

### Recurring Calculation
```
Monthly Payment: $50
Recurring Rate: 5%
Monthly Commission: $2.50
Duration: 12 months
Total Commission: $30
```

## Commission Tracking

### Dashboard
```
View:
- Pending commissions
- Approved commissions
- Paid commissions
- Lifetime value
```

### Affiliate Reports
```
Per Affiliate:
- Referrals count
- Sales volume
- Commission earned
- Paid out
- Balance
```

## Commission Adjustments

### Manual Adjustments
```
Options:
- Add bonus commission
- Deduct for refunds
- Adjust for disputes
```

### Automatic Adjustments
```
Process:
- Refund: Deduct commission
- Chargeback: Deduct commission
- Cancellation: Adjust recurring
```

## Commission Payout

### Processing
1. Generate commission report
2. Review pending commissions
3. Approve for payout
4. Process payment

### Minimum Payout
```
Set:
- Minimum amount: $50
- Below minimum: Carry over
- Monthly/Quarterly schedule
```

## Testing Commission

### Test Scenarios
1. New order - calculate commission
2. Refund - adjust commission
3. Tier change - verify new rate
4. Recurring - verify monthly

## Best Practices

### Guidelines
```
- Clear commission structure
- Competitive rates
- Transparent tracking
- Timely payouts
```

## Related Workflows
- whmcs-affiliate-setup
- whmcs-affiliate-payout
- whmcs-affiliate-tracking