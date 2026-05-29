# WHMCS Coupon Expiration Workflow

## Purpose
Manage coupon expiration dates and handle expired coupons.

## Expiration Settings

### Setting Expiration
```
Options:
- Never expires
- Specific date (2024-12-31)
- Number of days from creation
- After X uses
```

### Configuring Duration
```
Example:
- Valid from: 2024-01-01
- Valid until: 2024-03-31
- Duration: 90 days
```

## Before Expiration

### Pre-Expiration Actions
```
30 days before:
- Review performance
- Plan renewal/discontinuation
- Notify stakeholders

7 days before:
- Send reminder to clients
- Update marketing materials
- Prepare next promotion
```

### Renewal Options
```
Options:
- Extend expiration date
- Create new version
- Archive and replace
```

## At Expiration

### Automatic Handling
```
At expiration:
- Coupon becomes invalid
- Show "expired" message
- Redirect to new offers
```

### Custom Messages
```
Display:
- "This coupon has expired"
- "Try our new offers"
- "Subscribe for new codes"
```

## Post-Expiration

### Archive Coupon
```
Keep for:
- Historical records
- Tax/reporting
- Performance analysis
```

### Delete Coupon
```
Delete:
- Only after archiving
- Remove completely
- Cannot be recovered
```

## Expiration Notifications

### Admin Notifications
```
Configure:
- Alert 30 days before
- Alert 7 days before
- Alert on expiration
```

### Client Notifications
```
Options:
- Notify of expiration
- Remind to use
- Suggest alternatives
```

## Bulk Expiration

### Schedule Multiple
```
Create:
- Batch expiration
- End of month
- End of quarter
```

### Extend Bulk
```
Extend:
- Select multiple coupons
- Set new expiration
- Apply to all
```

## Expiration Handling Examples

### Limited Time Offer
```
Coupon: SUMMER2024
Created: June 1
Expires: August 31
Automatically expires Sept 1
```

### Auto-Renewal
```
Coupon: ANNUAL
Created: January 1
Expires: December 31
Creates new code for next year
```

## Testing Expiration

### Test Scenarios
1. Before expiration - should work
2. On expiration date - should fail
3. After expiration - should fail
4. Extended expiration - should work

### Verify Behavior
```
Check:
- Date calculation
- Time zone handling
- Error messages
- Alternative offers
```

## Best Practices

### Guidelines
```
- Set clear expiration dates
- Communicate expiration
- Plan ahead for renewals
- Keep expiration records
```

## Related Workflows
- whmcs-coupon-create
- whmcs-coupon-limit
- whmcs-coupon-tracking