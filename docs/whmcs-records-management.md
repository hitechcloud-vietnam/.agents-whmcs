# WHMCS DNS Records Management Reference

## Overview

This document provides detailed reference information for all DNS record types supported by WHMCS, including specifications, validation rules, and best practices.

## Record Type Reference

### A Record (Address)

**Purpose:** Maps a domain name to an IPv4 address.

**RFC:** RFC 1035

**Syntax:**
```
Name TTL Class Type Value
www 3600 IN A 192.0.2.1
```

**Parameters:**

| Field | Description | Constraints |
|-------|-------------|-------------|
| Name | Subdomain label | 1-63 characters, alphanumeric, hyphens |
| TTL | Time to live | 0-2147483647 seconds |
| Value | IPv4 address | Dotted decimal notation |

**Validation:**
- Must be valid IPv4 (0.0.0.0 to 255.255.255.255)
- Cannot be 0.0.0.0 or 255.255.255.255 (usually invalid)

**Examples:**
```
@    3600 IN A 192.0.2.1
www  3600 IN A 192.0.2.1
blog 3600 IN A 192.0.2.10
*    3600 IN A 192.0.2.1
```

**Common Uses:**
- Pointing main domain to web server
- Subdomain hosting
- Wildcard subdomains

### AAAA Record (IPv6 Address)

**Purpose:** Maps a domain name to an IPv6 address.

**RFC:** RFC 3596

**Syntax:**
```
Name TTL Class Type Value
www 3600 IN AAAA 2001:db8::1
```

**Parameters:**

| Field | Description | Constraints |
|-------|-------------|-------------|
| Name | Subdomain label | 1-63 characters |
| Value | IPv6 address | 8 groups of 16-bit values, :: compression allowed |

**Validation:**
- Must be valid IPv6 address
- Compressed format allowed (::1, 2001:db8::)
- Full expansion: 2001:0db8:0000:0000:0000:0000:0000:0001

**Examples:**
```
www  3600 IN AAAA 2001:db8::1
mail 3600 IN AAAA 2001:db8::10
ipv6 3600 IN AAAA fe80::1
```

### CNAME Record (Canonical Name)

**Purpose:** Creates an alias from one domain to another.

**RFC:** RFC 1035

**Syntax:**
```
Name TTL Class Type Value
blog 3600 IN CNAME myblog.wordpress.com
```

**Parameters:**

| Field | Description | Constraints |
|-------|-------------|-------------|
| Name | Alias subdomain | Cannot be @ or root |
| Value | Target domain | Must be fully qualified |

**Important Rules:**
1. CNAME cannot coexist with other record types at the same name
2. CNAME cannot point to an IP address
3. CNAME chain maximum recommended: 2 hops
4. Cannot create CNAME at root (@) with some providers

**Validation:**
- Target must be valid domain name
- Cannot create circular references (A points to B, B points to A)
- Cannot chain more than recommended limit

**Examples:**
```
blog 3600 IN CNAME myblog.wordpress.com
shop 3600 IN CNAME mystore.shopify.com
cdn  3600 IN CNAME cdn.cloudprovider.net
```

**Anti-Patterns:**
```
BAD: @    IN CNAME example.com     (MX also exists)
BAD: www  IN CNAME 192.0.2.1       (Points to IP)
BAD: a    IN CNAME b               (Chain too long)
```

### MX Record (Mail Exchange)

**Purpose:** Specifies mail servers for receiving email.

**RFC:** RFC 1035

**Syntax:**
```
Name TTL Class Type Priority Value
@    3600 IN MX  10      mail.example.com
```

**Parameters:**

| Field | Description | Constraints |
|-------|-------------|-------------|
| Name | Target subdomain | Usually @ for root |
| Priority | Server priority | 0-65535, lower = higher priority |
| Value | Mail server hostname | Must be A or AAAA record |

**Priority Guidelines:**
- Primary server: 10
- Secondary server: 20
- Tertiary server: 30
- Lower numbers are tried first

**Examples:**
```
@    3600 IN MX  10 mail.example.com
@    3600 IN MX  20 mail2.example.com
@    3600 IN MX  30 mail3.example.com
```

**Best Practices:**
- Always have at least 2 MX records
- Use distinct priorities
- Ensure mail servers are reliable
- Include TTL considerations for failover

### TXT Record

**Purpose:** Free-form text for various purposes.

**RFC:** RFC 1035

**Syntax:**
```
Name TTL Class Type Value
@    3600 IN TXT "v=spf1 mx -all"
```

**Parameters:**

| Field | Description | Constraints |
|-------|-------------|-------------|
| Name | Target subdomain | Usually @ for root |
| Value | Text content | Max 255 characters per string; multiple strings allowed |

**Multiple Strings:**
```
@    3600 IN TXT "part1" "part2" "part3"
```

**Common Uses:**
- SPF (Sender Policy Framework)
- DKIM (DomainKeys Identified Mail)
- DMARC (Domain-based Message Authentication)
- Domain verification
- General information

**Validation:**
- 255 character limit per string
- Escape quotes within value
- UTF-8 allowed for verification records

### SPF Record (Sender Policy Framework)

**Purpose:** Authorize servers to send email for your domain.

**RFC:** RFC 7208

**Syntax:**
```
v=spf1 [mechanisms] [qualifiers] [directive]
```

**Mechanisms:**

| Mechanism | Description | Example |
|-----------|-------------|---------|
| `a` | Match A/AAAA of domain | `a` or `a:example.com` |
| `mx` | Match MX servers | `mx` or `mx:example.com` |
| `ip4` | Match IPv4 | `ip4:192.0.2.0/24` |
| `ip6` | Match IPv6 | `ip6:2001:db8::/32` |
| `include` | Include external SPF | `include:_spf.google.com` |
| `all` | End of rule | `-all`, `~all`, `+all` |

**Qualifiers:**

| Qualifier | Action | Symbol |
|-----------|--------|--------|
| Pass | Allow | `+` (default) |
| Fail | Reject | `-` |
| SoftFail | Accept but mark | `~` |
| Neutral | No assertion | `?` |

**Examples:**
```
v=spf1 mx -all
v=spf1 include:_spf.google.com mx -all
v=spf1 ip4:192.0.2.0/24 mx -all
v=spf1 a mx include:_spf.spf-example.com ~all
```

**Best Practices:**
- Keep under 10 DNS lookups (include + mx + a = lookups)
- Use `-all` (fail) not `~all` (softfail)
- Test before deployment with `-all`
- Monitor bounce rates after changes

### DKIM Record

**Purpose:** Email authentication via public key cryptography.

**RFC:** RFC 6376

**Selector Format:**
```
selector._domainkey.domain.com. 3600 IN TXT "v=DKIM1; k=rsa; p=public_key..."
```

**Parameters:**

| Field | Description |
|-------|-------------|
| Selector | Identifies DKIM key (usually provider name) |
| `v=DKIM1` | Protocol version |
| `k=` | Key type (rsa, ed25519) |
| `p=` | Public key (Base64 encoded) |

**Common Selectors:**
```
google._domainkey
mail._domainkey
dkim
selector1
```

**Examples:**
```
google._domainkey.example.com. 3600 IN TXT "v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8..."
mail._domainkey.example.com. 3600 IN TXT "v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8..."
```

### DMARC Record

**Purpose:** Reporting and policy for email authentication failures.

**RFC:** RFC 7489

**Syntax:**
```
_dmarc.domain.com. 3600 IN TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@example.com"
```

**Parameters:**

| Tag | Purpose | Values |
|-----|---------|--------|
| `v` | Version | DMARC1 (required) |
| `p` | Policy | none, quarantine, reject |
| `rua` | Aggregate reports | mailto:email |
| `ruf` | Forensic reports | mailto:email |
| `pct` | Percentage | 0-100 |
| `sp` | Subdomain policy | none, quarantine, reject |
| `adkim` | DKIM alignment | r (relaxed), s (strict) |
| `aspf` | SPF alignment | r (relaxed), s (strict) |

**Examples:**
```
v=DMARC1; p=none; rua=mailto:dmarc@example.com
v=DMARC1; p=quarantine; pct=25; rua=mailto:dmarc@example.com
v=DMARC1; p=reject; rua=mailto:dmarc@example.com; ruf=mailto:dmarc-forensic@example.com
```

### NS Record (Nameserver)

**Purpose:** Delegates a DNS zone to use specific nameservers.

**RFC:** RFC 1035

**Syntax:**
```
@ 3600 IN NS ns1.registrar.com
@ 3600 IN NS ns2.registrar.com
subdomain 3600 IN NS ns.subdomain.com
```

**Common Uses:**
- Point domain to hosting nameservers
- Delegate subdomain to different DNS
- Glue records for nameserver addresses

**Examples:**
```
@         3600 IN NS ns1.webhost.com.
@         3600 IN NS ns2.webhost.com.
api       3600 IN NS ns.api-provider.com.
cdn       3600 IN NS ns.cdn-provider.com.
```

### SRV Record (Service)

**Purpose:** Defines location of specific services.

**RFC:** RFC 2782

**Syntax:**
```
_service._protocol.name. TTL Class SRV Priority Weight Port Target
_sip._tcp            3600 IN SRV  10       5     5060  sip.example.com
_xmpp._tcp           3600 IN SRV  10       0     5222  xmpp.example.com
```

**Parameters:**

| Field | Description | Constraints |
|-------|-------------|-------------|
| Service | Service name | _ prefix, e.g., _sip, _http |
| Protocol | Transport protocol | _tcp, _udp |
| Priority | Priority (like MX) | 0-65535 |
| Weight | Relative weight | 0-65535 |
| Port | Service port | 0-65535 |
| Target | Server hostname | Must have A/AAAA record |

**Common SRV Records:**

| Service | Port | Typical Use |
|---------|------|-------------|
| _sip._tcp | 5060 | SIP VoIP |
| _xmpp-client._tcp | 5222 | XMPP chat |
| _xmpp-server._tcp | 5269 | XMPP server |
| _ldap._tcp | 389 | Active Directory |
| _kerberos._tcp | 88 | Kerberos |
| _autodiscover._tcp | 443 | Exchange AutoDiscover |
| _minecraft._tcp | 25565 | Minecraft servers |

**Examples:**
```
_sip._tcp      3600 IN SRV 10 5  5060 sip.example.com
_sip._tcp      3600 IN SRV 20 5  5060 sip2.example.com
_autodiscover._tcp 3600 IN SRV 0 0 443 autodiscover.example.com
```

### CAA Record (Certification Authority Authorization)

**Purpose:** Specifies which CAs can issue certificates.

**RFC:** RFC 6844

**Syntax:**
```
name TTL Class Type Flags Tag Value
@    3600 IN CAA 0 issue "letsencrypt.org"
@    3600 IN CAA 0 issuewild "letsencrypt.org"
@    3600 IN CAA 0 iodef "mailto:security@example.com"
```

**Parameters:**

| Field | Description | Constraints |
|-------|-------------|-------------|
| Flags | Critical flag | 0 (standard) or 128 (critical) |
| Tag | Property | issue, issuewild, iodef |
| Value | Property value | CA domain or URL |

**Tag Meanings:**
- `issue` - CA may issue certs for domain
- `issuewild` - CA may issue wildcard certs
- `iodef` - URL for reporting violations

**Examples:**
```
@ 3600 IN CAA 0 issue "letsencrypt.org"
@ 3600 IN CAA 0 issue "digicert.com"
@ 3600 IN CAA 0 issuewild "letsencrypt.org"
@ 3600 IN CAA 0 iodef "mailto:caa@example.com"
```

### SOA Record (Start of Authority)

**Purpose:** Contains administrative information about zone.

**RFC:** RFC 1035

**Syntax:**
```
@ 3600 IN SOA ns1.provider.com. admin.example.com. (
     2024011501 ; Serial
     7200       ; Refresh
     3600       ; Retry
     1209600    ; Expire
     86400 )    ; Minimum TTL
```

**Parameters:**

| Field | Description | WHMCS Default |
|-------|-------------|---------------|
| MNAME | Primary nameserver | Set by provider |
| RNAME | Administrator email | admin@domain.com |
| Serial | Zone version | YYYYMMDDNN |
| Refresh | Secondary sync interval | 7200 (2 hours) |
| Retry | Failed sync retry | 3600 (1 hour) |
| Expire | Stale data retention | 1209600 (14 days) |
| Minimum | Negative cache TTL | 86400 (1 day) |

**Serial Number:**
- Format: YYYYMMDDNN (YearMonthDaySequence)
- Increment on every zone change
- Secondary DNS checks this to determine if zone changed

### PTR Record (Pointer)

**Purpose:** Reverse DNS lookup (IP to domain).

**RFC:** RFC 1035

**Common Use:**
- Email server verification (PTR matches SMTP banner)
- Spam filtering
- Logging and forensics

**Example:**
```
1.2.0.192.in-addr.arpa. 3600 IN PTR mail.example.com.
```

**Note:** PTR records are typically managed by the IP owner (ISP or hosting provider).

## Record Limits

| Record Type | Per-Zone Limit | Notes |
|-------------|----------------|-------|
| A | 500 | Per subdomain |
| AAAA | 500 | Per subdomain |
| CNAME | 1 | Per subdomain |
| MX | 100 | Per domain |
| TXT | 100 | Per name |
| SPF | 1 | Per name (uses TXT) |
| DKIM | 50 | Per selector |
| DMARC | 1 | Per domain |
| SRV | 500 | Per service/protocol |
| NS | 50 | Per zone (excluding delegation) |
| CAA | 100 | Per name |
| SOA | 1 | Per zone |

## TTL Guidelines

| TTL Value | Duration | Use Case |
|-----------|----------|----------|
| 300 | 5 minutes | Frequently changing |
| 3600 | 1 hour | Standard (default) |
| 7200 | 2 hours | Stable records |
| 14400 | 4 hours | Slow propagation acceptable |
| 86400 | 24 hours | Very stable |
| 604800 | 1 week | Static/minimal changes |

**Best Practices:**
- Reduce TTL to 300 before planned changes
- Wait for full propagation before increasing TTL
- Use 3600 as standard TTL

## Record Validation

### IP Address Validation

**IPv4 (A Record):**
```php
function isValidIPv4(string $ip): bool {
    return filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_IPV4) !== false;
}
```

**IPv6 (AAAA Record):**
```php
function isValidIPv6(string $ip): bool {
    return filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_IPV6) !== false;
}
```

### Domain Name Validation

```php
function isValidDomain(string $domain): bool {
    return preg_match('/^(?!:\/\/)(?=.{1,255}$)((.{1,63}\.){1,127}|[a-f0-9:]{1,63}:[a-f0-9:]{1,63})$/i', $domain) === 1;
}
```

### SPF Validation

```php
function isValidSPF(string $spf): bool {
    // Check basic structure
    if (!str_starts_with($spf, 'v=spf1 ')) {
        return false;
    }
    // Check mechanism count (recommend < 10 DNS lookups)
    // Check for valid mechanisms
    return true;
}
```

## Special Characters

| Character | Usage | Escaping |
|-----------|-------|----------|
| `.` | Domain separator | `\.` in regex |
| `*` | Wildcard | No escaping needed |
| `@` | Root/apex | Use `@` or leave blank |
| `\` | Escape | Double for `\\` |
| `"` | String delimiter | `\"` in TXT values |
| `;` | Comment | Use in quoted strings only |

## See Also

- [DNS Management API](./whmcs-dns-management-api.md)
- [Zone Editor](./whmcs-zone-editor.md)
- [TTL Management](./whmcs-ttl-management.md)
