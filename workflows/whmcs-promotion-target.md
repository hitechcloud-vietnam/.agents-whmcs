# WHMCS Promotion Target Workflow

## Purpose
Target promotions to specific customer segments.

## Targeting Options

### Client-Based
```
Target by:
- Client group (VIP, Standard)
- Client status (Active, Suspended)
- Registration date (New, Existing)
- Location (Country, State)
```

### Product-Based
```
Target by:
- Product type (Hosting, Domain)
- Product category
- Specific products
- Add-ons owned
```

### Behavior-Based
```
Target by:
- Order history
- Spending level
- Activity level
- Engagement score
```

## Creating Targeted Promotions

### Step 1: Define Audience
1. Navigate to: Configuration > Promotions
2. Create new promotion
3. Find "Applies To" section

### Step 2: Set Conditions
```
Condition Builder:
- IF client_group == "VIP"
- AND order_total > 100
- THEN apply promotion
```

### Step 3: Combine Rules
```
Multiple conditions:
- AND (all must match)
- OR (any can match)
- Nested groups
```

### Step 4: Exclude Segments
```
Exclude:
- Existing customers (for new customer offers)
- Specific products
- Certain client groups
```

## Targeting Examples

### New Customer Offer
```
Target:
- Client status: New
- No previous orders
- First order only

Exclude:
- Existing clients
- Test accounts
```

### High-Value Client Reward
```
Target:
- Client group: VIP
- Total spent > $1000
- Active status

Offer:
- Extra 10% discount
- Priority support
- Early access
```

### Lapsed Customer Win-Back
```
Target:
- Last order: 60+ days ago
- No active subscriptions
- Email subscribed

Offer:
- 30% off return
- Free shipping
- Limited time
```

### Product-Specific Upgrade
```
Target:
- Product: Basic Hosting
- No upgrade in 6 months

Offer:
- 20% off Pro plan
- Free migration
- 3 months free
```

## Dynamic Targeting

### Real-Time Rules
```
Apply based on:
- Current cart contents
- Time of day
- Device type
- Referrer source
```

### Example
```
IF cart_total > 500
AND device == mobile
THEN show mobile-exclusive discount
```

## Segment Creation

### Create Segments
1. Navigate to: Configuration > Clients > Client Groups
2. Create custom groups
3. Add members automatically
4. Use for targeting

### Auto-Segmentation
```
Automatic groups:
- High spenders (>$1000)
- At-risk (no login 30 days)
- Champions (referrals)
- New customers (last 30 days)
```

## Testing Targeting

### Test Mode
```
Verify:
- Correct clients targeted
- Exclusions work
- Multiple conditions
- Edge cases
```

### Preview
```
Show:
- Number of eligible clients
- Estimated impact
- Verification of rules
```

## Related Workflows
- whmcs-promotion-create
- whmcs-notification-filter
- whmcs-client-segment