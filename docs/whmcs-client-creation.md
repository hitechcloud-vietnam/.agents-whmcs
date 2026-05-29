# WHMCS Client Creation

## Overview

Client creation in WHMCS is the process of registering new customers in the system. This can occur through self-registration, admin creation, or API integration. Each client record contains personal information, billing details, and account preferences.

## Client Creation Methods

### Self-Registration

**Client Area > Register / Sign Up**

```php
// Registration form fields
[
    'first_name' => 'required',
    'last_name' => 'required',
    'email' => 'required|unique',
    'phone_number' => 'optional',
    'company_name' => 'optional',
    'password' => 'required|min:8',
    'address_line_1' => 'required',
    'city' => 'required',
    'state' => 'required',
    'postcode' => 'required',
    'country' => 'required|dropdown'
]
```

### Admin Creation

**Admin: Clients > Add New Client**

```php
// Admin client creation
[
    'firstname' => 'John',
    'lastname' => 'Doe',
    'email' => 'john@example.com',
    'password' => 'auto_generate',  // or specified
    'client_group' => 'default',
    'language' => 'english',
    'custom_fields' => [...]
]
```

### API Creation

```php
// Create client via API
$params = [
    'firstname' => 'John',
    'lastname' => 'Doe',
    'email' => 'john@example.com',
    'password' => 'secure_password',
    'address1' => '123 Main St',
    'city' => 'New York',
    'state' => 'NY',
    'postcode' => '10001',
    'country' => 'US'
];
$result = localAPI('AddClient', $params);
```

## Required Fields

### Minimum Required Information

```php
// Required for client creation
[
    'firstname' => 'string|required',
    ' 'lastname' => 'string|required',
    'email' => 'email|required|unique',
    'password' => 'string|required'
]
```

### Full Address Information

```php
// Complete address
[
    'address1' => '123 Main Street',
    'address2' => 'Suite 100',
    'city' => 'New York',
    'state' => 'NY',
    'postcode' => '10001',
    'country' => 'US'
]
```

## Client Data Fields

### Personal Information

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| firstname | string | Yes | First name |
| lastname | string | Yes | Last name |
| email | email | Yes | Primary email |
| phonenumber | phone | No | Phone number |
| companyname | string | No | Company name |

### Address Information

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| address1 | string | Yes | Address line 1 |
| address2 | string | No | Address line 2 |
| city | string | Yes | City |
| state | string | Yes | State/Province |
| postcode | string | Yes | Postal code |
| country | string | Yes | Country code |

## Client Group Assignment

### Default Group Assignment

```php
// Assign client to group
[
    'client_group' => 'default',       // default, premium, vip
    'group_id' => 1,
    'auto_assign' => true               // Based on rules
]
```

### Group-Based Auto Assignment

```php
// Auto-assign rules
[
    'rules' => [
        ['field' => 'country', 'value' => 'US', 'assign_group' => 'us_customers'],
        ['field' => 'country', 'value' => 'UK', 'assign_group' => 'uk_customers'],
        ['field' => 'amount', 'operator' => '>=', 'value' => 1000, 'assign_group' => 'enterprise']
    ]
]
```

## Client Creation Workflow

### Step 1: Form Submission

```php
// Validate registration data
[
    'email_unique' => true,
    'password_strength' => 'medium',
    'required_fields' => ['firstname', 'lastname', 'email', 'password']
]
```

### Step 2: Account Creation

```php
// Create client record
[
    'status' => 'Active',
    'email_verified' => false,
    'created_at' => '2024-05-15',
    'language' => 'english',
    'currency' => 'USD'
]
```

### Step 3: Welcome Email

```php
// Send welcome email
[
    'template' => 'welcome',
    'include_verification' => true,
    'include_login_details' => true
]
```

### Step 4: Email Verification (Optional)

```php
// Verify email address
[
    'require_verification' => true,
    'verification_expiry' => 24,  // hours
    'verification_link' => '/verify.php?token=xxx'
]
```

## Post-Creation Actions

### Automatic Actions

```php
// After client creation
[
    'create_pending_order' => false,
    'send_welcome_email' => true,
    'add_to_newsletter' => true,
    'apply_signup_discount' => false,
    'assign_default_group' => true
]
```

### Product/Service Assignment

```php
// Assign initial service
[
    'product_id' => 1,
    'billing_cycle' => 'monthly',
    'create_invoice' => false,
    'send_approval_email' => false
]
```

## Validation Rules

### Email Validation

```php
[
    'format' => 'valid_email',
    'unique' => true,
    'mx_check' => true,
    'disposable_check' => false
]
```

### Password Requirements

```php
[
    'min_length' => 8,
    'require_uppercase' => false,
    'require_lowercase' => true,
    'require_number' => true,
    'require_special' => false
]
```

## Duplicate Detection

### Email Duplicate Check

```php
// Prevent duplicate accounts
[
    'check_existing' => true,
    'on_duplicate' => 'warn',         // warn, block, merge
    'allow_resend_verification' => true
]
```

### Merge Options

```php
// If duplicate detected
[
    'merge_option' => 'ask',          // ask, auto, never
    'merge_fields' => ['address', 'phone'],
    'keep_latest_notes' => true
]
```

## Client Creation Hooks

```php
// Hook: ClientAreaHomepage
add_hook('ClientAreaHomepage', 1, function($vars) {
    // Called after client creation
});

// Hook: PreClientCreate
add_hook('PreClientCreate', 1, function($vars) {
    // Validate before creation
    return ['allow' => true];
});

// Hook: ClientCreated
add_hook('ClientCreated', 1, function($vars) {
    // $vars['userid']
    // $vars['email']
    // Perform post-creation actions
});
```

## API Functions

```php
// Add new client
$result = localAPI('AddClient', [
    'firstname' => 'John',
    'lastname' => 'Doe',
    'email' => 'john@example.com',
    'password' => 'secure123',
    'address1' => '123 Main St',
    'city' => 'New York',
    'state' => 'NY',
    'postcode' => '10001',
    'country' => 'US'
]);

// Get client details
$result = localAPI('GetClient', [
    'clientid' => 123
]);
```

## Best Practices

1. **Collect only necessary data**: Don't require excessive information
2. **Clear validation messages**: Help users fix errors
3. **Email verification**: Ensure valid contact information
4. **Strong passwords**: Require secure passwords
5. **Welcome sequence**: Guide new clients through setup

## Related Documentation

- [Client Groups](./whmcs-client-groups.md)
- [Client Authentication](./whmcs-client-authentication.md)
- [Client Custom Fields](./whmcs-client-custom-fields.md)
- [Client Import](./whmcs-client-import.md)