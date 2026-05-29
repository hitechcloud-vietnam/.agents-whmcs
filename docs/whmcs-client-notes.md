# WHMCS Client Notes

## Overview

Client notes in WHMCS allow administrators to add internal annotations and information about clients. Notes are visible only to staff and help track client interactions, preferences, and important information that shouldn't be shared with clients.

## Notes Configuration

### Basic Settings

**Configuration > Support > Notes Settings**

```php
[
    'allow_notes' => true,
    'max_note_length' => 5000,
    'allow_attachments' => false,
    'note_visibility' => 'internal'     // internal, restricted
]
```

## Adding Notes

### Manual Note Entry

**Admin: Clients > Select Client > Notes > Add Note**

```php
// New client note
[
    'userid' => 123,
    'note' => 'Customer prefers email communication over phone.',
    'created_by' => 'admin_id',
    'created_at' => '2024-05-15 10:30:00',
    'sticky' => false,
    'visibility' => 'internal'
]
```

### Note Fields

| Field | Description |
|-------|-------------|
| note | Note content (required) |
| created_by | Admin who created note |
| created_at | Creation timestamp |
| modified_at | Last modification time |
| modified_by | Last modifier admin |
| sticky | Pin to top of notes list |
| visibility | internal, restricted |

## Note Types

### General Notes

```php
// Standard client notes
[
    'type' => 'general',
    'note' => 'Customer has been with us for 5 years.',
    'category' => 'General'
]
```

### Interaction Notes

```php
// Notes about client interactions
[
    'type' => 'interaction',
    'note' => 'Called customer regarding overdue invoice. 
               They promised payment by end of week.',
    'category' => 'Phone Call',
    'reference_id' => 'ticket_1234'
]
```

### Internal Notes

```php
// Highly sensitive notes
[
    'type' => 'internal',
    'note' => 'Flagged for potential fraud - verify identity 
               before processing large orders.',
    'visibility' => 'restricted',
    'access_roles' => ['admin', 'manager']
]
```

### Sticky Notes

```php
// Pinned important notes
[
    'sticky' => true,
    'note' => 'VIP customer - escalate issues immediately.',
    'priority' => 'high'
]
```

## Note Management

### Editing Notes

```php
// Update existing note
[
    'note_id' => 789,
    'note' => 'Updated note content.',
    'modified_by' => 'admin_id',
    'modified_at' => '2024-05-15 14:30:00'
]
```

### Deleting Notes

```php
// Delete note
[
    'note_id' => 789,
    'deleted_by' => 'admin_id',
    'deleted_at' => '2024-05-15 14:30:00',
    'soft_delete' => true                 // Keep for audit
]
```

### Note Organization

```php
// Organize notes
[
    'pin_to_top' => true,               // Sticky notes
    'color_code' => 'red',              // Color coding
    'add_tags' => ['vip', 'enterprise'] // Tagging
]
```

## Note Visibility

### Access Control

```php
// Note visibility levels
[
    'all_staff' => true,                // Visible to all admin users
    'creator_only' => false,            // Only creator can see
    'role_based' => [
        'admin' => true,
        'billing' => true,
        'support' => true,
        'restricted' => false
    ]
]
```

### Restricted Notes

```php
// Notes only visible to certain roles
[
    'visibility' => 'restricted',
    'allowed_roles' => ['admin', 'manager'],
    'allowed_users' => [1, 2, 3]
]
```

## Note Search

### Search Functionality

```php
// Search client notes
[
    'search_term' => 'payment',
    'userid' => 123,
    'date_from' => '2024-01-01',
    'date_to' => '2024-05-31',
    'created_by' => null,
    'sticky_only' => false
]
```

### Global Note Search

**Admin: Search > Notes**

```php
// Search across all clients
[
    'term' => 'fraud',
    'search_all_clients' => true,
    'include_deleted' => false
]
```

## Note Integration

### Link Notes to Tickets

```php
// Associate note with ticket
[
    'note_id' => 789,
    'ticket_id' => 1234,
    'auto_link' => true
]
```

### Link Notes to Invoices

```php
// Associate note with invoice
[
    'note_id' => 789,
    'invoice_id' => 5678,
    'reference_type' => 'invoice'
]
```

## Note Display

### Client Summary

**Admin: Clients > View Client > Notes**

```php
// Display notes for client
[
    ['date' => '2024-05-15', 'note' => '...', 'created_by' => 'Admin'],
    ['date' => '2024-05-10', 'note' => '...', 'created_by' => 'Support'],
    ['date' => '2024-05-01', 'note' => '...', 'created_by' => 'Admin', 'sticky' => true]
]
```

### Note Formatting

```smarty
{foreach $client.notes as $note}
    <div class="note {if $note.sticky}sticky{/if}">
        <div class="note-meta">
            {$note.created_at|date_format}
            by {$note.created_by_name}
        </div>
        <div class="note-content">
            {$note.note}
        </div>
    </div>
{/foreach}
```

## Automation

### Auto-Create Notes

```php
// Create notes automatically
[
    'trigger' => 'ticket_response',
    'action' => 'create_note',
    'note_content' => 'Ticket #{$ticket_id} response logged'
]
```

### Note Reminders

```php
// Reminder based on note
[
    'note_id' => 789,
    'reminder_date' => '2024-05-20',
    'reminder_text' => 'Follow up with customer',
    'assigned_to' => 'admin_id'
]
```

## API Functions

```php
// Add client note
$result = localAPI('AddClientNote', [
    'userid' => 123,
    'note' => 'Customer note content',
    'sticky' => false
]);

// Get client notes
$result = localAPI('GetClientNotes', [
    'clientid' => 123
]);

// Update note
$result = localAPI('UpdateClientNote', [
    'noteid' => 789,
    'note' => 'Updated content'
]);

// Delete note
$result = localAPI('DeleteClientNote', [
    'noteid' => 789
]);
```

## Hooks

```php
// Hook: ClientNoteCreated
add_hook('ClientNoteCreated', 1, function($vars) {
    // $vars['noteid']
    // $vars['userid']
    // $vars['note']
});

// Hook: ClientNoteEdited
add_hook('ClientNoteEdited', 1, function($vars) {
    // $vars['noteid']
    // $vars['modified_by']
});
```

## Best Practices

1. **Keep notes relevant**: Only add useful information
2. **Use sticky notes**: Highlight important information
3. **Organize systematically**: Use categories and tags
4. **Follow privacy**: Don't store sensitive data in notes
5. **Regular cleanup**: Archive old notes

## Related Documentation

- [Client Creation](./whmcs-client-creation.md)
- [Client Authentication](./whmcs-client-authentication.md)
- [Client Custom Fields](./whmcs-client-custom-fields.md)
- [Client Relationships](./whmcs-client-relationships.md)