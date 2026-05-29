# WHMCS Nameserver IP Management Documentation

## Overview

Nameserver IP management in WHMCS enables configuration of nameserver addresses, glue records, and IP assignments for DNS infrastructure.

## Nameserver Configuration

### Basic Configuration

Navigate to: **Configuration > System Settings > DNS Settings**

```php
// Nameserver Configuration
$nameserverConfig = [
    // Default Nameservers
    'default_ns1' => 'ns1.yourdomain.com',
    'default_ns2' => 'ns2.yourdomain.com',
    'default_ns3' => 'ns3.yourdomain.com',
    'default_ns4' => 'ns4.yourdomain.com',

    // IP Addresses
    'ns1_ip' => '192.0.2.1',
    'ns2_ip' => '192.0.2.2',
    'ns3_ip' => '192.0.2.3',
    'ns4_ip' => '192.0.2.4',

    // IPv6
    'ns1_ipv6' => '2001:db8::1',
    'ns2_ipv6' => '2001:db8::2',

    // Options
    'auto_assign' => true,
    'allow_custom_ns' => true,
    'require_glue' => true
];
```

### Nameserver Assignment

#### Per-Domain Nameservers

```php
// Assign custom nameservers to domain
$domain->nameservers = [
    'ns1.custom-ns.com' => '192.0.2.10',
    'ns2.custom-ns.com' => '192.0.2.11'
];
$domain->save();
```

#### Template-Based Assignment

```php
// Nameserver Template
$nsTemplate = [
    'standard' => [
        'ns1.yourdomain.com' => '192.0.2.1',
        'ns2.yourdomain.com' => '192.0.2.2'
    ],
    'premium' => [
        'ns1.premium-dns.com' => '192.0.2.100',
        'ns2.premium-dns.com' => '192.0.2.101'
    ],
    'enterprise' => [
        'ns1.enterprise-dns.com' => '192.0.2.200',
        'ns2.enterprise-dns.com' => '192.0.2.201',
        'ns3.enterprise-dns.com' => '192.0.2.202'
    ]
];
```

## Glue Records

### What Are Glue Records?

Glue records are A/AAAA records added to the parent zone to resolve nameserver names that are within the delegated zone.

**Example:**
```
Parent zone (.com registry):
example.com. IN NS ns1.example.com.
ns1.example.com. IN A 192.0.2.1  <- Glue record

Your zone (example.com):
ns1.example.com. IN A 192.0.2.1  <- Your A record
```

### Managing Glue Records

```php
// Add/Update Glue Record
$api->setGlueRecord([
    'domain' => 'example.com',
    'nameserver' => 'ns1.example.com',
    'ipv4' => '192.0.2.1',
    'ipv6' => '2001:db8::1'
]);
```

### Glue Record Requirements

| Registry | Glue Required | Notes |
|----------|---------------|-------|
| .com | If NS in-zone | Best practice always |
| .net | If NS in-zone | Best practice always |
| .org | If NS in-zone | Best practice always |
| .io | Yes | Required |
| .co | Yes | Required |
| .uk | Conditional | If using nominet |
| .de | Conditional | DENIC specific |

## Nameserver Pools

### Creating a Pool

```php
// Define nameserver pool
$pool = new NameserverPool();
$pool->name = 'Standard Pool';
$pool->nameservers = [
    [
        'host' => 'ns1.primary.com',
        'ip' => '192.0.2.1',
        'weight' => 100,
        'region' => 'us-east'
    ],
    [
        'host' => 'ns1.backup.com',
        'ip' => '192.0.2.10',
        'weight' => 50,
        'region' => 'us-west'
    ]
];
$pool->save();
```

### Pool Distribution

| Pool Type | Description | Use Case |
|-----------|-------------|----------|
| Round Robin | Rotate through list | Simple load distribution |
| Weighted | Weight-based distribution | Traffic shaping |
| Geographic | Region-specific NS | GeoDNS routing |
| Latency | Lowest latency first | Performance optimization |

### Geographic Pool Example

```php
$geoPool = [
    'default' => [
        'ns1.global-dns.com' => '192.0.2.1',
        'ns2.global-dns.com' => '192.0.2.2'
    ],
    'regions' => [
        'us-east' => [
            'ns1.useast.com' => '192.0.2.10'
        ],
        'us-west' => [
            'ns1.uswest.com' => '192.0.2.20'
        ],
        'eu' => [
            'ns1.eu.dns.com' => '192.0.2.30'
        ],
        'apac' => [
            'ns1.apac.dns.com' => '192.0.2.40'
        ]
    ]
];
```

## IP Address Management

### IP Pool Configuration

```php
// IP Address Pool
$ipPool = [
    // IPv4 Addresses
    'ipv4' => [
        '192.0.2.0/24' => [
            'region' => 'us-east',
            'purpose' => 'primary-ns',
            'assigned' => ['192.0.2.1', '192.0.2.2'],
            'available' => ['192.0.2.3', '192.0.2.4']
        ],
        '192.0.3.0/24' => [
            'region' => 'us-west',
            'purpose' => 'secondary-ns',
            'assigned' => ['192.0.3.1', '192.0.3.2'],
            'available' => ['192.0.3.3']
        ]
    ],
    // IPv6 Addresses
    'ipv6' => [
        '2001:db8:1::/48' => [
            'region' => 'us-east',
            'assigned' => ['2001:db8:1::1', '2001:db8:1::2'],
            'available' => ['2001:db8:1::3']
        ]
    ]
];
```

### IP Assignment API

```php
// Assign IP to nameserver
$assignment = NameserverIP::assign([
    'nameserver' => 'ns3.new-dns.com',
    'type' => 'ipv4',
    'region' => 'us-east',
    'purpose' => 'additional-ns'
]);

// Response
[
    'nameserver' => 'ns3.new-dns.com',
    'ipv4' => '192.0.2.50',
    'assigned_at' => '2024-01-15T10:30:00Z'
]
```

### IP Release

```php
// Release IP address
NameserverIP::release('192.0.2.50', 'ns3.new-dns.com');
```

## DNSSEC with Nameservers

### DS Record Management

```php
// Generate DS Record
$dsRecord = DNSSEC::generateDSRecord(
    'example.com',
    $keySigningKey
);

// Output
[
    'key_tag' => 12345,
    'algorithm' => 13,      // ECDSAP256SHA256
    'digest_type' => 2,     // SHA-256
    'digest' => 'AABBCCDDEEFF...'
]
```

### DS Record Formats

**DNSKEY reference:**
```
example.com.  IN DS 12345 13 2 AABBCCDDEEFF001122...
```

### Key Signing Key (KSK) Rollover

```php
// Schedule KSK rollover
DNSSEC::scheduleKSKRollover([
    'domain' => 'example.com',
    'rollover_date' => '2024-04-01',
    'notify_registrar' => true
]);
```

## Custom Nameservers

### Allowing Customer Nameservers

```php
// Enable custom nameservers
$config = [
    'allow_custom_nameservers' => true,
    'min_custom_ns' => 2,
    'max_custom_ns' => 13,
    'require_glue_for_custom' => true,
    'validate_custom_ns' => true
];
```

### Custom NS Validation

```php
// Validate custom nameserver
$validation = WHMCS\Domains\Nameserver::validateCustom([
    'ns1.customers-ns.com' => '192.0.2.100',
    'ns2.customers-ns.com' => '192.0.2.101'
]);

// Validation checks:
// 1. Valid domain format
// 2. IP addresses are valid
// 3. Glue records are provided
// 4. Nameservers are functional
```

## Nameserver Templates

### Template Types

| Template | Nameservers | Use Case |
|----------|-------------|----------|
| Default | ns1/ns2.registrar.com | Standard registration |
| Premium | ns1/ns2.premium-dns.com | Premium DNS service |
| Enterprise | 4+ custom nameservers | Large organizations |
| Custom | Customer-provided | Customer choice |

### Creating Templates

```php
// Create nameserver template
$template = new NameserverTemplate();
$template->name = 'Premium DNS';
$template->is_default = false;
$template->nameservers = [
    ['host' => 'ns1.premium.whmcs.com', 'ip' => '192.0.2.1'],
    ['host' => 'ns2.premium.whmcs.com', 'ip' => '192.0.2.2'],
    ['host' => 'ns3.premium.whmcs.com', 'ip' => '192.0.2.3'],
    ['host' => 'ns4.premium.whmcs.com', 'ip' => '192.0.2.4']
];
$template->save();
```

## Failover Configuration

### Nameserver Failover

```php
// Configure nameserver failover
$failover = [
    'enabled' => true,
    'primary_ns' => 'ns1.primary.com',
    'secondary_ns' => 'ns2.secondary.com',
    'health_check' => [
        'enabled' => true,
        'interval' => 60,           // seconds
        'timeout' => 10,
        'retries' => 3,
        'check_type' => 'dns'      // dns, tcp, http
    ],
    'failover_behavior' => 'update_glue'
];
```

### Failover Flow

```
1. Primary NS health check fails
2. Wait for retry threshold
3. Mark primary as unhealthy
4. Update glue records to secondary
5. Send notification
6. Begin recovery checks on primary
7. When primary healthy, switch back (optional)
```

## Reporting

### Nameserver Status Report

```http
GET /admin/reports/nameservers
```

**Report Fields:**
- Total domains per nameserver
- IP address assignments
- Glue record status
- DNSSEC signing status
- Health check status
- Recent changes

### DNS Health Metrics

| Metric | Description | Threshold |
|--------|-------------|-----------|
| Query Success Rate | % successful queries | > 99.9% |
| Response Time | Average response time | < 100ms |
| Zone AXFR | Transfer success | 100% |
| DNSSEC Valid | Signature validation | > 99.9% |

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| NS not resolving | Missing glue | Add glue records |
| Propagation slow | High TTL | Lower TTL before changes |
| Custom NS rejected | Missing glue | Add glue for in-zone NS |
| IP conflict | Duplicate assignment | Release and reassign |
| DNSSEC broken | Expired KSK | Perform key rollover |

### Debug Commands

```bash
# Check nameserver configuration
whmcscli dns nameserver-list

# Verify glue records
whmcscli dns glue-check --domain=example.com

# Test nameserver resolution
whmcscli dns ns-lookup --domain=example.com --server=8.8.8.8

# DNS zone transfer test
whmcscli dns axfr --zone=example.com
```

### Nameserver Propagation

```
Nameserver changes typically propagate within:
- 24-48 hours for most TLDs
- Up to 72 hours for some registries
- Immediate for your local DNS cache (if low TTL)
```

## Best Practices

1. **Always provide glue records** for in-zone nameservers
2. **Use 2-4 nameservers** for redundancy
3. **Geographic distribution** for global services
4. **Low TTL before changes** for faster propagation
5. **Monitor nameserver health** continuously
6. **Keep IP addresses updated** when changes occur
7. **Document all nameserver assignments**
8. **Use IPv6** for modern infrastructure

## See Also

- [Zone Editor](./whmcs-zone-editor.md)
- [DNS Management API](./whmcs-dns-management-api.md)
- [DNSSEC API](./whmcs-dnssec-api.md)
