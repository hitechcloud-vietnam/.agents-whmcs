# WHMCS TicketOpen Hook Reference

## Overview

The `TicketOpen` hook fires when a new support ticket is opened in WHMCS. This hook triggers when a client or admin creates a new ticket through the client area or admin panel.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `ticketid` | int | The unique ticket ID |
| `tid` | string | The ticket tracking ID |
| `userid` | int | The client ID (0 if not logged in) |
| `contactid` | int | The contact ID (if applicable) |
| `subject` | string | Ticket subject |
| `message` | string | Initial ticket message |
| `priority` | string | Priority (Low, Medium, High, Urgent) |
| `status` | string | Ticket status |
| `departmentid` | int | Department ID |
| `model` | object | The Ticket model instance |

## Example Implementation

```php
<?php
add_hook('TicketOpen', 1, function(array $params) {
    // Log ticket creation
    logActivity("Support ticket opened: {$params['tid']} - {$params['subject']}");
    
    // Get client info for additional processing
    if ($params['userid'] > 0) {
        $client = getClientsDetails($params['userid']);
    }
    
    return $params;
});
```

## Auto-Assignment and Routing

```php
<?php
add_hook('TicketOpen', 1, function(array $params) {
    // Auto-assign based on keywords in subject
    $subject = strtolower($params['subject']);
    
    if (strpos($subject, 'billing') !== false || strpos($subject, 'invoice') !== false) {
        $newDept = 2; // Billing department
        $newAssignedId = getLeastLoadedAdminInDept(2);
    } elseif (strpos($subject, 'technical') !== false || strpos($subject, 'error') !== false) {
        $newDept = 3; // Technical department
        $newAssignedId = getLeastLoadedAdminInDept(3);
    } elseif (strpos($subject, 'sales') !== false || strpos($subject, 'quote') !== false) {
        $newDept = 4; // Sales department
        $newAssignedId = getLeastLoadedAdminInDept(4);
    } else {
        // Default to general support
        $newDept = 1;
        $newAssignedId = getLeastLoadedAdminInDept(1);
    }
    
    // Update ticket assignment
    update_query('tbltickets', [
        'did' => $newDept,
        'admin' => $newAssignedId
    ], ['id' => $params['ticketid']]);
    
    return $params;
});
```

## Priority and SLA Management

```php
<?php
add_hook('TicketOpen', 1, function(array $params) {
    $userId = (int)$params['userid'];
    
    // 1. Determine priority based on client tier
    if ($userId > 0) {
        $client = getClientsDetails($userId);
        $clientGroup = $client['groupid'];
        
        if ($clientGroup == 3) { // Premium tier
            $autoPriority = 'High';
        } elseif ($clientGroup == 4) { // VIP tier
            $autoPriority = 'Urgent';
        } else {
            $autoPriority = $params['priority'];
        }
        
        update_query('tbltickets', [
            'urgency' => $autoPriority
        ], ['id' => $params['ticketid']]);
    }
    
    // 2. Calculate SLA deadline
    $slaHours = match ($params['priority']) {
        'Urgent' => 1,
        'High' => 4,
        'Medium' => 8,
        'Low' => 24,
        default => 24
    };
    
    $slaDeadline = date('Y-m-d H:i:s', strtotime("+{$slaHours} hours"));
    
    // 3. Store SLA in custom field
    insert_query('tblticketcustomfieldsvalues', [
        'ticketid' => $params['ticketid'],
        'fieldid' => getCustomFieldId('SLA Deadline'),
        'value' => $slaDeadline
    ]);
    
    return $params;
});
```

## Notifications and Automation

```php
<?php
add_hook('TicketOpen', 1, function(array $params) {
    // 1. Send notification to Slack
    sendSlackMessage([
        'channel' => '#support-tickets',
        'message' => "New ticket: {$params['tid']}\n" .
                     "Subject: {$params['subject']}\n" .
                     "Priority: {$params['priority']}\n" .
                     "From: " . ($params['userid'] > 0 ? "Client #{$params['userid']}" : 'Guest')
    ]);
    
    // 2. Auto-reply with canned response
    $cannedResponse = getCannedResponse('initial_reply');
    addTicketReply($params['ticketid'], $cannedResponse, 'system');
    
    // 3. Notify specific admin if keyword match
    if (stripos($params['subject'], 'wordpress') !== false) {
        notifyAdmin('wordpress_specialist@example.com', 'New WordPress ticket', $params);
    }
    
    // 4. Create escalation for high-priority tickets
    if ($params['priority'] === 'Urgent') {
        createEscalation($params['ticketid'], 'urgent_queue');
    }
    
    return $params;
});
```

## Use Cases

- **Auto-Routing**: Direct tickets to appropriate departments
- **Priority Management**: Adjust priority based on client tier
- **SLA Tracking**: Set and track response deadlines
- **Notifications**: Alert staff via Slack, email, etc.
- **Canned Responses**: Auto-respond with templates

## Notes

- Runs after ticket is created but before confirmation email
- Can modify ticket properties before final save
- Use `TicketReply` for new replies to existing tickets
- Consider rate limiting to prevent ticket spam

## Related Hooks

- `TicketReply` - When new reply is added
- `TicketClose` - When ticket is closed
- `AdminLogin` - For admin context

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Support System Configuration](../whmcs-support-setup.md)