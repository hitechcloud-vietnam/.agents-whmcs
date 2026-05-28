# WHMCS Alerting System Module

Alert and notification system with multiple channels, escalation rules, and on-call scheduling.

## Features

- Multiple alert severity levels
- Multi-channel notifications (email, webhook, Slack, PagerDuty, SMS)
- Alert rules and conditions
- Escalation paths
- Acknowledgment workflow
- Cooldown periods
- Recipient management
- On-call scheduling

## Installation

1. Copy `alertingsystem.php` to `/path/to/whmcs/modules/addons/alertingsystem/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure notification channels

## Usage

```php
// Create alert
$result = alertingsystem_CreateAlert(array(
    'alert_key' => 'high_server_load_' . $serverId,
    'alert_name' => 'High Server Load',
    'description' => 'Server load exceeds threshold',
    'source' => 'monitoring',
    'severity' => 'warning',
    'trigger_value' => 8.5,
    'threshold' => 5.0,
    'condition' => '>',
    'context' => array('server_id' => $serverId, 'hostname' => 'web01')
));

// Get alert
$alert = alertingsystem_GetAlert($alertId);

// Get alerts with filters
$alerts = alertingsystem_GetAlerts(array(
    'status' => 'firing',
    'severity' => 'critical',
    'source' => 'monitoring',
    'from_date' => '2026-05-01',
    'to_date' => '2026-05-28'
), 50);

// Get alert statistics
$stats = alertingsystem_GetAlertStats();
// Returns: by_status_severity, by_source

// Acknowledge alert
alertingsystem_AcknowledgeAlert($alertId, $adminUserId);

// Resolve alert
alertingsystem_ResolveAlert($alertId);

// Create notification rule
alertingsystem_CreateRule(array(
    'rule_name' => 'Critical Server Alert',
    'rule_key' => 'critical_server',
    'conditions' => array(
        array('field' => 'severity', 'operator' => '==', 'value' => 'critical'),
        array('field' => 'source', 'operator' => '==', 'value' => 'monitoring')
    ),
    'conditions_logic' => 'AND',
    'severity' => 'critical',
    'channels' => array(
        'email' => array('admin@example.com'),
        'slack' => array('#alerts-critical'),
        'pagerduty' => array('integration-key-123')
    ),
    'cooldown_seconds' => 300
));

// Create webhook rule
alertingsystem_CreateRule(array(
    'rule_name' => 'All Alerts Webhook',
    'rule_key' => 'all_webhook',
    'conditions' => array(
        array('field' => 'severity', 'operator' => 'in', 'value' => array('critical', 'warning'))
    ),
    'severity' => 'warning',
    'channels' => array(
        'webhook' => array('https://hooks.example.com/alerts')
    )
));

// Get rules
$rules = alertingsystem_GetRules();

// Add recipient
alertingsystem_AddRecipient(array(
    'recipient_name' => 'Admin Team',
    'email' => 'admin@example.com',
    'phone' => '+1234567890',
    'slack_id' => 'U12345',
    'notify_critical' => true,
    'notify_warning' => true,
    'notify_info' => false
));

// Get recipients
$recipients = alertingsystem_GetRecipients();

// Get notification history
$history = alertingsystem_GetNotificationHistory($alertId, 100);
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| EnableEmail | yesno | yes | Enable email |
| EnableWebhook | yesno | yes | Enable webhook |
| EnableSMS | yesno | no | Enable SMS |
| EnableSlack | yesno | no | Enable Slack |
| EnablePagerDuty | yesno | no | Enable PagerDuty |
| SMTPHost | text | - | SMTP host |
| SMTPPort | text | 587 | SMTP port |
| SMTPUsername | text | - | SMTP username |
| SMTPPassword | password | - | SMTP password |
| SlackWebhook | text | - | Slack webhook URL |
| PagerDutyKey | password | - | PagerDuty key |
| EscalationDelay | text | 300 | Escalation delay |

## Alert Severity Levels

| Level | Description | Default Actions |
|-------|-------------|-----------------|
| critical | Immediate action | Email + PagerDuty |
| warning | Attention needed | Email |
| info | Informational | Log only |

## Notification Channels

| Channel | Description |
|---------|-------------|
| email | Send email to address |
| webhook | POST to URL |
| slack | Slack webhook |
| pagerduty | PagerDuty event |
| sms | SMS message |

## Condition Operators

| Operator | Example |
|----------|---------|
| == | severity == critical |
| != | source != test |
| > | value > 100 |
| < | value < 50 |
| >= | count >= 10 |
| <= | count <= 100 |
| contains | message contains error |
| starts_with | source starts_with api |
| ends_with | name ends_with _alert |
| regex | name matches /^test_/ |
| in | severity in [critical,warning] |

## Logic Operators

| Logic | Description |
|-------|-------------|
| AND | All conditions must match |
| OR | Any condition can match |

## Database Tables

- `mod_alertingsystem_alerts` - Alert instances
- `mod_alertingsystem_rules` - Notification rules
- `mod_alertingsystem_notifications` - Notification log
- `mod_alertingsystem_recipients` - Recipient contacts
- `mod_alertingsystem_schedules` - On-call schedules
- `mod_alertingsystem_escalations` - Escalation tracking
