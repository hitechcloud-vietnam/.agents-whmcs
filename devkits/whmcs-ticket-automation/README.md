# WHMCS Ticket Automation Module

Automated ticket handling with routing rules, auto-responses, escalation, and SLA management.

## Features

- Ticket routing rules
- Auto-reply templates
- Automatic escalation
- SLA tracking
- Priority-based processing
- Condition-based actions

## Installation

Copy module to `/path/to/whmcs/modules/servers/ticketautomation/` and activate.

## Usage

```php
// Create routing rule
ticketautomation_CreateRule(array(
    'name' => 'Billing Department Routing',
    'rule_type' => 'routing',
    'conditions' => array(array('field' => 'subject', 'operator' => 'contains', 'value' => 'billing')),
    'actions' => array(array('type' => 'set_department', 'department_id' => 2)),
    'priority' => 10
));

// Create escalation rule
ticketautomation_CreateRule(array(
    'name' => 'High Priority Escalation',
    'rule_type' => 'escalation',
    'conditions' => array(array('field' => 'urgency', 'operator' => 'equals', 'value' => 'High')),
    'actions' => array(array('type' => 'escalate', 'level' => 1), array('type' => 'set_priority', 'priority' => 'High'))
));

// Create SLA rule
ticketautomation_CreateRule(array(
    'name' => 'Standard SLA',
    'rule_type' => 'sla',
    'conditions' => array(),
    'actions' => array(array('type' => 'set_sla', 'response_hours' => 4, 'resolution_hours' => 24))
));

// Process ticket
ticketautomation_ProcessTicket($ticketId);

// Get SLA status
$sla = ticketautomation_GetSLAStatus($ticketId);

// Get breached SLAs
$breached = ticketautomation_GetBreachedSLAs();
```

## API Functions

| Function | Description |
|----------|-------------|
| `ticketautomation_CreateRule()` | Create automation rule |
| `ticketautomation_ProcessTicket()` | Process ticket with rules |
| `ticketautomation_CreateSLAPlan()` | Create SLA plan |
| `ticketautomation_GetSLAStatus()` | Get SLA status |
| `ticketautomation_UpdateSLAStatus()` | Update SLA status |
| `ticketautomation_GetBreachedSLAs()` | Get breached SLAs |
| `ticketautomation_GetEscalations()` | Get ticket escalations |
