# WHMCS Service Upgrade Request

## Overview

Service upgrade requests in WHMCS allow clients to submit requests for upgrading their services. These requests can be automatically processed or require admin approval.

## Request Types

### Product Upgrade Request

```php
// Request product change
[
    'service_id' => 1,
    'request_type' => 'product_upgrade',
    'current_product' => 'basic',
    'requested_product' => 'premium',
    'billing_cycle' => 'monthly',
    'prorate_amount' => 5.00
]
```

### Resource Upgrade Request

```php
// Request more resources
[
    'service_id' => 1,
    'request_type' => 'resource_upgrade',
    'current_resources' => ['disk' => '10GB', 'bandwidth' => '100GB'],
    'requested_resources' => ['disk' => '50GB', 'bandwidth' => '500GB'],
    'price_increase' => 5.00
]
```

## Request Process

### Client Submits Request

**Client Area > Services > Request Upgrade**

```php
// Client upgrade request
[
    'service_id' => 1,
    'upgrade_to' => 2,
    'effective' => 'immediately',
    'notes' => 'Need more disk space'
]
```

### Admin Reviews Request

**Admin: Orders > Upgrade Requests**

```php
// Review request
[
    'request_id' => 789,
    'service_id' => 1,
    'current_product' => 'Basic',
    'requested_product' => 'Premium',
    'price_difference' => 10.00,
    'status' => 'pending'
]
```

## Request Status

| Status | Description |
|--------|-------------|
| Pending | Awaiting review |
| Approved | Approved, processing |
| Rejected | Request denied |
| Completed | Upgrade done |

## Approval Workflow

### Auto-Approve

```php
// Auto-approve small upgrades
[
    'auto_approve' => true,
    'max_auto_approve_amount' => 10.00,
    'require_approval_above' => 10.00
]
```

### Manual Approval

```php
// Require admin approval
[
    'require_approval' => true,
    'approvers' => ['admin', 'manager'],
    'approval_email' => true,
    'auto_expire_days' => 7
]
```

## Request Validation

### Validate Upgrade

```php
// Check upgrade is possible
[
    'check' => [
        'target_exists' => true,
        'compatible' => true,
        'resources_available' => true,
        'price_calculated' => true
    ]
]
```

### Block Upgrades

```php
// Prevent upgrade if
[
    'block_if' => [
        'service_suspended' => true,
        'has_pending_requests' => true,
        'in_trial' => true
    ]
]
```

## Request Notification

### Notify Admin

```smarty
Subject: New Upgrade Request - {$service_domain}

Service: {$service_domain}
Current: {$current_product}
Requested: {$requested_product}
Amount: ${$prorate_amount}

Review: {$admin_link}
```

### Notify Client

```smarty
Subject: Upgrade Request Received

Dear {$client_name},

We received your upgrade request.

From: {$current_product}
To: {$requested_product}
Amount: ${$prorate_amount}

We'll review and process shortly.

{$company_name}
```

## Complete Upgrade

### Process Approved Request

```php
// Execute upgrade
[
    'request_id' => 789,
    'action' => 'approve',
    'execute_now' => true,
    'send_confirmation' => true
]
```

## Reject Request

### Deny Upgrade

```php
// Reject request
[
    'request_id' => 789,
    'reason' => 'Resource limitations',
    'notify_client' => true,
    'suggest_alternative' => 'basic_plus'
]
```

## API Functions

```php
// Submit upgrade request
$result = localAPI('SubmitUpgradeRequest', [
    'serviceid' => 1,
    'newproductid' => 2,
    'type' => 'product'
]);

// Approve upgrade request
$result = localAPI('ApproveUpgradeRequest', [
    'requestid' => 789
]);

// Reject upgrade request
$result = localAPI('RejectUpgradeRequest', [
    'requestid' => 789,
    'reason' => 'Not available'
]);
```

## Hooks

```php
// Hook: UpgradeRequestSubmitted
add_hook('UpgradeRequestSubmitted', 1, function($vars) {
    // $vars['requestid']
    // $vars['serviceid']
    // Notify admin, log, etc.
});

// Hook: UpgradeRequestApproved
add_hook('UpgradeRequestApproved', 1, function($vars) {
    // $vars['requestid']
    // $vars['serviceid']
    // Process upgrade, notify, etc.
});
```

## Best Practices

1. **Clear pricing**: Show proration clearly
2. **Quick response**: Process requests promptly
3. **Communicate**: Keep client informed
4. **Document**: Track all requests
5. **Offer alternatives**: Suggest alternatives if blocked

## Related Documentation

- [Service Upgrade](./whmcs-service-upgrade.md)
- [Service Downgrade](./whmcs-service-downgrade.md)
- [Service Modification](./whmcs-service-modification.md)
- [Pro-Rated Billing](./whmcs-pro-rated-billing.md)