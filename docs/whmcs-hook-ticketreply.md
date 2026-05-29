# WHMCS TicketReply Hook Reference

## Overview

The `TicketReply` hook fires when a reply is added to a support ticket. This hook triggers for both client and admin replies, enabling automated responses, escalation, and integration workflows.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `ticketid` | int | The unique ticket ID |
| `tid` | string | The ticket tracking ID |
| `userid` | int | The client ID (0 if not logged in) |
| `contactid` | int | The contact ID (if applicable) |
| `admin` | string | Admin username (empty if client reply) |
| `message` | string | Reply message content |
| `status` | string | New ticket status |
| `model` | object | The Ticket model instance |

## Example Implementation

```php
<?php
add_hook('TicketReply', 1, function(array $params) {
    // Log the reply
    $replier = !empty($params['admin']) ? "Admin {$params['admin']}" : "Client #{$params['userid']}";
    logActivity("Reply added to ticket {$params['tid']} by {$replier}");
    
    return $params;
});
```

## Auto-Response System

```php
<?php
add_hook('TicketReply', 1, function(array $params) {
    // Only process client replies
    if (!empty($params['admin'])) {
        return $params;
    }
    
    $ticketId = (int)$params['ticketid'];
    $message = strtolower($params['message']);
    
    // 1. Check for common questions and auto-respond
    $autoResponses = [
        'how to' => 'Thanks for your question! For guides, visit: https://docs.example.com',
        'password' => 'To reset your password, visit: https://billing.example.com/pwreset',
        'refund' => 'Our refund policy is at: https://example.com/refund-policy',
        'cancel' => 'To cancel, visit: https://billing.example.com/cancel'
    ];
    
    foreach ($autoResponses as $keyword => $response) {
        if (strpos($message, $keyword) !== false) {
            addTicketReply($ticketId, $response, 'system');
            break;
        }
    }
    
    return $params;
});
```

## Escalation Rules

```php
<?php
add_hook('TicketReply', 1, function(array $params) {
    $ticketId = (int)$params['ticketid'];
    $message = $params['message'];
    
    // 1. Escalate if certain keywords detected
    $escalationKeywords = ['escalate', 'manager', 'supervisor', 'urgent', 'immediately'];
    $shouldEscalate = false;
    
    foreach ($escalationKeywords as $keyword) {
        if (stripos($message, $keyword) !== false) {
            $shouldEscalate = true;
            break;
        }
    }
    
    if ($shouldEscalate) {
        // Escalate ticket
        update_query('tbltickets', [
            'flag' => getAdminIdByUsername('manager'),
            'urgency' => 'Urgent'
        ], ['id' => $ticketId]);
        
        // Notify manager
        $managerEmail = getAdminEmail('manager');
        sendTemplatedEmail('Ticket Escalated', $managerEmail, [
            'ticket_id' => $params['tid'],
            'original_message' => $message
        ]);
        
        // Add internal note
        addTicketNote($ticketId, 'Ticket escalated by auto-detection');
    }
    
    // 2. Close if client says resolved
    if (preg_match('/\b(resolved|closed?|fixed|solved|thanks?)\b/i', $message)) {
        update_query('tbltickets', [
            'status' => 'Resolved'
        ], ['id' => $ticketId]);
    }
    
    return $params;
});
```

## Satisfaction Tracking

```php
<?php
add_hook('TicketReply', 1, function(array $params) {
    // Track response time for SLA compliance
    $ticketId = (int)$params['ticketid'];
    
    // Get ticket creation time
    $ticket = getTicketInfo($ticketId);
    $createdAt = strtotime($ticket['created']);
    $repliedAt = time();
    $responseTime = $repliedAt - $createdAt;
    
    // Log response time
    insert_query('tbl_ticket_response_times', [
        'ticket_id' => $ticketId,
        'responder_type' => !empty($params['admin']) ? 'admin' : 'client',
        'responder_id' => !empty($params['admin']) ? $params['admin'] : $params['userid'],
        'response_time_seconds' => $responseTime,
        'created_at' => date('Y-m-d H:i:s')
    ]);
    
    // Notify if SLA breach likely
    if ($responseTime > 3600) { // Over 1 hour
        updateTicketCustomField($ticketId, 'Response Delay', 'Yes');
        
        sendAdminNotification('system', [
            'subject' => 'Slow Response Alert',
            'message' => "Ticket {$params['tid']} response time: " . 
                        round($responseTime/60) . " minutes"
        ]);
    }
    
    return $params;
});
```

## Third-Party Integrations

```php
<?php
add_hook('TicketReply', 1, function(array $params) {
    // 1. Sync to external helpdesk (e.g., Zendesk)
    syncTicketReplyToZendesk($params['ticketid'], [
        'message' => $params['message'],
        'author_type' => !empty($params['admin']) ? 'agent' : 'end-user',
        'author_id' => !empty($params['admin']) ? $params['admin'] : $params['userid'],
        'timestamp' => date('c')
    ]);
    
    // 2. Post to CRM for client context
    if ($params['userid'] > 0) {
        updateCRMticketActivity($params['userid'], [
            'ticket_id' => $params['tid'],
            'action' => 'reply_added',
            'message_preview' => substr($params['message'], 0, 100)
        ]);
    }
    
    // 3. Update knowledge base articles
    if (stripos($params['message'], 'how do i') !== false) {
        flagForKBArticle($params['ticketid'], $params['message']);
    }
    
    return $params;
});
```

## Use Cases

- **Auto-Responses**: Reply to common questions
- **Escalation**: Detect and escalate urgent tickets
- **SLA Tracking**: Monitor response times
- **Integrations**: Sync with external helpdesks
- **Satisfaction Metrics**: Track client feedback

## Notes

- Fires for both client and admin replies
- Can modify ticket status through the reply
- Use with `TicketClose` for complete lifecycle
- Consider rate limiting for spam prevention

## Related Hooks

- `TicketOpen` - When ticket is created
- `TicketClose` - When ticket is closed
- `AdminLogin` - For admin context

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Support System Configuration](../whmcs-support-setup.md)