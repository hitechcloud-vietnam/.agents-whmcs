# WHMCS Automation Rules Workflow

## Overview
This workflow defines rule-based automation for WHMCS operations, enabling automated responses to system events without custom code.

## Prerequisites
- WHMCS installation (v7.0+)
- Admin access to Configuration > Automation Settings
- Basic understanding of WHMCS lifecycle hooks

## Step-by-Step Process

### Step 1: Access Automation Rules
1. Log into WHMCS admin area
2. Navigate to **Configuration > System Settings > Automation Rules**
3. Review existing rules before creating new ones

### Step 2: Create New Automation Rule
1. Click **Add New Rule**
2. Configure basic settings:
   - **Rule Name**: Descriptive identifier
   - **Description**: Purpose explanation
   - **Status**: Enabled/Disabled toggle
   - **Order**: Execution priority (lower = first)

### Step 3: Define Trigger Conditions
1. Select trigger type:
   - **Service Changes**: Create, Suspend, Unsuspend, Terminate, Upgrade/Downgrade
   - **Invoice Events**: Create, Due, Paid, Overdue, Void
   - **Client Events**: Signup, Login, Update, Delete
   - **Domain Events**: Registration, Transfer, Renewal, Expiration
   - **Support Events**: Ticket Create, Reply, Close
   - **Order Events**: New, Pending, Active, Cancelled, Fraud

2. Add condition filters:
   ```
   AND/OR logic groups:
   - Product/Service type equals [specific product]
   - Payment term equals [monthly/annual]
   - Client group equals [VIP/Premium]
   - Invoice amount greater than [threshold]
   ```

### Step 4: Configure Actions
Select one or more actions to execute:
- **Service Actions**: Suspend, Unsuspend, Terminate, Change Package
- **Email Actions**: Send notification, Send template
- **Module Actions**: Call module function
- **API Actions**: Execute API call
- **Workflow Actions**: Create ticket, Add note

### Step 5: Set Time-Based Triggers
For scheduled automation:
1. Enable **Scheduled Execution**
2. Set cron schedule:
   ```
   */5 * * * *  = Every 5 minutes
   0 0 * * *     = Daily at midnight
   0 9 * * 1     = Every Monday at 9 AM
   ```

### Step 6: Test Automation Rule
1. Set rule to **Test Mode** (process but don't execute actions)
2. Trigger a test event matching conditions
3. Review automation log in **Utilities > Logs > Automation Log**
4. Verify expected behavior
5. Disable test mode for production

## Example Rules

### Example 1: Auto-Suspend Overdue Services
```php
Trigger: Invoice Status = Overdue for 7+ days
Conditions:
  - Service status = Active
  - Invoice balance > 0
Actions:
  - Suspend service
  - Send suspension notice email
  - Add admin note
```

### Example 2: Upgrade Notification
```php
Trigger: Service Created
Conditions:
  - Product type = Cloud Server
  - Package tier = Basic
Actions:
  - Send upgrade offer email
  - Create internal ticket for follow-up
```

### Example 3: VIP Client Priority Handling
```php
Trigger: Support Ticket Created
Conditions:
  - Client group = VIP
  - Department = Billing
Actions:
  - Set priority = High
  - Assign to premium support queue
  - Send acknowledgment with SLA
```

## Advanced Configuration

### Rule Chaining
Link rules together for complex workflows:
1. Rule A triggers on initial event
2. Rule A action sets flag on client/service
3. Rule B triggers on flag change
4. Rule B executes secondary actions

### Dynamic Values
Use Smarty templates for dynamic content:
```smarty
{if $client_state == 'overdue'}
  Your service will be suspended in {$days_remaining} days
{/if}
```

### Integration with Hooks
Combine automation rules with custom hooks:
```php
add_hook('AutomationRuleExecuted', 1, function($vars) {
    logActivity("Rule executed: " . $vars['rule_name']);
});
```

## Best Practices

1. **Order of Execution**: Place more specific rules before general ones
2. **Avoid Conflicts**: Never create contradictory rules
3. **Logging**: Always enable detailed logging for troubleshooting
4. **Testing**: Test rules in staging before production
5. **Performance**: Limit rule complexity; use module callbacks for heavy processing
6. **Monitoring**: Regularly review automation logs

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Rule not triggering | Check trigger type matches event |
| Action not executing | Verify conditions are met |
| Infinite loop | Set execution limits per rule |
| Performance impact | Optimize condition checks |

## Related Workflows
- [WHMCS Cron Automation](./whmcs-cron-automation.md)
- [WHMCS Event-Driven Automation](./whmcs-event-driven-automation.md)
- [WHMCS Webhook Automation](./whmcs-webhook-automation.md)