# WHMCS Ticket Functions

Complete reference for support ticket management functions in WHMCS.

## Overview

WHMCS provides comprehensive ticket management including creation, updates, replies, and status management.

## Ticket CRUD Operations

### createTicket()

Creates a new support ticket.

```php
/**
 * Create a new ticket
 * 
 * @param array $data Ticket data
 * @return int Ticket ID
 */
function createTicket(array $data): int
{
    $ticketMask = generateTicketMask();
    
    return Capsule::table('tbltickets')->insertGetId([
        'tid' => $ticketMask,
        'userid' => $data['clientid'] ?? 0,
        'contactid' => $data['contactid'] ?? 0,
        'did' => $data['departmentid'] ?? 0,
        'subject' => $data['subject'],
        'status' => $data['status'] ?? 'Open',
        'priority' => $data['priority'] ?? 'Medium',
        'admin' => $data['assignedto'] ?? '',
        'lastreply' => date('Y-m-d H:i:s'),
        'createtime' => date('Y-m-d H:i:s'),
        'urgency' => $data['urgency'] ?? 0,
        'flag' => $data['flaggedto'] ?? 0,
    ]);
}
```

**Example:**
```php
$ticketId = createTicket([
    'clientid' => 123,
    'departmentid' => 1,
    'subject' => 'Unable to access my account',
    'priority' => 'High',
    'status' => 'Open'
]);
```

### getTicket()

Retrieves a ticket by ID.

```php
/**
 * Get ticket by ID
 * 
 * @param int $ticketId Ticket ID
 * @return array|null Ticket data
 */
function getTicket(int $ticketId): ?array
{
    $result = Capsule::table('tbltickets')
        ->where('id', $ticketId)
        ->first();
    
    return $result ? (array) $result : null;
}
```

### getTicketByMask()

Retrieves a ticket by ticket ID (mask).

```php
/**
 * Get ticket by ticket mask ID
 * 
 * @param string $ticketMask Ticket mask (e.g., ABC-123-45678)
 * @return array|null Ticket data
 */
function getTicketByMask(string $ticketMask): ?array
{
    $result = Capsule::table('tbltickets')
        ->where('tid', $ticketMask)
        ->first();
    
    return $result ? (array) $result : null;
}
```

**Example:**
```php
$ticket = getTicketByMask('ABC-123-45678');

if ($ticket) {
    echo "Subject: {$ticket['subject']}";
    echo "Status: {$ticket['status']}";
}
```

### getTickets()

Retrieves tickets with filtering.

```php
/**
 * Get tickets with filters
 * 
 * @param array $filters Filter options
 * @param int $limit Number of records
 * @param int $offset Starting offset
 * @return array Tickets
 */
function getTickets(array $filters = [], int $limit = 50, int $offset = 0): array
{
    $query = Capsule::table('tbltickets')
        ->select('tbltickets.*', 'tblclients.firstname', 'tblclients.lastname', 
                 'tbldepartments.name as department_name')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tbltickets.userid')
        ->leftJoin('tbldepartments', 'tbldepartments.id', '=', 'tbltickets.did')
        ->orderBy('tbltickets.id', 'desc');
    
    if (!empty($filters['clientId'])) {
        $query->where('tbltickets.userid', $filters['clientId']);
    }
    
    if (!empty($filters['status'])) {
        $query->where('tbltickets.status', $filters['status']);
    }
    
    if (!empty($filters['departmentId'])) {
        $query->where('tbltickets.did', $filters['departmentId']);
    }
    
    if (!empty($filters['priority'])) {
        $query->where('tbltickets.priority', $filters['priority']);
    }
    
    if (!empty($filters['assignedTo'])) {
        $query->where('tbltickets.admin', $filters['assignedTo']);
    }
    
    return $query->limit($limit)->offset($offset)->get()->toArray();
}
```

**Example:**
```php
// Get all open tickets
$openTickets = getTickets(['status' => 'Open']);

// Get high priority tickets assigned to admin
$urgent = getTickets([
    'priority' => 'High',
    'assignedTo' => 'admin_user'
]);
```

### updateTicket()

Updates an existing ticket.

```php
/**
 * Update a ticket
 * 
 * @param int $ticketId Ticket ID
 * @param array $data Updated data
 * @return bool Success status
 */
function updateTicket(int $ticketId, array $data): bool
{
    $data['updated_at'] = date('Y-m-d H:i:s');
    
    return Capsule::table('tbltickets')
        ->where('id', $ticketId)
        ->update($data) > 0;
}
```

**Example:**
```php
updateTicket(1234, [
    'status' => 'In Progress',
    'priority' => 'Medium',
    'admin' => 'admin_user'
]);
```

## Ticket Replies

### addTicketReply()

Adds a reply to a ticket.

```php
/**
 * Add reply to ticket
 * 
 * @param int $ticketId Ticket ID
 * @param array $data Reply data
 * @return int Reply ID
 */
function addTicketReply(int $ticketId, array $data): int
{
    $replyId = Capsule::table('tblticketreplies')->insertGetId([
        'tid' => $ticketId,
        'userid' => $data['userid'] ?? 0,
        'admin' => $data['admin'] ?? '',
        'name' => $data['name'] ?? '',
        'email' => $data['email'] ?? '',
        'date' => date('Y-m-d H:i:s'),
        'message' => $data['message'],
        'clientread' => $data['clientread'] ?? 1,
        'adminread' => $data['adminread'] ?? 0,
    ]);
    
    // Update ticket last reply time
    Capsule::table('tbltickets')
        ->where('id', $ticketId)
        ->update(['lastreply' => date('Y-m-d H:i:s')]);
    
    return $replyId;
}
```

**Example:**
```php
// Admin reply
addTicketReply(1234, [
    'admin' => 'admin_user',
    'message' => 'Thank you for contacting us. We are investigating this issue.',
    'adminread' => 1
]);

// Client reply
addTicketReply(1234, [
    'userid' => 123,
    'message' => 'Any update on this issue?',
    'clientread' => 1
]);
```

### getTicketReplies()

Gets all replies for a ticket.

```php
/**
 * Get ticket replies
 * 
 * @param int $ticketId Ticket ID
 * @return array Replies
 */
function getTicketReplies(int $ticketId): array
{
    return Capsule::table('tblticketreplies')
        ->where('tid', $ticketId)
        ->orderBy('date', 'asc')
        ->get()
        ->toArray();
}
```

## Ticket Notes

### addTicketNote()

Adds a note to a ticket (internal).

```php
/**
 * Add note to ticket
 * 
 * @param int $ticketId Ticket ID
 * @param string $note Note content
 * @param string $admin Admin who added note
 * @return int Note ID
 */
function addTicketNote(int $ticketId, string $note, string $admin = ''): int
{
    return Capsule::table('tblticketnotes')->insertGetId([
        'tid' => $ticketId,
        'admin' => $admin,
        'date' => date('Y-m-d H:i:s'),
        'message' => $note,
        'visibility' => 'internal',
    ]);
}
```

### getTicketNotes()

Gets all notes for a ticket.

```php
/**
 * Get ticket notes
 * 
 * @param int $ticketId Ticket ID
 * @return array Notes
 */
function getTicketNotes(int $ticketId): array
{
    return Capsule::table('tblticketnotes')
        ->where('tid', $ticketId)
        ->orderBy('date', 'asc')
        ->get()
        ->toArray();
}
```

## Ticket Status Management

```php
/**
 * Change ticket status
 * 
 * @param int $ticketId Ticket ID
 * @param string $status New status
 * @return bool Success status
 */
function changeTicketStatus(int $ticketId, string $status): bool
{
    $validStatuses = ['Open', 'Answered', 'Customer Reply', 'Closed', 'Merged'];
    
    if (!in_array($status, $validStatuses)) {
        return false;
    }
    
    return updateTicket($ticketId, [
        'status' => $status,
        'lastreply' => date('Y-m-d H:i:s')
    ]);
}

/**
 * Close a ticket
 * 
 * @param int $ticketId Ticket ID
 * @param string $reason Closing reason
 * @return bool Success status
 */
function closeTicket(int $ticketId, string $reason = ''): bool
{
    $result = changeTicketStatus($ticketId, 'Closed');
    
    if ($result) {
        addTicketNote($ticketId, "Ticket closed: {$reason}", 'system');
        logActivity("Ticket #{$ticketId} closed", 0);
    }
    
    return $result;
}
```

## Ticket Assignment

```php
/**
 * Assign ticket to admin
 * 
 * @param int $ticketId Ticket ID
 * @param string $adminId Admin user ID
 * @return bool Success status
 */
function assignTicket(int $ticketId, string $adminId): bool
{
    return updateTicket($ticketId, [
        'admin' => $adminId,
        'flag' => 0 // Clear flag when assigned
    ]);
}

/**
 * Flag ticket for attention
 * 
 * @param int $ticketId Ticket ID
 * @param string $adminId Admin to flag to
 * @return bool Success status
 */
function flagTicket(int $ticketId, string $adminId): bool
{
    return updateTicket($ticketId, [
        'flag' => $adminId
    ]);
}
```

## Ticket Merging

```php
/**
 * Merge tickets
 * 
 * @param int $targetTicketId Target ticket ID (will remain)
 * @param int $sourceTicketId Source ticket ID (will be merged)
 * @return bool Success status
 */
function mergeTickets(int $targetTicketId, int $sourceTicketId): bool
{
    $source = getTicket($sourceTicketId);
    
    if (!$source) {
        return false;
    }
    
    // Move all replies to target
    Capsule::table('tblticketreplies')
        ->where('tid', $sourceTicketId)
        ->update(['tid' => $targetTicketId]);
    
    // Move all notes
    Capsule::table('tblticketnotes')
        ->where('tid', $sourceTicketId)
        ->update(['tid' => $targetTicketId]);
    
    // Update source status
    updateTicket($sourceTicketId, [
        'status' => 'Merged',
        'merged_ticket_id' => $targetTicketId
    ]);
    
    // Update target last reply
    updateTicket($targetTicketId, [
        'lastreply' => date('Y-m-d H:i:s')
    ]);
    
    return true;
}
```

## Ticket Deletion

```php
/**
 * Delete a ticket
 * 
 * @param int $ticketId Ticket ID
 * @param bool $hardDelete Permanently delete
 * @return bool Success status
 */
function deleteTicket(int $ticketId, bool $hardDelete = false): bool
{
    if ($hardDelete) {
        // Delete replies and notes
        Capsule::table('tblticketreplies')->where('tid', $ticketId)->delete();
        Capsule::table('tblticketnotes')->where('tid', $ticketId)->delete();
        
        return Capsule::table('tbltickets')
            ->where('id', $ticketId)
            ->delete() > 0;
    }
    
    // Soft delete - mark as deleted
    return updateTicket($ticketId, [
        'status' => 'Deleted',
        'subject' => '[DELETED] ' . getTicket($ticketId)['subject']
    ]);
}
```

## Ticket Notifications

```php
/**
 * Send ticket notification
 * 
 * @param int $ticketId Ticket ID
 * @param string $type Notification type
 * @param array $data Additional data
 * @return bool Success status
 */
function sendTicketNotification(int $ticketId, string $type, array $data = []): bool
{
    $ticket = getTicket($ticketId);
    
    if (!$ticket) {
        return false;
    }
    
    $client = getClient($ticket['userid']);
    $department = getDepartment($ticket['did']);
    
    $emailData = [
        'ticket_id' => $ticket['id'],
        'ticket_mask' => $ticket['tid'],
        'subject' => $ticket['subject'],
        'priority' => $ticket['priority'],
        'status' => $ticket['status'],
        'client_name' => $client ? "{$client['firstname']} {$client['lastname']}" : 'Guest',
        'department' => $department['name'] ?? '',
    ];
    
    switch ($type) {
        case 'new_ticket':
            // Notify admins
            return notifyAdmins('new_ticket', $emailData);
        
        case 'new_reply':
            // Notify client
            return notifyClient($ticket, 'ticket_reply', $emailData);
        
        case 'ticket_assigned':
            // Notify assigned admin
            return notifyAdmin($ticket['admin'], 'ticket_assigned', $emailData);
        
        case 'ticket_flagged':
            // Notify flagged admin
            return notifyAdmin($ticket['flag'], 'ticket_flagged', $emailData);
    }
    
    return false;
}
```

## Ticket Searching

```php
/**
 * Search tickets
 * 
 * @param string $term Search term
 * @param array $options Search options
 * @return array Matching tickets
 */
function searchTickets(string $term, array $options = []): array
{
    $query = Capsule::table('tbltickets')
        ->select('tbltickets.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tbltickets.userid');
    
    // Search in subject and message
    $query->where(function($q) use ($term) {
        $q->where('tbltickets.subject', 'like', '%' . $term . '%')
          ->orWhere('tbltickets.tid', 'like', '%' . $term . '%');
    });
    
    // Include replies in search
    $replyMatches = Capsule::table('tblticketreplies')
        ->select('tid')
        ->where('message', 'like', '%' . $term . '%')
        ->get()
        ->pluck('tid')
        ->toArray();
    
    if (!empty($replyMatches)) {
        $query->orWhereIn('tbltickets.id', $replyMatches);
    }
    
    if (!empty($options['status'])) {
        $query->where('tbltickets.status', $options['status']);
    }
    
    return $query->limit(50)->get()->toArray();
}
```

## Ticket Statistics

```php
/**
 * Get ticket statistics
 * 
 * @param array $filters Filter options
 * @return array Statistics
 */
function getTicketStatistics(array $filters = []): array
{
    $query = Capsule::table('tbltickets');
    
    if (!empty($filters['departmentId'])) {
        $query->where('did', $filters['departmentId']);
    }
    
    $stats = $query->selectRaw("
        COUNT(*) as total,
        COUNT(CASE WHEN status = 'Open' THEN 1 END) as open,
        COUNT(CASE WHEN status = 'Answered' THEN 1 END) as answered,
        COUNT(CASE WHEN status = 'Customer Reply' THEN 1 END) as awaiting,
        COUNT(CASE WHEN status = 'Closed' THEN 1 END) as closed,
        COUNT(CASE WHEN priority = 'High' AND status != 'Closed' THEN 1 END) as high_priority
    ")->first();
    
    return (array) $stats;
}
```

## Ticket Canned Responses

```php
/**
 * Get canned responses
 * 
 * @param int $departmentId Department ID
 * @return array Canned responses
 */
function getCannedResponses(int $departmentId = 0): array
{
    $query = Capsule::table('tblcannedresponses')
        ->where('parentid', 0);
    
    if ($departmentId > 0) {
        $query->where(function($q) use ($departmentId) {
            $q->where('did', $departmentId)
              ->orWhere('did', 0);
        });
    }
    
    return $query->orderBy('name', 'asc')->get()->toArray();
}

/**
 * Insert canned response into ticket
 * 
 * @param int $ticketId Ticket ID
 * @param int $cannedResponseId Canned response ID
 * @return string Response text
 */
function insertCannedResponse(int $ticketId, int $cannedResponseId): string
{
    $canned = Capsule::table('tblcannedresponses')
        ->where('id', $cannedResponseId)
        ->first();
    
    if (!$canned) {
        return '';
    }
    
    // Replace placeholders
    $ticket = getTicket($ticketId);
    $message = $canned->message;
    
    $message = str_replace(['{client_name}', '{ticket_id}', '{ticket_subject}'], [
        getClient($ticket['userid'])['firstname'] ?? '',
        $ticket['tid'],
        $ticket['subject']
    ], $message);
    
    return $message;
}
```

## Best Practices

1. **Use ticket masks** - Never expose internal ticket IDs to users
2. **Track response times** - Monitor SLA compliance
3. **Use departments** - Route tickets appropriately
4. **Implement escalation** - Automatic priority changes based on time
5. **Keep audit trail** - Log all ticket actions
6. **Merge duplicates** - Consolidate related tickets

## Related Functions

- [whmcs-functions-email.md](whmcs-functions-email.md) - Email notifications
- [whmcs-schema-tickets.md](whmcs-schema-tickets.md) - Ticket database schema
- [whmcs-integration-email.md](whmcs-integration-email.md) - Email integration