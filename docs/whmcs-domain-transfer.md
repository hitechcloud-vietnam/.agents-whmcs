# WHMCS Domain Transfer

## Overview

Domain transfer in WHMCS handles the process of moving domains from another registrar to your management. This requires authorization from the current registrant.

## Transfer Process

### Initiate Transfer

**Client Area > Transfer Domain**

```php
// Transfer request
[
    'domain' => 'example.com',
    'auth_code' => 'EPPCODE123',
    'transfer_years' => 1,
    'registrant_approval' => true
]
```

### Transfer Steps

1. Validate domain and auth code
2. Check transfer eligibility
3. Initiate with current registrar
4. Send approval email to registrant
5. Confirm with current registrar
6. Complete transfer
7. Update DNS

## Authorization Code

### EPP Code

```php
// Authorization code
[
    'domain' => 'example.com',
    'epp_code' => 'ABC123XYZ',
    'code_format' => 'alphanumeric'
]
```

### Retrieve EPP Code

```php
// Request code from registrar
[
    'domain_id' => 1,
    'action' => 'request_epp',
    'send_to_email' => 'registrant@example.com'
]
```

## Transfer Eligibility

### Check Requirements

```php
// Validate transfer
[
    'domain' => 'example.com',
    'checks' => [
        'registered_days' => 60,          // Must be 60+ days
        'not_expired' => true,
        'not_pending' => true,
        'auth_code_valid' => true
    ]
]
```

### Block Transfer

```php
// Cannot transfer if
[
    'recently_registered' => true,       // < 60 days
    'recently_transferred' => true,     // < 60 days
    'expired_domain' => true,
    'pending_transfer' => true,
    'registry_lock' => true
]
```

## Transfer Pricing

### Transfer Costs

```php
// Transfer pricing
[
    'tld' => 'com',
    'transfer_price' => 9.95,
    'renewal_included' => 1,             // 1 year renewal included
    'registry_fee' => 7.50
]
```

## Transfer Status

### Status Tracking

| Status | Description |
|--------|-------------|
| Pending | Transfer initiated |
| Pending Owner | Awaiting registrant approval |
| Pending Registry | Registrar processing |
| Completed | Transfer successful |
| Rejected | Transfer rejected |
| Cancelled | Transfer cancelled |

## Approve Transfer

### Registrant Approval

```php
// Registrant approves
[
    'domain' => 'example.com',
    'approve' => true,
    'approved_by' => 'registrant',
    'approved_at' => '2024-05-15'
]
```

### Registrar Approval

```php
// Current registrar releases
[
    'domain' => 'example.com',
    'action' => 'release',
    'new_registrar' => 'your_registrar'
]
```

## Transfer Completion

### Complete Transfer

```php
// Transfer completed
[
    'domain' => 'example.com',
    'status' => 'Completed',
    'expiry_date' => '2025-05-15',      // Extended by transfer
    'nameservers' => ['ns1', 'ns2'],
    'registrant' => [...],
    'auto_renew' => true
]
```

## Transfer Failure

### Handle Failed Transfer

```php
// Transfer rejected
[
    'domain' => 'example.com',
    'status' => 'Rejected',
    'reason' => 'Invalid auth code',
    'retry_allowed' => true
]
```

## Transfer Confirmation

### Client Notification

```smarty
Subject: Domain Transfer Complete - {$domain}

Dear {$client_name},

Your domain transfer has been completed.

Domain: {$domain}
New Expiry: {$expiry_date}
Nameservers: {$nameservers}

{$company_name}
```

## API Functions

```php
// Initiate transfer
$result = localAPI('TransferDomain', [
    'domain' => 'example.com',
    'transfersecret' => 'EPPCODE123'
]);

// Get transfer status
$result = localAPI('GetTransferStatus', [
    'domain' => 'example.com'
]);

// Cancel transfer
$result = localAPI('CancelTransfer', [
    'domain' => 'example.com'
]);
```

## Hooks

```php
// Hook: DomainTransferCompleted
add_hook('DomainTransferCompleted', 1, function($vars) {
    // $vars['domainid']
    // $vars['domain']
    // Update DNS, notify, etc.
});
```

## Best Practices

1. **Validate auth code**: Verify before initiating
2. **Check eligibility**: Ensure domain can transfer
3. **Communicate**: Keep client informed
4. **Monitor status**: Track transfer progress
5. **Handle failures**: Process rejections properly

## Related Documentation

- [Domain Registration](./whmcs-domain-registration.md)
- [Domain Renewal](./whmcs-domain-renewal.md)
- [Domain Pricing](./whmcs-domain-pricing.md)
- [EPP Code](./whmcs-domain-epp-code.md)