# WHMCS DNS Templates Documentation

## Overview

DNS templates allow you to pre-define DNS record sets that can be quickly applied to domains, simplifying zone configuration and ensuring consistency.

## Template System

### Template Types

| Type | Description | Use Case |
|------|-------------|----------|
| Standard | Basic record set | Common web hosting |
| Email | Email-focused records | MX, SPF, DKIM, DMARC |
| Full Service | Complete configuration | All-in-one setup |
| Custom | User-defined templates | Application-specific |

## Configuration

### Enable DNS Templates

Navigate to: **Configuration > Products/Services > DNS Templates**

```php
// DNS Template Configuration
$templateConfig = [
    'enabled' => true,
    'allow_customer_create' => false,
    'default_template' => 'standard',
    'template_versioning' => true,
    'auto_apply_on_registration' => false
];
```

## Template Structure

### Template Definition

```json
{
  "name": "Standard Web Hosting",
  "version": "2.0",
  "description": "Standard setup for web hosting with email",
  "records": [
    {
      "name": "@",
      "type": "A",
      "value": "192.0.2.1",
      "ttl": 3600,
      "priority": null
    },
    {
      "name": "www",
      "type": "CNAME",
      "value": "@",
      "ttl": 3600
    },
    {
      "name": "@",
      "type": "MX",
      "value": "mail.example.com",
      "priority": 10,
      "ttl": 3600
    },
    {
      "name": "@",
      "type": "TXT",
      "value": "v=spf1 mx -all",
      "ttl": 3600
    }
  ],
  "variables": [
    {
      "name": "server_ip",
      "type": "ip",
      "required": true
    },
    {
      "name": "mail_server",
      "type": "domain",
      "required": false,
      "default": "mail"
    }
  ]
}
```

## Built-in Templates

### Standard Web Template

```json
{
  "name": "standard-web",
  "display_name": "Standard Web Hosting",
  "records": [
    {"name": "@", "type": "A", "value": "{server_ip}", "ttl": 3600},
    {"name": "www", "type": "CNAME", "value": "@", "ttl": 3600}
  ]
}
```

### Email Template

```json
{
  "name": "email-full",
  "display_name": "Full Email Service",
  "records": [
    {"name": "@", "type": "MX", "value": "mail.{domain}", "priority": 10, "ttl": 3600},
    {"name": "@", "type": "MX", "value": "mail2.{domain}", "priority": 20, "ttl": 3600},
    {"name": "mail", "type": "A", "value": "{mail_server_ip}", "ttl": 3600},
    {"name": "@", "type": "TXT", "value": "v=spf1 mx a ip4:{mail_server_ip} -all", "ttl": 3600},
    {
      "name": "google._domainkey",
      "type": "TXT",
      "value": "{dkim_public_key}",
      "ttl": 3600
    },
    {
      "name": "@",
      "type": "TXT",
      "value": "v=DMARC1; p=quarantine; rua=mailto:dmarc@{domain}",
      "ttl": 3600
    }
  ]
}
```

### WordPress Template

```json
{
  "name": "wordpress",
  "display_name": "WordPress Hosting",
  "records": [
    {"name": "@", "type": "A", "value": "{server_ip}", "ttl": 3600},
    {"name": "www", "type": "CNAME", "value": "@", "ttl": 3600},
    {"name": "blog", "type": "CNAME", "value": "@", "ttl": 3600},
    {"name": "@", "type": "MX", "value": "mail.{domain}", "priority": 10, "ttl": 3600},
    {"name": "@", "type": "TXT", "value": "v=spf1 mx -all", "ttl": 3600}
  ]
}
```

### E-commerce Template

```json
{
  "name": "ecommerce",
  "display_name": "E-commerce Platform",
  "records": [
    {"name": "@", "type": "A", "value": "{server_ip}", "ttl": 3600},
    {"name": "www", "type": "CNAME", "value": "@", "ttl": 3600},
    {"name": "shop", "type": "A", "value": "{server_ip}", "ttl": 3600},
    {"name": "api", "type": "A", "value": "{api_server_ip}", "ttl": 3600},
    {"name": "@", "type": "MX", "value": "mail.{domain}", "priority": 10, "ttl": 3600},
    {"name": "@", "type": "TXT", "value": "v=spf1 mx -all", "ttl": 3600},
    {"name": "_dmarc", "type": "TXT", "value": "v=DMARC1; p=quarantine; rua=mailto:dmarc@{domain}", "ttl": 3600}
  ]
}
```

## Creating Custom Templates

### Template Editor

**Configuration > DNS Templates > Create Template**

```
+------------------------------------------------------------------+
|  Create DNS Template                                              |
+------------------------------------------------------------------+
|                                                                  |
|  Template Name: [My Custom Template_________________]           |
|  Display Name: [My Custom Template________________]             |
|  Description: [Standard custom setup_______________]             |
|                                                                  |
|  Variables:                                                      |
|  +--------------------------------------------------------------+|
|  | Name          | Type    | Required | Default                  ||
|  |---------------|---------|----------|-------------------------||
|  | server_ip     | IP      | Yes      | -                       ||
|  | mail_server   | Domain  | No       | mail                    ||
|  | backup_ip     | IP      | No       | -                       ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  Records:                                                        |
|  +--------------------------------------------------------------+|
|  | Name | Type | Value                    | TTL    | Priority   ||
|  |------|------|--------------------------|--------|------------||
|  | @    | A    | {server_ip}              | 3600   | -          ||
|  | www  | CNAME| @                        | 3600   | -          ||
|  | + Add Record                                            |      ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  [Save Template] [Save & Apply] [Preview]                       |
+------------------------------------------------------------------+
```

### Template API

```http
POST /dns/templates
```

**Request Body:**

```json
{
  "name": "custom-web",
  "display_name": "Custom Web Template",
  "description": "Custom configuration for web hosting",
  "variables": [
    {"name": "server_ip", "type": "ip", "required": true},
    {"name": "server_ip_2", "type": "ip", "required": false}
  ],
  "records": [
    {
      "name": "@",
      "type": "A",
      "value": "{server_ip}",
      "ttl": 3600
    },
    {
      "name": "www",
      "type": "CNAME",
      "value": "@",
      "ttl": 3600
    },
    {
      "name": "@",
      "type": "MX",
      "value": "mail.{domain}",
      "priority": 10,
      "ttl": 3600
    }
  ]
}
```

## Template Variables

### Variable Types

| Type | Format | Validation |
|------|--------|------------|
| `ip` | IPv4 address | Valid IPv4 |
| `ipv6` | IPv6 address | Valid IPv6 |
| `domain` | Domain name | Valid domain |
| `string` | Any text | No validation |
| `number` | Numeric value | Numeric only |
| `select` | Predefined options | Must match options |

### System Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `{domain}` | Current domain name | example.com |
| `{domain_name}` | Same as domain | example.com |
| `{subdomain}` | Subdomain part | www |
| `{tld}` | Top-level domain | com |

### Custom Variables

```json
{
  "variables": [
    {
      "name": "primary_ip",
      "type": "ip",
      "label": "Primary Server IP",
      "required": true
    },
    {
      "name": "secondary_ip",
      "type": "ip",
      "label": "Secondary Server IP",
      "required": false
    },
    {
      "name": "mail_provider",
      "type": "select",
      "options": ["google", "microsoft", "custom"],
      "default": "google"
    }
  ]
}
```

## Applying Templates

### Apply to Single Domain

```http
POST /dns/zones/{zone}/apply-template
```

**Request Body:**

```json
{
  "template": "standard-web",
  "variables": {
    "server_ip": "192.0.2.1"
  },
  "options": {
    "replace_existing": false,
    "skip_conflicts": true
  }
}
```

### Apply to Multiple Domains

```http
POST /dns/templates/{template_id}/apply
```

**Request Body:**

```json
{
  "domains": ["example.com", "example.net", "example.org"],
  "variables": {
    "server_ip": "192.0.2.1"
  },
  "options": {
    "replace_existing": false
  }
}
```

## Template Management

### List Templates

```http
GET /dns/templates
```

**Response:**

```json
{
  "templates": [
    {
      "id": "TMPL-001",
      "name": "standard-web",
      "display_name": "Standard Web Hosting",
      "description": "Basic web hosting records",
      "record_count": 4,
      "usage_count": 156,
      "created_at": "2024-01-01T00:00:00Z"
    },
    {
      "id": "TMPL-002",
      "name": "email-full",
      "display_name": "Full Email Service",
      "description": "Complete email configuration",
      "record_count": 8,
      "usage_count": 89,
      "created_at": "2024-01-01T00:00:00Z"
    }
  ]
}
```

### Update Template

```http
PUT /dns/templates/{template_id}
```

### Delete Template

```http
DELETE /dns/templates/{template_id}
```

### Duplicate Template

```http
POST /dns/templates/{template_id}/duplicate
```

**Request Body:**

```json
{
  "new_name": "custom-web-v2",
  "new_display_name": "Custom Web v2"
}
```

## Template Versioning

### Version Control

```php
// Enable versioning
$versioningConfig = [
    'enabled' => true,
    'keep_versions' => 10,
    'auto_backup' => true
];
```

### Version History

```http
GET /dns/templates/{template_id}/versions
```

**Response:**

```json
{
  "template_id": "TMPL-001",
  "current_version": "2.1",
  "versions": [
    {
      "version": "2.1",
      "created_at": "2024-01-15T10:00:00Z",
      "created_by": "admin",
      "changes": "Added www CNAME record"
    },
    {
      "version": "2.0",
      "created_at": "2024-01-01T00:00:00Z",
      "created_by": "admin",
      "changes": "Major update - new record structure"
    },
    {
      "version": "1.0",
      "created_at": "2023-06-01T00:00:00Z",
      "created_by": "admin",
      "changes": "Initial version"
    }
  ]
}
```

### Rollback Template

```http
POST /dns/templates/{template_id}/rollback
```

**Request Body:**

```json
{
  "version": "1.0"
}
```

## Template Import/Export

### Export Template

```http
GET /dns/templates/{template_id}/export
```

**Response:**

```json
{
  "format": "json",
  "template": {
    "name": "standard-web",
    "display_name": "Standard Web Hosting",
    "records": [...]
  },
  "exported_at": "2024-01-15T10:00:00Z"
}
```

### Import Template

```http
POST /dns/templates/import
```

**Request Body:**

```json
{
  "format": "json",
  "template": {
    "name": "imported-web",
    "display_name": "Imported Web Template",
    "records": [...]
  },
  "overwrite": false
}
```

## Customer Templates

### Customer Template Access

```php
// Enable customer templates
$customerAccess = [
    'allow_customer_create' => true,
    'customer_max_templates' => 5,
    'require_approval' => false,
    'allowed_record_types' => ['A', 'AAAA', 'CNAME', 'MX', 'TXT', 'SRV']
];
```

### Customer Template Editor

**Client Area > My Domains > DNS Templates**

Customers can create their own templates for quick application:

```
+------------------------------------------------------------------+
|  My DNS Templates                                                |
+------------------------------------------------------------------+
|                                                                  |
|  [Create Template]                                              |
|                                                                  |
|  My Templates:                                                   |
|  +--------------------------------------------------------------+|
|  | Name              | Records | Used In | Actions              ||
|  |-------------------|---------|---------|---------------------||
|  | Personal Blog     | 4       | 2       | [Edit][Apply][Delete]||
|  | Custom API        | 3       | 1       | [Edit][Apply][Delete]||
|  +--------------------------------------------------------------+|
+------------------------------------------------------------------+
```

## Template Presets

### Product-Based Presets

Link templates to hosting products:

```php
$productPresets = [
    'starter-hosting' => 'standard-web',
    'business-email' => 'email-full',
    'wordpress-hosting' => 'wordpress',
    'reseller' => 'reseller-records'
];
```

### Automatic Application

```php
// Auto-apply template on domain registration
$autoApply = [
    'enabled' => true,
    'template' => 'standard-web',
    'variables' => [
        'server_ip' => 'default_server_ip'
    ],
    'products' => ['starter-hosting', 'basic-hosting']
];
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Variable not resolved | Missing variable | Add variable or default value |
| Invalid IP format | Wrong IP in variable | Use valid IP address |
| Template not found | Wrong template name | Check template name |
| Record conflict | Record already exists | Use replace_existing option |

### Debug Template Resolution

```bash
# Preview template resolution
whmcscli dns template-preview --template=standard-web --domain=example.com \
    --variables='{"server_ip":"192.0.2.1"}'
```

## See Also

- [Zone Editor](./whmcs-zone-editor.md)
- [Records Management](./whmcs-records-management.md)
- [Bulk DNS Operations](./whmcs-bulk-dns.md)
