# WHMCS Domain DNSSEC

## Overview

DNSSEC (Domain Name System Security Extensions) in WHMCS adds cryptographic signatures to DNS data to protect against DNS spoofing and cache poisoning attacks.

## DNSSEC Configuration

### Enable DNSSEC

**Configuration > Domains > DNSSEC**

```php
// DNSSEC settings
[
    'enable_dnssec' => true,
    'auto_sign' => true,
    'algorithm' => 'RSASHA256',
    'key_size' => 2048
]
```

## Key Management

### Generate Keys

```php
// Generate DNSSEC keys
[
    'domain_id' => 1,
    'algorithm' => 'RSASHA256',
    'key_type' => 'KSK',           // Key Signing Key
    'key_size' => 2048,
    'key_tag' => '12345'
]
```

### Key Types

| Type | Purpose |
|------|---------|
| KSK | Key Signing Key - signs ZSK |
| ZSK | Zone Signing Key - signs DNS records |

## DS Records

### DS Record Configuration

```php
// DS record data
[
    'domain_id' => 1,
    'key_tag' => 12345,
    'algorithm' => 8,                // RSASHA256
    'digest_type' => 2,             // SHA-256
    'digest' => 'ABCD1234...'
]
```

## DNSSEC Status

### Check Status

```php
// DNSSEC status
[
    'domain_id' => 1,
    'dnssec_enabled' => true,
    'keys' => [
        ['type' => 'KSK', 'key_tag' => '12345', 'status' => 'active'],
        ['type' => 'ZSK', 'key_tag' => '67890', 'status' => 'active']
    ],
    'ds_records' => 'configured',
    'parent_ds' => 'valid'
]
```

## API Functions

```php
// Enable DNSSEC
$result = localAPI('EnableDomainDNSSEC', [
    'domainid' => 1
]);

// Get DNSSEC info
$result = localAPI('GetDomainDNSSEC', [
    'domainid' => 1
]);
```

## Best Practices

1. **Enable DNSSEC**: Protect domain from DNS attacks
2. **Key rotation**: Regularly rotate signing keys
3. **Monitor status**: Check DNSSEC validation
4. **DS records**: Ensure DS properly configured

## Related Documentation

- [Domain Registration](./whmcs-domain-registration.md)
- [Domain Sync](./whmcs-domain-sync.md)
- [Domain Nameservers](./whmcs-domain-nameservers.md)