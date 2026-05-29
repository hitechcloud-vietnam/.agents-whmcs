# WHMCS CAA Record Setup Documentation

## Overview

Certification Authority Authorization (CAA) records specify which Certificate Authorities (CAs) are allowed to issue SSL/TLS certificates for a domain. This provides an additional layer of security for domain ownership.

## Understanding CAA Records

### Record Purpose

CAA records help:
- Prevent unauthorized CAs from issuing certificates
- Reduce risk of certificate mis-issuance
- Improve security posture for domains
- Provide control over certificate issuance

### Record Structure

```
domain.com.  IN CAA  0 issue "letsencrypt.org"
domain.com.  IN CAA  0 issuewild "letsencrypt.org"
domain.com.  IN CAA  0 iodef "mailto:security@example.com"
```

### Flags

| Flag | Value | Meaning |
|------|-------|---------|
| 0 | 0 | Non-critical - CA may ignore if unsupported |
| 128 | 128 | Critical - CA MUST understand and follow |

### Tags

| Tag | Purpose | Value |
|-----|---------|-------|
| `issue` | Authorize CA to issue certs | CA domain |
| `issuewild` | Authorize CA to issue wildcards | CA domain |
| `iodef` | URL for violation reports | URL or mailto |

## Configuration

### Enable CAA Support

Navigate to: **Configuration > SSL Certificates > CAA Settings**

```php
// CAA Configuration
$caaConfig = [
    'enabled' => true,
    'auto_add_default' => true,
    'default_ca' => 'letsencrypt.org',
    'allow_wildcard' => false,
    'notify_email' => 'security@example.com',
    'validation' => 'strict'
];
```

### CAA Presets

```php
// Predefined CAA configurations
$caaPresets = [
    'letsencrypt_only' => [
        'issue' => 'letsencrypt.org',
        'issuewild' => 'letsencrypt.org',
        'iodef' => 'mailto:security@example.com'
    ],
    'digicert_only' => [
        'issue' => 'digicert.com',
        'issuewild' => 'digicert.com',
        'iodef' => 'mailto:security@example.com'
    ],
    'multiple_ca' => [
        'issue' => ['letsencrypt.org', 'digicert.com'],
        'issuewild' => 'letsencrypt.org',
        'iodef' => 'mailto:security@example.com'
    ],
    'block_all' => [
        'issue' => ';',
        'iodef' => 'mailto:security@example.com'
    ]
];
```

## Common CAA Configurations

### Let's Encrypt Only

```
@  IN CAA  0 issue "letsencrypt.org"
@  IN CAA  0 issuewild "letsencrypt.org"
@  IN CAA  0 iodef "mailto:security@example.com"
```

### Multiple CAs

```
@  IN CAA  0 issue "letsencrypt.org"
@  IN CAA  0 issue "digicert.com"
@  IN CAA  0 issuewild "letsencrypt.org"
@  IN CAA  0 iodef "mailto:security@example.com"
```

### Block All Certificate Issuance

```
@  IN CAA  0 issue ";"
@  IN CAA  0 iodef "mailto:security@example.com"
```

### AWS Certificate Manager

```
@  IN CAA  0 issue "amazon.com"
@  IN CAA  0 iodef "mailto:security@example.com"
```

## API Reference

### Add CAA Record

```http
POST /dns/zones/{zone}/records/caa
```

**Request Body:**

```json
{
  "name": "@",
  "flags": 0,
  "tag": "issue",
  "value": "letsencrypt.org",
  "ttl": 3600
}
```

### Add Multiple CAA Records

```http
POST /dns/zones/{zone}/records
```

**Request Body:**

```json
{
  "records": [
    {
      "name": "@",
      "type": "CAA",
      "flags": 0,
      "tag": "issue",
      "value": "letsencrypt.org",
      "ttl": 3600
    },
    {
      "name": "@",
      "type": "CAA",
      "flags": 0,
      "tag": "issuewild",
      "value": "letsencrypt.org",
      "ttl": 3600
    },
    {
      "name": "@",
      "type": "CAA",
      "flags": 0,
      "tag": "iodef",
      "value": "mailto:security@example.com",
      "ttl": 3600
    }
  ]
}
```

### List CAA Records

```http
GET /dns/zones/{zone}/records?type=CAA
```

**Response:**

```json
{
  "zone": "example.com",
  "records": [
    {
      "id": "REC-001",
      "name": "@",
      "type": "CAA",
      "flags": 0,
      "tag": "issue",
      "value": "letsencrypt.org",
      "ttl": 3600
    },
    {
      "id": "REC-002",
      "name": "@",
      "type": "CAA",
      "flags": 0,
      "tag": "issuewild",
      "value": "letsencrypt.org",
      "ttl": 3600
    },
    {
      "id": "REC-003",
      "name": "@",
      "type": "CAA",
      "flags": 0,
      "tag": "iodef",
      "value": "mailto:security@example.com",
      "ttl": 3600
    }
  ]
}
```

### Update CAA Record

```http
PUT /dns/zones/{zone}/records/{record_id}
```

### Delete CAA Record

```http
DELETE /dns/zones/{zone}/records/{record_id}
```

## Certificate Authority Codes

### Common CAs

| CA | Issue Value | Notes |
|----|-------------|-------|
| Let's Encrypt | `letsencrypt.org` | Free, automated |
| DigiCert | `digicert.com` | Premium EV/OV |
| Sectigo | `sectigo.com` | Formerly Comodo |
| GlobalSign | `globalsign.com` | Enterprise |
| GoDaddy | `godaddy.com` | Standard certificates |
| Amazon | `amazon.com` | AWS ACM |
| Cloudflare | `cloudflare.com` | Cloudflare SSL |

### CA Domain Patterns

```php
// Known CA domain patterns
$caDomains = [
    'letsencrypt.org' => [
        'name' => 'Let\'s Encrypt',
        'website' => 'https://letsencrypt.org',
        'wildcard_support' => true
    ],
    'sectigo.com' => [
        'name' => 'Sectigo',
        'website' => 'https://sectigo.com',
        'wildcard_support' => true
    ],
    'digicert.com' => [
        'name' => 'DigiCert',
        'website' => 'https://digicert.com',
        'wildcard_support' => true
    ],
    'globalsign.com' => [
        'name' => 'GlobalSign',
        'website' => 'https://globalsign.com',
        'wildcard_support' => true
    ]
];
```

## IODEF Reporting

### Email IODEF

```
@  IN CAA  0 iodef "mailto:security@example.com"
```

### HTTPS IODEF

```
@  IN CAA  0 iodef "https://security.example.com/caa-report"
```

### IODEF Payload Format

When a CA receives a request for a domain with CAA:

```json
{
  "version": "1.0",
  "type": "incident",
  "lang": "en",
  "contact": "CA contact email",
  "created": "2024-01-15T10:30:00Z",
  "range": "dns",
  "resource-record": {
    "domain": "example.com",
    "caa": "example.com. 0 issue \"unauthorized-ca.com\""
  },
  "event": {
    "event-type": "certificate-request",
    "action": "policy-violation"
  }
}
```

## Auto-Provisioning

### Enable Auto-CAA

```php
// Auto-provision CAA when SSL ordered
$autoProvision = [
    'enabled' => true,
    'on_ssl_order' => true,
    'ca_to_use' => 'letsencrypt.org',
    'add_issue' => true,
    'add_issuewild' => true,
    'add_iodef' => true,
    'notification_email' => 'security@example.com'
];
```

### On Certificate Order

```php
// When SSL certificate is ordered
$order = WHMCS\SSL\Order::create([...]);

// Automatically add CAA record
if ($autoProvision['enabled']) {
    WHMCS\Dns\Caa::addForDomain($order->domain, [
        'issue' => $autoProvision['ca_to_use'],
        'issuewild' => $autoProvision['ca_to_use'],
        'iodef' => $autoProvision['notification_email']
    ]);
}
```

## Validation

### CAA Record Validation

```php
// Validate CAA record
function validateCaaRecord($record) {
    $errors = [];

    // Validate flags
    if (!in_array($record['flags'], [0, 128])) {
        $errors[] = "Flags must be 0 or 128";
    }

    // Validate tag
    if (!in_array($record['tag'], ['issue', 'issuewild', 'iodef'])) {
        $errors[] = "Tag must be issue, issuewild, or iodef";
    }

    // Validate value based on tag
    if ($record['tag'] === 'issue' || $record['tag'] === 'issuewild') {
        if (empty($record['value']) || $record['value'] === ';') {
            // Valid - blocks issuance
        } elseif (!isValidDomain($record['value'])) {
            $errors[] = "Invalid CA domain in value";
        }
    }

    if ($record['tag'] === 'iodef') {
        if (!isValidUrl($record['value']) && !isValidMailto($record['value'])) {
            $errors[] = "Invalid IODEF URL or mailto address";
        }
    }

    return ['valid' => empty($errors), 'errors' => $errors];
}
```

## Troubleshooting

### Common CAA Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| CAA validation fails | Missing CAA record | Add CAA record |
| Wrong CA authorized | Wrong issue value | Update CAA record |
| Certificates failing | CA not in CAA | Add CA to CAA record |
| IODEF not working | Invalid URL format | Check IODEF URL |

### CAA Checking Tools

| Tool | URL |
|------|-----|
| SSLMate CAA Checker | https://sslmate.com/caa |
| DigiCert CAA Checker | https://www.digicert.com/caa |
| Sectigo CAA Lookup | https://sectigo.com/resource/caa-lookup |

### Debug Commands

```bash
# Check CAA records
dig @ns1.registrar.com CAA example.com +short

# CAA propagation check
whmcscli dns caa-check --zone=example.com

# Validate CAA configuration
whmcscli dns caa-validate --zone=example.com
```

## Security Best Practices

### Recommendations

1. **Always include issuewild** when using issue
2. **Use iodef** to receive violation reports
3. **Block unused CAs** by not including them
4. **Use critical flag** (128) for strict enforcement
5. **Review CAA regularly** for accuracy

### CAA Policy Template

```json
{
  "domain": "example.com",
  "caa_policy": {
    "authorized_cas": ["letsencrypt.org"],
    "wildcard_allowed": true,
    "reporting_enabled": true,
    "report_to": "mailto:security@example.com"
  },
  "review_frequency": "quarterly"
}
```

## See Also

- [Records Management](./whmcs-records-management.md)
- [Zone Editor](./whmcs-zone-editor.md)
- [DNSSEC API](./whmcs-dnssec-api.md)
