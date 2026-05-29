# WHMCS TicketClose Hook Reference

## Overview

The `TicketClose` hook fires when a support ticket is closed in WHMCS. This hook triggers when an admin closes a ticket or when the system automatically closes resolved tickets.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `ticketid` | int | The unique ticket ID |
| `tid` | string | The ticket tracking ID |
| `userid` | int | The client ID |
| `subject` | string | Ticket subject |
| `status` | string | Final status (Closed/Resolved) |
| `admin` | string | Admin who closed (empty if auto-closed) |
| `model` | object | The Ticket model instance |

## Example Implementation

```php
<?php
add_hook('TicketClose', 1, function(array $params) {
    // Log ticket closure
    logActivity("Ticket {$params['tid']} closed by " . 
        (!empty($params['admin']) ? $params['admin'] : 'system'));
    
    // Calculate ticket duration
    $ticket = getTicketInfo($params['ticketid']);
    $duration = strtotime($ticket['lastreply']) - strtotime($ticket['created']);
    
    return $params;
});
```

## Satisfaction Survey

```php
<?php
add_hook('TicketClose', 1, function(array $params) {
    // Send satisfaction survey
    if ($params['userid'] > 0) {
        $client = getClientsDetails($params['userid']);
        
        // Generate unique survey token
        $surveyToken = generateSurveyToken($params['ticketid'], $params['userid']);
        $surveyUrl = "https://billing.example.com/survey?token={$surveyToken}";
        
        // Send survey email
        sendTemplatedEmail('Support Survey', $client['email'], [
            'ticket_id' => $params['tid'],
            'survey_url' => $surveyUrl,
            'subject' => $params['subject']
        ]);
        
        // Store survey status
        update_query('tbltickets', [
            'survey_sent' => '1'
        ], ['id' => $params['ticketid']]);
    }
    
    return $params;
});
```

## Stats and Reporting

```php
<?php
add_hook('TicketClose', 1, function(array $params) {
    $ticketId = (int)$params['ticketid'];
    
    // Get ticket details for stats
    $ticket = getTicketInfo($ticketId);
    $created = strtotime($ticket['created']);
    $closed = time();
    $duration = $closed - $created;
    
    // Calculate business hours only
    $businessHours = calculateBusinessHours($created, $closed);
    
    // 1. Store closure statistics
    insert_query('tbl_ticket_stats', [
        'ticket_id' => $ticketId,
        'department_id' => $ticket['did'],
        'assigned_admin' => $ticket['admin'],
        'priority' => $ticket['urgency'],
        'created_at' => $ticket['created'],
        'closed_at' => date('Y-m-d H:i:s'),
        'total_duration_seconds' => $duration,
        'business_hours' => $businessHours,
        'reply_count' => getTicketReplyCount($ticketId)
    ]);
    
    // 2. Update admin stats
    if (!empty($params['admin'])) {
        incrementAdminStat($params['admin'], 'tickets_closed');
    }
    
    // 3. Track average response time
    updateGlobalStat('avg_ticket_duration', $duration);
    
    return $params;
});
```

## Follow-up and Cleanup

```php
<?php
add_hook('TicketClose', 1, function(array $params) {
    // 1. Schedule follow-up for high-priority tickets
    if ($params['model']->urgency === 'Urgent') {
        scheduleFollowUp([
            'ticket_id' => $params['ticketid'],
            'scheduled_for' => date('Y-m-d H:i:s', strtotime('+7 days')),
            'reminder_type' => 'satisfaction_check'
        ]);
    }
    
    // 2. Archive ticket for retention
    archiveTicketForRetention($params['ticketid'], '+90 days');
    
    // 3. Update client satisfaction if survey completed
    $surveyResult = getSurveyResult($params['ticketid']);
    if ($surveyResult) {
        updateClientSatisfactionScore($params['userid'], $surveyResult['rating']);
    }
    
    // 4. Remove escalation flags
    clearEscalationFlags($params['ticketid']);
    
    // 5. Update external helpdesk
    syncTicketStatusToZendesk($params['tid'], 'closed');
    
    return $params;
});
```

## Auto-Reopen Rules

```php
<?php
add_hook('TicketClose', 1, function(array $params) {
    $message = $params['model']->getLastReply();
    
    // Check if client added reply before closure
    if (strtotime($message['date']) > strtotime($params['model']->lastreply)) {
        // Client replied - reopen ticket
        update_query('tbltickets', [
            'status' => 'Customer-Reply'
        ], ['id' => $params['ticketid']]);
        
        logActivity("Ticket {$params['tid']} auto-reopened due to client reply");
        return ['prevent_close' => true];
    }
    
    // Check for negative sentiment
    $sentiment = analyzeSentiment($params['model']->getAllReplies());
    if ($sentiment === 'negative' && $params['model']->urgency !== 'Urgent') {
        // Keep ticket open for manager review
        update_query('tbltickets', [
            'status' => 'On Hold',
            'flag' => getManagerAdminId()
        ], ['id' => $params['ticketid']]);
        
        addTicketNote($params['ticketid'], 'Auto-held for negative sentiment review');
        
        sendAdminNotification('system', [
            'subject' => 'Negative Feedback Detected',
            'message' => "Ticket {$params['tid']} held for review"
        ]);
    }
    
    return $params;
});
```

## Use Cases

- **Satisfaction Surveys**: Request client feedback
- **Statistics**: Track ticket metrics
- **Follow-ups**: Schedule future contact
- **Integrations**: Sync with external systems
- **Auto-Reopen**: Detect late replies

## Notes

- Runs after ticket is closed
- Can prevent closure by returning specific values
- Survey sending should respect user preferences
- Consider GDPR for ticket data retention

## Related Hooks

- `TicketOpen` - When ticket is opened
- `TicketReply` - When ticket is replied to
- `DailyCronJob` - For automated closing

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Support System Configuration](../whmcs-support-setup.md)