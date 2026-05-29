# WHMCS Promotion Limit Workflow

## Purpose
Configure usage limits for promotional offers.

## Limit Types

### Total Uses
```
Global limit:
- Maximum 100 uses total
- When reached, promotion ends
- Track remaining uses
```

### Per-Client Limit
```
Individual limit:
- One use per client
- Two uses per client
- Unlimited per client
```

### Time-Based Limits
```
Duration limits:
- First 50 customers
- Uses per hour
- Uses per day
```

## Setting Limits

### Step 1: Access Promotion
1. Navigate to: Configuration > Promotions
2. Edit promotion
3. Find "Limit" section

### Step 2: Configure Limits
```
Limit Options:
- Maximum Uses: 100
- Uses Per Client: 1
- Require Unique Code: No
- Auto-generate codes: Yes
```

### Step 3: Set Expiration
```
Expiration:
- Never expires
- Expires on date: 2024-12-31
- Expires after uses: 50
- Expires after days: 30
```

## Limit Examples

### One-Time Use Code
```
Limits:
- Maximum Uses: 1
- Uses Per Client: 1
- Applicable to: New clients only
```

### Limited Time Offer
```
Limits:
- Maximum Uses: 500
- Time limit: 7 days
- Per client: Unlimited
```

### VIP Exclusive
```
Limits:
- Maximum Uses: No limit
- Uses Per Client: 3
- Client group: VIP only
```

## Managing Limits

### Monitor Usage
```
Track:
- Uses remaining
- Uses per client
- Time remaining
- Popularity
```

### Adjust Limits
```
Options:
- Increase limit
- Decrease limit
- Extend expiration
- Add conditions
```

### Disable Limits
```
Temporarily disable:
- Remove maximum uses
- Extend expiration
- Allow multiple uses
```

## Limit Enforcement

### Automatic
```
WHHCS handles:
- Tracks usage in database
- Validates before applying
- Blocks when limits reached
```

### Custom Enforcement
```
Via hook:
add_hook('OrderCompleted', 1, function($vars) {
    // Custom limit logic
});
```

## Limit Notifications

### Admin Alerts
```
Configure:
- Alert when 80% used
- Alert when limit reached
- Alert for high usage
```

### Client Notifications
```
Display:
- Uses remaining
- Expires soon
- Limit reached message
```

## Testing Limits

### Test Scenarios
1. First use - should apply
2. Second use (limited) - should fail
3. Expired promotion - should fail
4. Different client - should apply

## Best Practices

### Guidelines
```
- Set reasonable limits
- Monitor usage
- Plan for high demand
- Have backup promotions
```

## Related Workflows
- whmcs-promotion-create
- whmcs-coupon-limit
- whmcs-coupon-tracking