# WHMCS Coupon Limit Workflow

## Purpose
 set and manage usage limits for coupon codes.

## Limit Types

### Total Uses Limit
```
Set maximum:
- 100 uses total
- Unlimited uses
- Custom number
```

### Per-Client Limit
```
Restrict:
- One use per client
- Two uses per client
- No limit per client
```

### Combined Limits
```
Example:
- Total: 500 uses
- Per client: 1 use
- Client must be new
```

## Setting Limits

### Step 1: Configure Limits
1. Navigate to: Configuration > Promotions
2. Select coupon
3. Find "Limits" section

### Step 2: Set Maximum Uses
```
Maximum Uses:
- Unlimited: No limit
- Limited: Enter number
- Example: 100 uses
```

### Step 3: Set Per-Client Limit
```
Uses Per Client:
- Unlimited: No limit
- Limited: Enter number
- Example: 1 per client
```

### Step 4: Save Settings
```
Save limits:
- Track remaining uses
- Block when exhausted
- Show usage to clients
```

## Limit Enforcement

### Automatic Enforcement
```
WHHCS handles:
- Check before applying
- Increment usage on success
- Block when limit reached
```

### User Feedback
```
When limit reached:
- "Coupon usage limit exceeded"
- "This coupon has been fully redeemed"
- "Try another code"
```

## Managing Limits

### Monitor Usage
```
Track:
- Uses remaining
- Uses today
- Popular times
- By client
```

### Adjust Limits
```
Increase:
- Add more uses
- Remove per-client limit
- Extend duration

Decrease:
- Reduce uses
- Lower per-client limit
```

### Reset Usage
```
Options:
- Reset total uses
- Reset per-client uses
- Clear all history
```

## Limit Scenarios

### One-Time Use
```
Limits:
- Max uses: 1
- Per client: 1
- Use: New customer offer
```

### Limited Edition
```
Limits:
- Max uses: 100
- Per client: 2
- Use: Special promotion
```

### Partner Codes
```
Limits:
- Max uses: Unlimited
- Per client: 1
- Use: Partner distribution
```

## Limit Notifications

### Admin Alerts
```
Configure:
- Alert at 80% usage
- Alert when limit reached
- Daily usage summary
```

### Client Display
```
Show:
- "X uses remaining"
- "Limited time offer"
- "Don't miss out"
```

## Testing Limits

### Test Scenarios
1. First use - should work
2. Second use within limit - should work
3. Exceed limit - should fail
4. Different client after limit - should work

### Verify
```
Check:
- Count accuracy
- Per-client tracking
- Time-based resets
```

## Best Practices

### Guidelines
```
- Set reasonable limits
- Monitor usage closely
- Plan for demand
- Have backup codes ready
```

## Related Workflows
- whmcs-coupon-create
- whmcs-coupon-expire
- whmcs-coupon-tracking