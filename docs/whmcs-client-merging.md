# WHMCS Client Merging

## Overview

Client merging in WHMCS allows administrators to combine duplicate client accounts into a single account. This is useful when a customer has registered multiple times or when consolidating accounts is needed.

## Merge Configuration

### Enable Merging

**Configuration > Security > Client Merging**

```php
[
    'allow_merging' => true,
    'require_approval' => true,
    'approval_threshold' => 100.00,      // Require approval above this balance
    'merge_permissions' => ['admin', 'manager'],
    'log_all_merges' => true
]
```

## Finding Duplicates

### Duplicate Detection

**Admin: Clients > Find Duplicates**

```php
// Duplicate detection criteria
[
    'email_match' => true,              // Same email address
    'name_match' => true,                // Similar names
    'phone_match' => true,               // Same phone number
    'address_match' => true,             // Same address
    'min_match_score' => 80              // Percentage match required
]
```

### Duplicate Report

```php
// Duplicates found
[
    ['client_1' => 123, 'client_2' => 456, 'match_score' => 95, 'reason' => 'email'],
    ['client_1' => 789, 'client_2' => 790, 'match_score' => 85, 'reason' => 'name']
]
```

## Merge Process

### Step 1: Select Accounts

```php
// Primary account (survives)
[
    'userid' => 123,
    'name' => 'John Doe',
    'email' => 'john@example.com'
]

// Secondary account (to be merged)
[
    'userid' => 456,
    'name' => 'John Doe',
    'email' => 'johndoe2@example.com'
]
```

### Step 2: Review Data

```php
// Review data from both accounts
[
    'primary' => [
        'services' => 3,
        'invoices' => 20,
        'tickets' => 10,
        'total_spent' => 2000.00
    ],
    'secondary' => [
        'services' => 1,
        'invoices' => 5,
        'tickets' => 2,
        'total_spent' => 500.00
    ]
]
```

### Step 3: Choose Merge Options

```php
// What to merge
[
    'merge_services' => true,
    'merge_invoices' => true,
    'merge_tickets' => true,
    'merge_contacts' => true,
    'merge_notes' => true,
    'merge_custom_fields' => true,
    'merge_domain_contacts' => true,
    'update_primary_email' => false,
    'preserve_secondary_email' => 'contact_only'
]
```

## Data Merge Rules

### Service Merge

```php
// Merge services into primary
[
    'action' => 'merge_services',
    'from_userid' => 456,
    'to_userid' => 123,
    'services_affected' => 1
]
```

### Invoice Merge

```php
// Merge invoice records
[
    'action' => 'merge_invoices',
    'from_userid' => 456,
    'to_userid' => 123,
    'invoices_affected' => 5,
    'retain_original_numbers' => true
]
```

### Contact Merge

```php
// Merge additional contacts
[
    'action' => 'merge_contacts',
    'from_userid' => 456,
    'to_userid' => 123,
    'contacts' => [
        ['name' => 'Jane Doe', 'email' => 'jane@example.com', 'keep' => true],
        ['name' => 'Bob Smith', 'email' => 'bob@example.com', 'keep' => true]
    ]
]
```

## Merge Handling

### Handling Conflicts

```php
// When same data exists in both
[
    'email_conflict' => 'use_primary',    // use_primary, use_secondary, create_contact
    'phone_conflict' => 'keep_both',      // keep_both, use_primary, use_secondary
    'address_conflict' => 'use_primary',
    'notes_conflict' => 'merge_both'      // merge_both, keep_separate
]
```

### Duplicate Prevention

```php
// Prevent creating duplicate records
[
    'merge_contacts' => true,
    'check_for_existing' => true,
    'match_by_email' => true,
    'match_by_name' => true,
    'action_on_match' => 'link'            // link, keep_separate, merge
]
```

## Post-Merge Actions

### Update References

```php
// Update external references
[
    'update_domain_contacts' => true,
    'update_ticket_contacts' => true,
    'update_invoice_addresses' => true
]
```

### Secondary Account Status

```php
// After merge, secondary account
[
    'status' => 'Merged',
    'merged_into' => 123,
    'merged_at' => '2024-05-15',
    'data_retained' => true,
    'login_disabled' => true,
    'email_updated' => 'merged-456@example.com'
]
```

## Merge Validation

### Validation Checks

```php
// Validate before merge
[
    'check_services' => true,
    'check_invoices' => true,
    'check_balances' => true,
    'check_open_tickets' => true,
    'check_domain_registrations' => true
]
```

### Blocking Conditions

```php
// Conditions that block merge
[
    'has_unpaid_invoices_secondary' => true,   // Block if secondary has unpaid
    'has_active_services_secondary' => true,   // Require termination first
    'has_pending_orders' => true,
    'balance_conflict' => true                 // Cannot have different balances
]
```

## Merge Approval

### Require Approval

```php
// Approval workflow
[
    'require_approval' => true,
    'auto_approve_small' => true,
    'small_threshold' => 0,                  // Always require approval
    'notify_on_merge' => true
]
```

### Approval Request

```php
// Pending merge request
[
    'request_id' => 789,
    'primary_userid' => 123,
    'secondary_userid' => 456,
    'requested_by' => 'admin_id',
    'requested_at' => '2024-05-15',
    'status' => 'pending',
    'summary' => 'Merge 1 service, 5 invoices'
]
```

## Undoing a Merge

### Reverse Merge

```php
// Can only reverse within time limit
[
    'reverse_window' => 7,                   // days
    'reverse_possible' => true,
    'reverse_deadline' => '2024-05-22'
]
```

### Reverse Process

```php
// Revert merge action
[
    'action' => 'reverse_merge',
    'merge_id' => 789,
    'restore_secondary' => true,
    'restore_services' => true,
    'restore_invoices' => true,
    'confirm' => true
]
```

## API Functions

```php
// Merge clients
$result = localAPI('MergeClients', [
    'primary_clientid' => 123,
    'secondary_clientid' => 456,
    'merge_services' => true,
    'merge_invoices' => true
]);

// Get merge candidates
$result = localAPI('FindDuplicateClients', [
    'criteria' => 'email'
]);

// Reverse merge
$result = localAPI('ReverseClientMerge', [
    'merge_id' => 789
]);
```

## Hooks

```php
// Hook: PreClientMerge
add_hook('PreClientMerge', 1, function($vars) {
    // $vars['primary_userid']
    // $vars['secondary_userid']
    // Validate and potentially block
});

// Hook: ClientMerged
add_hook('ClientMerged', 1, function($vars) {
    // $vars['primary_userid']
    // $vars['secondary_userid']
    // $vars['merged_at']
    // Update external systems, notify, etc.
});
```

## Best Practices

1. **Backup data**: Export both accounts before merge
2. **Review carefully**: Check all data to preserve
3. **Communicate**: Inform customer of merge
4. **Document**: Keep record of merge for audit
5. **Test first**: Preview merge effects before committing

## Related Documentation

- [Client Creation](./whmcs-client-creation.md)
- [Client Deletion](./whmcs-client-deletion.md)
- [Client Export](./whmcs-client-export.md)
- [Client Import](./whmcs-client-import.md)