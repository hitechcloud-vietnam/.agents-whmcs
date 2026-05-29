# WHMCS Domain Privacy Service Documentation

## Overview

Domain privacy service (WHOIS privacy) protects domain registrants' personal information from public exposure. This documentation covers setup, configuration, and management of WHOIS privacy in WHMCS.

## What is Domain Privacy?

WHOIS privacy replaces public registrant information with generic proxy details while maintaining domain functionality. This service is essential for:

- Protecting personal address and phone information
- Reducing spam and unsolicited contact
- Meeting GDPR and privacy regulations
- Preventing domain-related identity theft

## Supported Privacy Services

### Built-in Privacy Options

1. **WHOIS Privacy** - Basic WHOIS masking
2. **ID Protect** - Enhanced privacy for EU domains
3. **Domain Privacy** - Generic proxy service
4. **GDPR Compliant** - European privacy standards

### Registrar-Specific Privacy

| Registrar | Privacy Service | TLD Support |
|-----------|-----------------|-------------|
| Enom | ID Protect | .com, .net, .org |
| GoDaddy | WHOIS Privacy | Global |
| Namecheap | WhoisGuard | All TLDs |
| OpenSRS | Privacy Protect | Major TLDs |
| ResellerClub | Domain Privacy | Asia-Pacific |

## Configuration

### Enable Privacy Service

1. Navigate to **Setup > Products/Services > Domain Pricing**
2. Select TLD row and click **Privacy Pricing**
3. Enable "Enable Privacy Protection" toggle
4. Set privacy service pricing
5. Configure default privacy preference

### Privacy Pricing Setup

```
+----------------------+------------+----------+
| TLD                  | Register   | Transfer |
+----------------------+------------+----------+
| .com                 | $8.99/yr   | $8.99/yr |
| .net                 | $9.99/yr   | $9.99/yr |
| .org                 | $8.99/yr   | $8.99/yr |
+----------------------+------------+----------+
```

### Global Privacy Settings

**Configuration > General Settings > Domains Tab:**

```php
// Domain Privacy Settings
'privacyAutoEnable' => true,           // Auto-enable for new registrations
'privacyDefaultPeriod' => 1,           // Privacy period in years
'privacyShowInCart' => true,          // Display in shopping cart
'privacyRequiredTLDs' => ['eu', 'de'], // TLDs requiring privacy
'privacyOptOutAllowed' => true,        // Allow customers to opt out
```

## Privacy Service Lifecycle

### Activation

When privacy is enabled:

1. WHMCS sends privacy activation request to registrar
2. Registrar replaces WHOIS data with proxy information
3. Confirmation received and logged
4. Customer notified of privacy activation

### Renewal

Privacy renewal follows domain renewal:

1. System detects domain approaching expiration
2. Privacy renewal quote generated with domain
3. Customer pays renewal invoice
4. Both domain and privacy renewed together
5. Privacy confirmation sent to customer

### Deactivation

Privacy can be disabled via:

- Customer request through support ticket
- Manual admin intervention
- Registrar policy changes
- TLD requirements (certain TLDs cannot have privacy)

## Customer Management

### Customer Privacy Portal

Customers can manage privacy from the Client Area:

**Client Area > My Domains > Domain Details > WHOIS Privacy**

Available actions:
- Enable/Disable privacy
- View current privacy status
- See privacy renewal date
- Request privacy removal
- Update contact preferences

### Privacy Consent

For GDPR compliance, maintain consent records:

```php
// Store privacy consent
$consentRecord = [
    'client_id' => 12345,
    'consent_type' => 'whois_privacy',
    'granted_at' => '2024-01-15T10:30:00Z',
    'ip_address' => '192.168.1.1',
    'consent_text' => 'I consent to WHOIS privacy service'
];
```

## Registrar Module Integration

### Privacy Command Interface

```php
/**
 * Activate WHOIS privacy for domain
 */
public function registerPrivacy($params)
{
    return [
        'success' => true,
        'privacy_id' => 'PRIV-' . $params['domain'] . '-' . time()
    ];
}

/**
 * Check privacy status
 */
public function getPrivacyStatus($params)
{
    return [
        'enabled' => true,
        'expires_at' => '2025-01-15',
        'proxy_email' => 'proxy@privacy-service.com'
    ];
}

/**
 * Disable privacy
 */
public function disablePrivacy($params)
{
    return [
        'success' => true,
        'message' => 'Privacy disabled successfully'
    ];
}
```

## Privacy for Specific TLDs

### EU Domains (.eu, .de)

EU domains have specific privacy requirements:

| Country | Privacy Allowed | Notes |
|---------|-----------------|-------|
| .eu | Conditional | GDPR compliant only |
| .de | No | DENIC requires real data |
| .it | No | Italian law requires real data |
| .fr | No | French law requires real data |

### ICANN-Required Contacts

Some TLDs require at least one visible contact:

- `.com`, `.net` - Full privacy allowed
- `.org` - Full privacy allowed
- `.biz` - Admin contact must be visible
- `.info` - Full privacy allowed

## Privacy Removal Requests

### Legitimate Reason Requests

Process privacy removal requests for:

1. Legal proceedings
2. Trademark disputes
3. Law enforcement requests
4. Security investigations

### Request Workflow

1. Customer submits privacy removal request
2. Admin reviews request with documentation
3. Approve or deny with explanation
4. If approved, execute registrar API call
5. Log all privacy bypass events

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Privacy not available | TLD restriction | Check TLD registry rules |
| Activation failed | Registrar API error | Check API credentials |
| Privacy shows expired | Sync issue | Force registrar sync |
| Cannot remove privacy | Pending transfer | Wait for transfer completion |

### Debug Commands

```bash
# Check privacy status
whmcscli domain privacy-status --domain=example.com

# Force privacy sync
whmcscli domain sync-privacy --domain=example.com

# List all private domains
whmcscli domain list-privacy --status=all
```

## Compliance and Legal

### GDPR Considerations

- Document legal basis for privacy service
- Maintain consent records
- Provide data export capabilities
- Implement right to be forgotten procedures

### Privacy Policy Requirements

Update your privacy policy to include:

- WHOIS privacy service description
- Data retention periods
- Third-party data sharing
- Customer rights under GDPR

## Pricing Strategies

### Privacy Pricing Models

| Model | Description | Example Price |
|-------|-------------|---------------|
| Bundled | Included with registration | Free |
| Optional Add-on | Per-year pricing | $8.99/year |
| Lifetime | One-time lifetime privacy | $49.99 |
| Tiered | Price by TLD | .com $8.99, .net $9.99 |

### Promotional Pricing

```php
// privacy_promotions config
$promotions = [
    'first_year_free' => true,
    'bulk_discount' => 0.15, // 15% off for 5+ domains
    'lifetime_offer' => 49.99
];
```

## See Also

- [Domain Pricing Configuration](../products-services/domain-pricing.md)
- [GDPR Compliance Guide](../compliance/gdpr-guide.md)
- [Registrar Module Development](../developer/registrar-modules.md)
