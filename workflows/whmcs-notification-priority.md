# WHMCS Notification Priority Workflow

## Purpose
Set and manage notification priority levels for proper routing.

## Priority Levels

### Critical
- Security breaches
- Fraud detection
- Payment failures
- System errors

### High
- Large orders
- VIP client issues
- SLA breaches
- Support escalations

### Medium
- Standard orders
- Regular updates
- Invoice notifications
- Support tickets

### Low
- Informational
- Newsletters
- Promotions
- General updates

## Setting Priority

### Step 1: Configure Default
1. Navigate to: Configuration > System > Notifications
2. Select notification
3. Set "Priority" level
4. Save

### Step 2: Dynamic Priority
Use conditions to set priority dynamically:
```
IF invoice_amount > 1000 THEN High
IF client_group == "VIP" THEN High
IF ticket_priority == "Urgent" THEN Critical
```

## Priority Routing

### By Channel
```
Critical: SMS + Email + Push
High: Email + Slack
Medium: Email only
Low: Digest only
```

### By Recipient
```
Critical: Manager + Director
High: Team Lead + Admin
Medium: Assigned Admin
Low: General admin queue
```

### By Timing
```
Critical: Immediate
High: Within 1 hour
Medium: Within 4 hours
Low: Within 24 hours
```

## Escalation

### Auto-Escalation Rules
```
After 1 hour no action:
- Low → Medium

After 4 hours no action:
- Medium → High

After 24 hours no action:
- High → Critical
```

### Escalation Actions
- Increase priority
- Add recipients
- Change channel
- Notify manager

## Priority Indicators

### Visual Indicators
- Critical: Red badge
- High: Orange badge
- Medium: Yellow badge
- Low: Gray badge

### Sound Alerts
- Critical: Alarm sound
- High: Alert sound
- Medium: Notification sound
- Low: No sound

## Priority Reports

### Metrics
- Notifications by priority
- Response times by priority
- Resolution times by priority

### SLA Tracking
- Critical: 1 hour response
- High: 4 hour response
- Medium: 24 hour response
- Low: 72 hour response

## Related Workflows
- whmcs-notification-rules
- whmcs-notification-create
- whmcs-notification-scheduling