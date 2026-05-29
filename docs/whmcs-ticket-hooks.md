# WHMCS Ticket Hooks

## Overview

Ticket hooks allow customization of support ticket operations including creation, updates, closures, and automated responses.

## Available Ticket Hooks

### Ticket Created Hook

```php
<?php
// Triggered when a new support ticket is created
add_hook('TicketOpen', 1, function(array $vars) {
    $ticketId = $vars['ticket_id'];
    $userId = $vars['user_id'];
    $subject = $vars['subject'];
    $departmentId = $vars['deptid'];
    
    // Auto-assign based on keywords
    $assignedAdmin = autoAssignTicket($subject, $departmentId);
    
    // Check for duplicate tickets
    $duplicates = findDuplicateTickets($userId, $subject);
    if (!empty($duplicates)) {
        notifyDuplicates($ticketId, $duplicates);
    }
    
    // Apply auto-responses
    applyAutoResponse($ticketId, $subject);
    
    // Route to appropriate team
    routeTicket($ticketId, $departmentId);
    
    // Log ticket creation
    logTicketCreation($ticketId, $vars);
    
    return ['success' => true, 'ticket_id' => $ticketId];
});

function autoAssignTicket(string $subject, int $deptId): ?int
{
    $keywords = [
        'billing' => ['invoice', 'payment', 'refund', 'charge'],
        'technical' => ['error', 'not working', 'bug', 'crash'],
        'sales' => ['upgrade', 'quote', 'pricing', 'trial'],
    ];
    
    $lowerSubject = strtolower($subject);
    
    foreach ($keywords as $adminType => $words) {
        foreach ($words as $word) {
            if (strpos($lowerSubject, $word) !== false) {
                return getAvailableAdmin($adminType);
            }
        }
    }
    
    return null;
}
```

### Ticket Reply Hook

```php
<?php
// Triggered when a ticket receives a reply
add_hook('TicketReply', 1, function(array $vars) {
    $ticketId = $vars['ticket_id'];
    $replyId = $vars['reply_id'];
    $isClient = $vars['is_client'];
    $message = $vars['message'];
    
    // Update ticket status
    updateTicketStatus($ticketId);
    
    // Parse attachments
    scanAttachments($ticketId);
    
    // Check for keywords
    checkForKeywords($ticketId, $message);
    
    // Update satisfaction survey
    resetSatisfactionSurvey($ticketId);
    
    // Log reply
    logTicketReply($ticketId, $replyId, $isClient);
    
    return ['success' => true];
});

function scanAttachments(int $ticketId): void
{
    $attachments = Capsule::table('tblticketattachments')
        ->where('ticketid', $ticketId)
        ->get();
    
    foreach ($attachments as $attachment) {
        // Scan for malware
        if (isSuspiciousFile($attachment->filename)) {
            quarantineAttachment($attachment->id);
            addTicketNote($ticketId, 'Suspicious attachment detected and quarantined');
        }
    }
}
```

### Ticket Closed Hook

```php
<?php
// Triggered when a ticket is closed
add_hook('TicketClose', 1, function(array $vars) {
    $ticketId = $vars['ticket_id'];
    $adminId = $vars['admin_id'] ?? 0;
    $reason = $vars['reason'] ?? '';
    
    // Send closure survey
    sendSatisfactionSurvey($ticketId);
    
    // Archive ticket data
    archiveTicket($ticketId);
    
    // Update metrics
    updateTicketMetrics($ticketId);
    
    // Send closure notification
    sendClosureNotification($ticketId);
    
    // Log closure
    logTicketClosure($ticketId, $adminId, $reason);
    
    return ['success' => true];
});

function sendSatisfactionSurvey(int $ticketId): void
{
    $ticket = Capsule::table('tbtickets')->where('id', $ticketId)->first();
    
    send_email([
        'type' => 'support',
        'id' => getSurveyTemplateId(),
        'customvars' => [
            'ticket_id' => $ticketId,
            'ticket_number' => $ticket->tid,
            'survey_url' => getSurveyUrl($ticketId),
        ],
    ], $ticket->userid);
}
```

### Ticket Merged Hook

```php
<?php
// Triggered when tickets are merged
add_hook('TicketMerge', 1, function(array $vars) {
    $sourceTicketId = $vars['source_ticket_id'];
    $targetTicketId = $vars['target_ticket_id'];
    
    // Log merge
    logMerge($sourceTicketId, $targetTicketId);
    
    // Update references
    updateReferences($sourceTicketId, $targetTicketId);
    
    return ['success' => true];
});
```

### Ticket Priority Changed Hook

```php
<?php
// Triggered when ticket priority changes
add_hook('TicketPriorityChange', 1, function(array $vars) {
    $ticketId = $vars['ticket_id'];
    $oldPriority = $vars['old_priority'];
    $newPriority = $vars['new_priority'];
    
    // Escalate if priority increased
    if ($newPriority === 'High' || $newPriority === 'Emergency') {
        escalateTicket($ticketId);
    }
    
    // Update SLA
    updateSlaDeadline($ticketId);
    
    // Notify stakeholders
    notifyPriorityChange($ticketId, $oldPriority, $newPriority);
    
    return ['success' => true];
});
```

### Pre-Ticket Creation Hook

```php
<?php
// Runs before ticket is created - validation
add_hook('PreTicketCreate', 1, function(array $vars) {
    $userId = $vars['user_id'];
    $subject = $vars['subject'];
    $message = $vars['message'];
    $departmentId = $vars['deptid'];
    
    $errors = [];
    
    // Check for spam
    if (isSpamContent($subject, $message)) {
        $errors[] = 'Ticket flagged as spam';
    }
    
    // Validate department access
    if (!hasDepartmentAccess($userId, $departmentId)) {
        $errors[] = 'No access to this department';
    }
    
    // Check rate limiting
    if (exceedsTicketRateLimit($userId)) {
        $errors[] = 'Rate limit exceeded. Please wait before creating more tickets.';
    }
    
    if (!empty($errors)) {
        return [
            'abort' => true,
            'error_msg' => implode('. ', $errors),
        ];
    }
    
    return ['abort' => false];
});
```

## Comprehensive Ticket Handler

```php
<?php
class TicketHookHandler {
    
    public function register(): void
    {
        add_hook('TicketOpen', 1, [$this, 'handleCreated']);
        add_hook('TicketReply', 1, [$this, 'handleReply']);
        add_hook('TicketClose', 1, [$this, 'handleClosed']);
        add_hook('TicketMerge', 1, [$this, 'handleMerge']);
        add_hook('TicketPriorityChange', 1, [$this, 'handlePriorityChange']);
        add_hook('PreTicketCreate', 1, [$this, 'handlePreCreate']);
    }
    
    public function handleCreated(array $vars): array
    {
        $this->autoAssign($vars['ticket_id'], $vars['subject']);
        $this->checkDuplicates($vars);
        $this->applyAutoResponse($vars['ticket_id']);
        $this->logCreation($vars);
        return ['success' => true];
    }
    
    public function handleReply(array $vars): array
    {
        $this->updateStatus($vars['ticket_id']);
        $this->scanAttachments($vars['ticket_id']);
        $this->resetSurvey($vars['ticket_id']);
        return ['success' => true];
    }
    
    public function handleClosed(array $vars): array
    {
        $this->sendSurvey($vars['ticket_id']);
        $this->archive($vars['ticket_id']);
        $this->updateMetrics($vars['ticket_id']);
        return ['success' => true];
    }
    
    public function handleMerge(array $vars): array
    {
        $this->logMerge($vars);
        $this->updateReferences($vars);
        return ['success' => true];
    }
    
    public function handlePriorityChange(array $vars): array
    {
        if (in_array($vars['new_priority'], ['High', 'Emergency'])) {
            $this->escalate($vars['ticket_id']);
        }
        $this->notifyPriorityChange($vars);
        return ['success' => true];
    }
    
    public function handlePreCreate(array $vars): array
    {
        $errors = [];
        
        if ($this->isSpam($vars)) {
            $errors[] = 'Spam detected';
        }
        
        if ($this->exceedsRateLimit($vars['user_id'])) {
            $errors[] = 'Rate limit exceeded';
        }
        
        if (!empty($errors)) {
            return ['abort' => true, 'error_msg' => implode('. ', $errors)];
        }
        
        return ['abort' => false];
    }
    
    private function autoAssign(int $ticketId, string $subject): ?int
    {
        // Auto-assignment logic
        return null;
    }
    
    private function checkDuplicates(array $vars): void
    {
        // Check for duplicates
    }
    
    private function applyAutoResponse(int $ticketId): void
    {
        // Apply auto-response
    }
    
    private function logCreation(array $vars): void
    {
        Capsule::table('mod_ticket_events')->insert([
            'ticket_id' => $vars['ticket_id'],
            'event' => 'created',
            'user_id' => $vars['user_id'],
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function updateStatus(int $ticketId): void
    {
        // Update status
    }
    
    private function scanAttachments(int $ticketId): void
    {
        // Scan attachments
    }
    
    private function resetSurvey(int $ticketId): void
    {
        // Reset satisfaction survey
    }
    
    private function sendSurvey(int $ticketId): void
    {
        // Send survey
    }
    
    private function archive(int $ticketId): void
    {
        // Archive ticket
    }
    
    private function updateMetrics(int $ticketId): void
    {
        // Update metrics
    }
    
    private function logMerge(array $vars): void
    {
        // Log merge
    }
    
    private function updateReferences(array $vars): void
    {
        // Update references
    }
    
    private function escalate(int $ticketId): void
    {
        // Escalate ticket
    }
    
    private function notifyPriorityChange(array $vars): void
    {
        // Notify about priority change
    }
    
    private function isSpam(array $vars): bool
    {
        return false;
    }
    
    private function exceedsRateLimit(int $userId): bool
    {
        return false;
    }
}

$handler = new TicketHookHandler();
$handler->register();
```

## Best Practices

1. **Don't create infinite loops** - Avoid updating tickets in their own hooks
2. **Queue heavy operations** - Process attachments asynchronously
3. **Log everything** - Maintain audit trail
4. **Handle failures** - Use try-catch blocks
5. **Respect rate limits** - Implement per-user limits

## Related Documentation

- [WHMCS Client Hooks](/docs/whmcs-client-hooks.md)
- [WHMCS Cron Hooks](/docs/whmcs-cron-hooks.md)