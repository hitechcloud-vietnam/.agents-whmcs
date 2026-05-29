# WHMCS Domain Privacy Policy

## Overview

Domain privacy policy in WHMCS defines how customer data is handled and protected when using ID protection services.

## Privacy Policy

### Policy Configuration

**Configuration > Domains > Privacy Policy**

```php
// Privacy policy settings
[
    'policy_enabled' => true,
    'policy_url' => '/privacy-policy.php',
    'data_retention_days' => 365,
    'data_disclosure' => 'limited'
]
```

## Data Handling

### Information Shared

```php
// What is shared via WHOIS
[
    'with_consent' => [
        'legal_requests' => true,
        'transfer_process' => true
    ],
    'never_shared' => [
        'marketing' => true,
        'third_party_sale' => true
    ]
]
```

## Privacy Features

### Protection Features

```php
// Privacy features
[
    'email_masking' => true,
    'phone_masking' => true,
    'address_masking' => true,
    'proxy_email' => true
]
```

## Customer Consent

### Consent Requirements

```php
// Consent for privacy
[
    'require_consent' => true,
    'consent_text' => 'Enable WHOIS privacy protection',
    'consent_optional' => true
]
```

## API Functions

```php
// Get privacy settings
$result = localAPI('GetDomainPrivacySettings', [
    'domainid' => 1
]);
```

## Best Practices

1. **Clear policy**: Define what data is protected
2. **Consent**: Obtain customer consent
3. **Compliance**: Follow GDPR/CCPA
4. **Transparency**: Be clear about limitations

## Related Documentation

- [Domain ID Protection](./whmcs-domain-id-protection.md)
- [Domain Contacts](./whmcs-domain-contacts.md)
- [Domain Privacy Policy](./whmcs-domain-privacy-police.md)