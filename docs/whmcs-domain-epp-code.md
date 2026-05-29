# WHMCS Domain EPP Code

## Overview

EPP (Extensible Provisioning Protocol) codes in WHMCS are authorization codes required for domain transfers. They verify the registrant has approved the transfer.

## EPP Code Overview

### What is EPP Code

```php
// EPP code details
[
    'name' => 'Auth Code',
    'also_known_as' => ['Transfer Code', 'Auth Info Code', 'Secret Key'],
    'purpose' => 'Authorize domain transfer',
    'format' => 'Alphanumeric string (6-16 characters)'
]
```

## Code Retrieval

### Request EPP Code

```php
// Request from current registrar
[
    'domain_id' => 1,
    'action' => 'request_epp',
    'send_to_email' => 'registrant@example.com',
    'registrar_api_call' => 'GetAuthCode'
]
```

### Display EPP Code

```php
// Show code to customer
[
    'domain_id' => 1,
    'epp_code' => 'ABC123XYZ789',
    'code_expires' => '2024-05-22',
    'single_use' => true
]
```

## Transfer with EPP

### Initiate Transfer

```php
// Transfer with code
[
    'domain' => 'example.com',
    'transfer_secret' => 'ABC123XYZ789',
    'validate_code' => true
]
```

### Code Validation

```php
// Validate before transfer
[
    'domain' => 'example.com',
    'code' => 'ABC123XYZ789',
    'valid' => true,
    'code_expires' => null
]
```

## Code Management

### Code Generation

```php
// Generate new code
[
    'domain_id' => 1,
    'action' => 'regenerate',
    'reason' => 'Customer request',
    'old_code_expires' => 'immediately'
]
```

### Code Security

```php
// Secure code handling
[
    'encrypt_storage' => true,
    'single_display' => true,          // Show once
    'log_access' => true,
    'notify_on_access' => false
]
```

## Registrar-Specific Codes

### Different Registrars

```php
// Registrar EPP formats
[
    'enom' => '8-16 alphanumeric',
    'godaddy' => '6-16 alphanumeric',
    'network_solutions' => '6-16 alphanumeric'
]
```

## API Functions

```php
// Get EPP code
$result = localAPI('GetDomainEPPCode', [
    'domainid' => 1
]);

// Request new code
$result = localAPI('RequestDomainEPPCode', [
    'domainid' => 1
]);
```

## Best Practices

1. **Secure storage**: Encrypt EPP codes
2. **Single display**: Show code only when needed
3. **Log access**: Track who views codes
4. **Validate codes**: Verify before transfer
5. **Regenerate securely**: Use secure generation

## Related Documentation

- [Domain Transfer](./whmcs-domain-transfer.md)
- [Domain Registration](./whmcs-domain-registration.md)
- [Transfer Lock](./whmcs-domain-transfer-lock.md)
- [Domain Command](./whmcs-domain-command.md)