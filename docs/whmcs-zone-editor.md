# WHMCS Zone Editor Documentation

## Overview

The WHMCS Zone Editor provides a web-based interface for managing DNS zones. It supports full zone manipulation, template application, and advanced DNS features.

## Access

### Admin Access

Navigate to: **Configuration > Domain Management > Zone Editor**

Required permissions:
- `Domain Management` (full access)
- `DNS Management` (read/write)

### Client Access

Navigate to: **Client Area > My Domains > DNS Management**

Access control:
- Domain must be owned by client
- DNS management must be enabled for product

## Interface Layout

### Main Zone Editor

```
+----------------------------------------------------------+
|  Zone Editor: example.com                    [Zone Info] |
+----------------------------------------------------------+
|  Zone Settings     |  Records List                       |
|  +--------------+  |  +--------------------------------+ |
|  | TTL: 3600    |  |  | Name    | Type | Value   | TTL | |
|  | Refresh: 7200|  |  |---------|------|---------|-----| |
|  | Retry: 3600  |  |  | @       | A    | 1.2.3.4  | 3600| |
|  | Expire:      |  |  | www     | A    | 1.2.3.4  | 3600| |
|  | 1209600     |  |  | mail    | A    | 1.2.3.5  | 3600| |
|  +--------------+  |  | @       | MX   | mail..  | 3600| |
|                    |  | @       | TXT  | v=spf1..| 3600| |
|  [Add Record]      |  |---------|------|---------|-----| |
|  [Add Template]    |                                      |
|  [Import Zone]     |  [+ Add Record]  [Export]            |
+----------------------------------------------------------+
```

## Zone Settings

### SOA Record Configuration

**Location:** Zone Settings > SOA Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| TTL | 3600 | Default record TTL |
| Refresh | 7200 | Secondary DNS refresh interval |
| Retry | 3600 | Retry interval on failure |
| Expire | 1209600 | Maximum time secondary holds data |
| Minimum TTL | 86400 | Minimum cache time |

### Zone Templates

Apply pre-configured record sets:

```
Available Templates:
- Standard Web Hosting
- Email Only
- Full Service
- Custom...
```

### Zone Actions

| Action | Description |
|--------|-------------|
| Add Record | Create new DNS record |
| Apply Template | Apply record template |
| Import Zone | Import from BIND format |
| Export Zone | Export to various formats |
| Reset Zone | Clear all records |
| DNSSEC Settings | Configure DNSSEC |

## Record Types

### A Record (IPv4 Address)

```
+---------------------------+
| Add A Record              |
+---------------------------+
| Name: [www____________]   |
| IPv4:  [192.0.2.1_______] |
| TTL:   [3600____________] |
|                     [Add] |
+---------------------------+
```

**Fields:**
- Name: Subdomain (use @ for root)
- IPv4 Address: Valid IPv4 address
- TTL: Time to live in seconds

### AAAA Record (IPv6 Address)

```
+---------------------------+
| Add AAAA Record           |
+---------------------------+
| Name: [www____________]   |
| IPv6: [2001:db8::1____]   |
| TTL:   [3600____________] |
|                     [Add] |
+---------------------------+
```

### CNAME Record (Canonical Name)

```
+-----------------------------+
| Add CNAME Record            |
+-----------------------------+
| Name: [blog_____________]   |
| Target: [myblog.site.com__] |
| TTL:   [3600____________]   |
|                       [Add] |
+-----------------------------+
```

**Important:** CNAME records cannot coexist with other record types at the same name.

### MX Record (Mail Exchange)

```
+----------------------------------+
| Add MX Record                   |
+----------------------------------+
| Name: [@_____________________]   |
| Mail Server: [mail.example.com]  |
| Priority: [10___________]       |
| TTL:      [3600____________]   |
|                          [Add]  |
+----------------------------------+
```

**Priority:** Lower number = higher priority (1 is highest)

### TXT Record

```
+----------------------------------+
| Add TXT Record                   |
+----------------------------------+
| Name: [@_____________________]   |
| Value: [v=spf1 mx -all________] |
| TTL:   [3600____________]       |
|                          [Add]  |
+----------------------------------+
```

### SPF Record Builder

```
+----------------------------------+
| SPF Record Builder               |
+----------------------------------+
| Version: v=spf1 (auto)           |
| Mechanisms:                      |
| [x] Include: _spf.google.com    |
| [x] MX                           |
| [x] A                            |
| Qualifier: [-] Fail               |
| Final Rule: [Fail (-all)]         |
|                          [Add]   |
+----------------------------------+
```

### DKIM Record

```
+----------------------------------+
| Add DKIM Record                  |
+----------------------------------+
| Selector: [google___________]   |
| Public Key:                      |
| [p=MIIBIjANBgkqhkiG9w0BAQEF...] |
| TTL:      [3600____________]   |
|                           [Add] |
+----------------------------------+
```

### SRV Record

```
+----------------------------------+
| Add SRV Record                   |
+----------------------------------+
| Service:  [_sip]                 |
| Protocol: [_tcp]                 |
| Name: [_sip._tcp_______________]|
| Target: [sip.example.com_____]  |
| Priority: [10___________]        |
| Weight:   [5___________]         |
| Port:     [5060___________]     |
| TTL:      [3600____________]    |
|                            [Add]|
+----------------------------------+
```

### CAA Record

```
+----------------------------------+
| Add CAA Record                   |
+----------------------------------+
| Name: [@_____________________]   |
| Flags: [0_____________]         |
| Tag:   [issue____________]      |
| Value: [letsencrypt.org_______] |
| TTL:   [3600____________]       |
|                          [Add]  |
+----------------------------------+
```

**Tag Options:**
- `issue` - Allow CA to issue certificates
- `issuewild` - Allow wildcard certificates
- `iodef` - URL for violation reports

## Bulk Operations

### Bulk Add Records

```
+----------------------------------+
| Bulk Add Records                 |
+----------------------------------+
| [Enter records (one per line)]   |
| [www,A,192.0.2.1,3600]          |
| [blog,A,192.0.2.2,3600]         |
| [mail,A,192.0.2.10,3600]        |
| [Add Records] [Cancel]           |
+----------------------------------+
```

**Format:** `Name,Type,Value,TTL` (TTL optional)

### Bulk Delete Records

```
+----------------------------------+
| Bulk Delete                      |
+----------------------------------+
| Select records to delete:        |
| [x] www A 192.0.2.1             |
| [x] mail A 192.0.2.10            |
| [ ] @ MX mail.example.com       |
|                                   |
| [Delete Selected] [Cancel]       |
+----------------------------------+
```

## Import/Export

### Import Zone (BIND Format)

```
+----------------------------------+
| Import Zone                      |
+----------------------------------+
| Format: [BIND Zone File___]     |
| Source:                           |
| [                                ]
| [$ORIGIN example.com.]           |
| [@ IN SOA ns1.provider.com. ...] |
| [@ IN NS ns1.provider.com.]     |
| [@ IN A 192.0.2.1]              ]
|                                   |
| Options:                          |
| [x] Replace existing records     |
| [ ] Skip validation              |
|                                   |
| [Import] [Cancel]                 |
+----------------------------------+
```

### Export Zone

```
+----------------------------------+
| Export Zone                      |
+----------------------------------+
| Zone: example.com                |
| Format: [BIND Zone File___]     |
| Include Comments: [ ]            |
| Compress TTL: [ ]                |
|                                   |
| [Export] [Copy to Clipboard]     |
+----------------------------------+
```

**Export Formats:**
- BIND Zone File
- JSON
- CSV
- YAML

## DNSSEC Management

### DNSSEC Settings

```
+----------------------------------+
| DNSSEC Configuration             |
+----------------------------------+
| DNSSEC Status: [Enabled]         |
|                                   |
| DS Records (for parent zone):    |
| +-----------------------------+  |
| | Key Tag   | Alg | Digest    |  |
| |-----------|-----|------------|  |
| | 12345     | 13  | AABBCC... |  |
| +-----------------------------+  |
|                                   |
| [Regenerate Keys]                 |
| [Add DS Record]                   |
+----------------------------------+
```

### Add DS Record

```
+----------------------------------+
| Add DS Record                    |
+----------------------------------+
| Key: [Select Key____________]    |
| Algorithm: [13 - ECDSAP256___]  |
| Digest Type: [2 - SHA-256____]  |
| Digest: [AABBCCDDEEFF..._____]  |
|                                   |
| [Add DS Record] [Cancel]         |
+----------------------------------+
```

## Advanced Features

### Dynamic DNS (DDNS)

Enable DDNS for automatic updates:

```
+----------------------------------+
| Dynamic DNS                      |
+----------------------------------+
| DDNS Status: [Enabled]           |
| Token: [abc123...        ] [Regen]|
| Update URL:                       |
| https://api.example.com/ddns     |
|                                   |
| [Save]                           |
+----------------------------------+
```

### Traffic Routing

#### GeoDNS

```
+----------------------------------+
| GeoDNS Configuration            |
+----------------------------------+
| Enable GeoDNS: [Yes]             |
|                                   |
| Default Records:                  |
| Name: @ | Type: A | Value: 1.2.3.4|
|                                   |
| Geographic Rules:                 |
| +-----------------------------+  |
| | Region    | Record          |  |
| |-----------|-----------------|  |
| | US       | 192.0.2.1       |  |
| | EU       | 192.0.2.2       |  |
| | APAC     | 192.0.2.3       |  |
| +-----------------------------+  |
|                                   |
| [Add Rule] [Save]                 |
+----------------------------------+
```

#### Latency Routing

```
+----------------------------------+
| Latency Routing                  |
+----------------------------------+
| Enable: [Yes]                    |
|                                   |
| Monitor URL:                     |
| [https://your-app.com/health___] |
|                                   |
| Fallback: [Primary Record_____]  |
|                                   |
| [Save]                           |
+----------------------------------+
```

### Failover

```
+----------------------------------+
| DNS Failover                     |
+----------------------------------+
| Enable: [Yes]                    |
|                                   |
| Primary: 192.0.2.1 (www)         |
| Secondary: 192.0.2.2 (failover)  |
|                                   |
| Health Check:                     |
| Interval: [60 seconds____]       |
| Timeout: [10 seconds_____]       |
| Unhealthy Threshold: [3]         |
|                                   |
| [Save Configuration]              |
+----------------------------------+
```

## Templates

### Built-in Templates

| Template | Records |
|----------|---------|
| Standard Web | A (www, @), CNAME (www), MX, TXT (SPF) |
| Email Only | MX, SPF, DKIM, DMARC |
| Full Service | All standard + subdomain records |
| WordPress | A records for common WP paths |

### Custom Templates

Create custom templates:

**Template: My Business**

```json
{
  "name": "My Business",
  "records": [
    {"name": "@", "type": "A", "value": "192.0.2.1", "ttl": 3600},
    {"name": "www", "type": "A", "value": "192.0.2.1", "ttl": 3600},
    {"name": "mail", "type": "A", "value": "192.0.2.10", "ttl": 3600},
    {"name": "@", "type": "MX", "value": "mail.example.com", "priority": 10, "ttl": 3600}
  ]
}
```

## Validation Rules

### Record Validation

| Record Type | Validation |
|-------------|------------|
| A | Valid IPv4 address |
| AAAA | Valid IPv6 address |
| CNAME | Valid domain name, no conflict |
| MX | Valid domain name, priority 0-65535 |
| TXT | Max 255 chars per string (can chain) |
| SPF | Valid SPF syntax |
| DKIM | Valid DKIM public key |
| SRV | Valid service/protocol format |
| CAA | Valid flags (0/128), valid tag |

### Common Validation Errors

| Error | Cause | Solution |
|-------|-------|----------|
| CNAME conflict | Other records at same name | Remove conflicting records |
| Invalid IP | Malformed address | Check IP format |
| MX target invalid | Not a valid domain | Use fully qualified domain |
| TXT too long | Over 255 chars | Split into multiple strings |
| Duplicate record | Same name/type exists | Edit existing or use different name |

## Propagation

### Check Propagation Status

```
+----------------------------------+
| Propagation Status               |
+----------------------------------+
| Domain: example.com               |
| Last Updated: 2024-01-15 10:30   |
| Status: In Progress (48%)         |
|                                   |
| Global Check:                      |
| +-----------------------------+  |
| | Location    | Status         |  |
| |-------------|----------------|  |
| | US-East    | Propagated     |  |
| | US-West    | Propagated     |  |
| | EU         | Propagated     |  |
| | Asia       | Pending...      |  |
| +-----------------------------+  |
|                                   |
| [Refresh Status]                  |
+----------------------------------+
```

### Propagation Time

Typical propagation times:

| Record Type | Typical Time |
|-------------|-------------|
| A/AAAA | 5 minutes - 24 hours |
| CNAME | 5 minutes - 24 hours |
| MX | 15 minutes - 4 hours |
| TXT | 15 minutes - 48 hours |
| NS | 24 - 72 hours |

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Records not resolving | Propagation pending | Wait and check again |
| CNAME error | Conflict with other records | Remove conflicting records |
| SPF failing | Incorrect syntax | Validate SPF record |
| TTL too high | Slow updates | Reduce TTL before changes |
| DNSSEC invalid | Missing DS records | Add DS to parent zone |

### Debug Tools

| Tool | Description |
|------|-------------|
| DNS Lookup | Check current DNS resolution |
| Propagation Check | Global propagation status |
| Record Syntax Validator | Validate record syntax |
| Zone Health Check | Check for common issues |

## See Also

- [DNS Management API](./whmcs-dns-management-api.md)
- [Records Management](./whmcs-records-management.md)
- [DNS Templates](./whmcs-dns-templates.md)
