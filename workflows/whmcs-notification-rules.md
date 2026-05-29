# WHMCS Notification Rules Workflow

## Purpose
Create and manage rules for conditional notification delivery.

## Rule Structure

### Rule Components
1. **Condition**: What triggers the rule
2. **Operator**: How to evaluate
3. **Value**: What to match
4. **Action**: What happens

## Creating Rules

### Step 1: Access Rules
1. Navigate to: Configuration > System > Notifications
2. Select notification
3. Click "Rules" tab

### Step 2: Add Rule
1. Click "Add Rule"
2. Select condition type:
   - Client attributes
   - Product/Service attributes
   - Invoice attributes
   - Order attributes

### Step 3: Configure Condition
```
Condition Types:
- Client Group equals "VIP"
- Invoice Amount greater than 500
- Product Type contains "Reseller"
- Service Status equals "Active"
- Domain Extension equals ".com"
```

### Step 4: Add Multiple Conditions
1. Use AND for all must match
2. Use OR for any can match
3. Nest groups with parentheses

### Step 5: Set Actions
```
Actions:
- Send notification: Yes/No
- Change priority: Low/Medium/High
- Add tags: custom tags
- Route to channel: specific channel
```

## Rule Examples

### High-Value Invoice Alert
```
IF:
  invoice_amount > 1000
THEN:
  Set priority: High
  Route to: Slack #finance-alerts
```

### New VIP Client
```
IF:
  client_group == "VIP"
THEN:
  Send notification: Yes
  Add tags: ["vip", "welcome"]
  Route to: Email + Slack
```

### Overdue Invoice Critical
```
IF:
  invoice_status == "Overdue"
  AND days_overdue > 7
THEN:
  Set priority: Critical
  Route to: SMS + Email
```

### Domain Renewal
```
IF:
  domain_days_until_expiry <= 30
  AND domain_days_until_expiry > 0
THEN:
  Send notification: Yes
  Route to: Admin + Client
```

## Rule Priority
1. Rules process in order
2. First matching rule wins
3. Set priority in rules list
4. Use "Else" for default

## Testing Rules
1. Use "Test Rule" function
2. Input sample data
3. Verify rule triggers
4. Check correct action

## Related Workflows
- whmcs-notification-create
- whmcs-notification-triggers
- whmcs-notification-filter