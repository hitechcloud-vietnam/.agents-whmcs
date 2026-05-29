# WHMCS Domain Nameservers

## Overview

Domain nameservers in WHMCS configure the DNS servers that a domain uses to resolve domain names to IP addresses.

## Nameserver Configuration

### Set Nameservers

```php
// Configure nameservers
[
    'domain_id' => 1,
    'nameservers' => [
        'ns1.yourcompany.com',
        'ns2.yourcompany.com'
    ]
]
```

## Default Nameservers

### Configure Defaults

```php
// Default NS settings
[
    'default_ns1' => 'ns1.yourcompany.com',
    'default_ns2' => 'ns2.yourcompany.com',
    'default_ns3' => 'ns3.yourcompany.com',
    'default_ns4' => 'ns4.yourcompany.com'
]
```

## Custom Nameservers

### Set Custom NS

```php
// Custom nameserver for domain
[
    'domain_id' => 1,
    'nameservers' => [
        'ns1.example.com',
        'ns2.example.com'
    ],
    'custom_glue' => true
]
```

## Glue Records

### Glue Record Configuration

```php
// IP addresses for nameservers
[
    'ns1.yourcompany.com' => '192.168.1.1',
    'ns2.yourcompany.com' => '192.168.1.2'
]
```

## Update Nameservers

### Change NS

```php
// Update nameservers
[
    'domain_id' => 1,
    'nameservers' => [
        'ns1.newcompany.com',
        'ns2.newcompany.com'
    ],
    'propagate_days' => 2
]
```

## API Functions

```php
// Update nameservers
$result = localAPI('UpdateDomainNameservers', [
    'domainid' => 1,
    'ns1' => 'ns1.example.com',
    'ns2' => 'ns2.example.com'
]);
```

## Best Practices

1. **Use reliable NS**: Ensure nameservers are stable
2. **Minimum 2**: Configure at least 2 nameservers
3. **Geographic distribution**: Spread across locations
4. **Monitor uptime**: Track NS availability

## Related Documentation

- [Domain Registration](./whmcs-domain-registration.md)
- [Domain DNS](./whmcs-domain-dnssec.md)
- [Domain Forwarding](./whmcs-domain-forwarding.md)