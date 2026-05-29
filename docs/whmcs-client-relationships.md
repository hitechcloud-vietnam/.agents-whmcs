# WHMCS Client Relationships

## Overview

Client relationships in WHMCS allow linking related client accounts. This is useful for corporate accounts, subsidiaries, resellers, and family accounts where multiple clients need to be associated.

## Relationship Types

### Parent-Child Relationship

```php
// Parent company with subsidiaries
[
    'parent_userid' => 123,
    'child_userid' => 456,
    'relationship_type' => 'subsidiary',
    'linked_at' => '2024-05-15'
]
```

### Reseller Relationship

```php
// Reseller and end customers
[
    'reseller_userid' => 789,
    'customer_userid' => 101,
    'relationship_type' => 'reseller_customer'
]
```

### Family Relationship

```php
// Family members
[
    'primary_userid' => 100,
    'related_userid' => 101,
    'relationship_type' => 'family',
    'relationship_note' => 'Spouse account'
]
```

## Creating Relationships

### Link Clients

**Admin: Clients > Select Client > Relationships > Add Relationship**

```php
// Create relationship
[
    'userid' => 123,
    'related_userid' => 456,
    'type' => 'subsidiary',
    'linked_by' => 'admin_id',
    'linked_at' => '2024-05-15'
]
```

### Relationship Types List

| Type | Description |
|------|-------------|
| subsidiary | Subsidiary company |
| parent | Parent company |
| reseller | Reseller account |
| reseller_customer | Reseller's customer |
| family | Family member |
| partner | Business partner |
| billing_contact | Billing contact |
| technical_contact | Technical contact |

## Relationship Types

### Billing Contact

```php
// Link billing contact
[
    'userid' => 123,
    'related_userid' => 456,
    'type' => 'billing_contact',
    'is_primary' => true
]
```

### Technical Contact

```php
// Link technical contact
[
    'userid' => 123,
    'related_userid' => 457,
    'type' => 'technical_contact',
    'can_view_services' => true,
    'can_manage_services' => false
]
```

## Managing Relationships

### View Relationships

**Admin: Clients > Select Client > Relationships**

```php
// Client relationships
[
    'userid' => 123,
    'parent' => null,
    'children' => [
        ['userid' => 456, 'type' => 'subsidiary', 'name' => 'Sub Company'],
        ['userid' => 457, 'type' => 'subsidiary', 'name' => 'Branch Office']
    ],
    'contacts' => [
        ['userid' => 458, 'type' => 'billing', 'name' => 'John Doe'],
        ['userid' => 459, 'type' => 'technical', 'name' => 'Jane Smith']
    ]
]
```

### Edit Relationship

```php
// Update relationship
[
    'relationship_id' => 789,
    'type' => 'billing_contact',
    'is_primary' => false
]
```

### Remove Relationship

```php
// Remove link
[
    'relationship_id' => 789,
    'userid' => 123,
    'related_userid' => 456,
    'removed_by' => 'admin_id',
    'removed_at' => '2024-05-15'
]
```

## Relationship Permissions

### Service Sharing

```php
// Share services between related clients
[
    'can_view_services' => true,
    'can_manage_services' => false,
    'can_view_invoices' => true,
    'can_pay_invoices' => false
]
```

### Access Control

```php
// Configure access
[
    'can_impersonate' => false,
    'can_transfer_services' => false,
    'can_view_notes' => false,
    'can_add_notes' => false
]
```

## Hierarchy View

### Parent Account View

```php
// View hierarchy from parent
[
    'userid' => 123,
    'name' => 'Parent Corp',
    'children' => [
        ['userid' => 456, 'name' => 'Sub 1', 'children' => [...]],
        ['userid' => 457, 'name' => 'Sub 2']
    ],
    'total_accounts' => 5
]
```

## Consolidated Billing

### Billing Hierarchy

```php
// Enable consolidated billing
[
    'billing_parent' => 123,
    'bill_children' => true,
    'invoice_separation' => 'single'     // single, separate
]
```

## Relationship Reports

### Relationship Report

**Reports > Clients > Client Relationships**

```php
// Relationship summary
[
    'total_relationships' => 150,
    'by_type' => [
        'subsidiary' => 50,
        'billing_contact' => 60,
        'technical_contact' => 40
    ],
    'orphan_accounts' => 10              // No relationships
]
```

## API Functions

```php
// Create relationship
$result = localAPI('CreateClientRelationship', [
    'userid' => 123,
    'related_userid' => 456,
    'type' => 'subsidiary'
]);

// Get relationships
$result = localAPI('GetClientRelationships', [
    'clientid' => 123
]);

// Remove relationship
$result = localAPI('RemoveClientRelationship', [
    'relationshipid' => 789
]);
```

## Best Practices

1. **Clear hierarchy**: Establish clear parent-child relationships
2. **Proper permissions**: Configure appropriate access controls
3. **Document relationships**: Note why accounts are linked
4. **Regular review**: Audit relationships periodically
5. **Consolidated billing**: Consider billing hierarchy benefits

## Related Documentation

- [Client Contacts](./whmcs-client-contacts.md)
- [Client Groups](./whmcs-client-groups.md)
- [Billing Hierarchy](./whmcs-billing-hierarchy.md)
- [Client Permissions](./whmcs-client-permissions.md)