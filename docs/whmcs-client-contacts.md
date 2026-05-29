# WHMCS Client Contacts

## Overview

Client contacts in WHMCS allow additional people to be associated with a client account. Contacts can have their own login access, receive notifications, and manage specific services without having full client access.

## Contact Management

### Accessing Contacts

**Admin: Clients > Select Client > Contacts**

### Add Contact

```php
// New contact
[
    'userid' => 123,
    'firstname' => 'Jane',
    'lastname' => 'Smith',
    'email' => 'jane@company.com',
    'company' => null,
    'phonenumber' => '555-0100',
    'subaccount' => true,
    'password' => 'SecurePass123!'
]
```

## Contact Types

### Billing Contact

```php
// Primary billing contact
[
    'type' => 'billing',
    'is_primary' => true,
    'can_view_invoices' => true,
    'can_pay_invoices' => true,
    'receive_invoice_emails' => true
]
```

### Technical Contact

```php
// Technical contact
[
    'type' => 'technical',
    'can_view_services' => true,
    'can_manage_services' => true,
    'receive_ticket_emails' => true
]
```

### General Contact

```php
// General contact
[
    'type' => 'general',
    'can_view_services' => true,
    'receive_announcements' => true
]
```

## Contact Permissions

### Subaccount Access

```php
// Configure contact access
[
    'subaccount' => true,
    'permissions' => [
        'services' => true,
        'domains' => false,
        'invoices' => false,
        'tickets' => true,
        'billing' => false
    ]
]
```

### Notification Settings

```php
// Contact notification preferences
[
    'email_invoices' => true,
    'email_ticket_updates' => true,
    'email_service_notifications' => false,
    'email_marketing' => false
]
```

## Contact Data

### Contact Information

```php
// Contact profile
[
    'contact_id' => 789,
    'userid' => 123,
    'firstname' => 'Jane',
    'lastname' => 'Smith',
    'email' => 'jane@company.com',
    'company' => null,
    'phonenumber' => '555-0100',
    'address1' => '123 Main St',
    'city' => 'New York',
    'state' => 'NY',
    'postcode' => '10001',
    'country' => 'US'
]
```

## Contact Login

### Subaccount Creation

```php
// Create login for contact
[
    'email' => 'jane@company.com',
    'password' => 'SecurePass123!',
    'send_welcome_email' => true,
    'require_email_verification' => true
]
```

### Contact Login Access

```php
// Contact can log in with own credentials
[
    'can_login' => true,
    'login_email' => 'jane@company.com',
    'last_login' => '2024-05-15 10:30:00'
]
```

## Contact Management

### Edit Contact

```php
// Update contact
[
    'contact_id' => 789,
    'firstname' => 'Jane',
    'lastname' => 'Smith',
    'email' => 'jane.new@company.com',
    'phonenumber' => '555-0200'
]
```

### Delete Contact

```php
// Remove contact
[
    'contact_id' => 789,
    'userid' => 123,
    'delete_login' => true,
    'reason' => 'No longer with company'
]
```

## Contact-Based Services

### Services Contact Can Manage

```php
// Assigned service management
[
    'contact_id' => 789,
    'can_manage_services' => [1, 2, 3],
    'can_view_services' => [4, 5]
]
```

## Contact Permissions Matrix

### Permission Levels

| Permission | Description |
|------------|-------------|
| view_services | View service details |
| manage_services | Modify services |
| view_domains | View domains |
| manage_domains | Modify domains |
| view_invoices | View invoices |
| pay_invoices | Make payments |
| view_tickets | View support tickets |
| create_tickets | Open tickets |
| manage_account | Update profile |

## Primary Contact

### Set Primary Contact

```php
// Mark as primary contact
[
    'contact_id' => 789,
    'userid' => 123,
    'is_primary' => true,
    'make_billing_contact' => true
]
```

## API Functions

```php
// Add contact
$result = localAPI('AddContact', [
    'clientid' => 123,
    'firstname' => 'Jane',
    'lastname' => 'Smith',
    'email' => 'jane@company.com',
    'subaccount' => true
]);

// Update contact
$result = localAPI('UpdateContact', [
    'contactid' => 789,
    'firstname' => 'Jane'
]);

// Get contacts
$result = localAPI('GetContacts', [
    'clientid' => 123
]);

// Delete contact
$result = localAPI('DeleteContact', [
    'contactid' => 789
]);
```

## Best Practices

1. **Limit subaccounts**: Only create necessary contacts
2. **Proper permissions**: Grant minimum required access
3. **Clear roles**: Define contact roles clearly
4. **Regular review**: Audit contact access periodically
5. **Remove unused**: Delete contacts no longer needed

## Related Documentation

- [Client Relationships](./whmcs-client-relationships.md)
- [Client Permissions](./whmcs-client-permissions.md)
- [Client Authentication](./whmcs-client-authentication.md)
- [Client Portal](./whmcs-client-portal.md)