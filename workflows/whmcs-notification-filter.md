# WHMCS Notification Filter Workflow

## Purpose
Filter and control which notifications are sent based on conditions.

## Filter Types

### Client Filters
- Client group
- Client status
- Client language
- Registration date
- Country

### Product Filters
- Product type
- Product category
- Billing cycle
- Associated addons
- Product status

### Invoice Filters
- Invoice status
- Invoice amount
- Due date
- Tax amount
- Payment method

### Order Filters
- Order status
- Order source
- Payment method
- Order date
- Order total

## Filter Operators

### String Operators
```
equals
not_equals
contains
not_contains
starts_with
ends_with
is_empty
is_not_empty
```

### Number Operators
```
equals
not_equals
greater_than
less_than
between
```

### Date Operators
```
equals
before
after
within_days
older_than
```

## Creating Filters

### Step 1: Access Filter Builder
1. Navigate to: Configuration > System > Notifications
2. Select notification
3. Click "Add Filter"

### Step 2: Define Conditions
```
Condition Builder:
- Field: client_group
- Operator: equals
- Value: VIP

AND/OR
- Field: invoice_amount
- Operator: greater_than
- Value: 500
```

### Step 3: Set Actions
```
When conditions match:
- Send notification: Yes/No
- Modify priority: Low/Medium/High
- Route to channel: specific channel
- Add tags: custom tags
```

## Filter Examples

### High-Value Orders
```
IF:
  order_total > 1000
THEN:
  Route to: Slack #high-value-orders
  Set priority: High
  Add admin: finance@company.com
```

### VIP Client Support
```
IF:
  client_group == "VIP"
  AND ticket_priority == "High"
THEN:
  Route to: Slack #vip-support
  Notify: VIP manager
  Set priority: Critical
```

### Expiring Services
```
IF:
  days_until_expiry <= 7
  AND service_status == "Active"
THEN:
  Send reminder
  Set priority: Medium
  Include renewal link
```

## Complex Filters

### Nested Conditions
```
IF:
  (client_group == "Enterprise" OR client_group == "VIP")
  AND
  (invoice_amount > 500 OR order_total > 1000)
THEN:
  Route to: Enterprise channel
  Set priority: High
```

### Multiple Rules
```
Rule 1: High value → Slack #alerts
Rule 2: VIP → Email to manager
Rule 3: All others → Standard
```

## Exclusions

### Always Exclude
```
Exclude if:
- client_unsubscribed == true
- email_invalid == true
- client_status == "Closed"
```

## Testing Filters

### Test Mode
1. Enable test mode
2. Simulate trigger
3. Check filter matches
4. Verify correct action

## Filter Performance

### Optimization
- Use indexed fields
- Limit conditions
- Simplify logic

### Monitoring
- Track filter matches
- Identify slow filters
- Optimize frequently used

## Related Workflows
- whmcs-notification-rules
- whmcs-notification-create
- whmcs-notification-groups