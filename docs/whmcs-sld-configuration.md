# WHMCS SLD Configuration

## Overview

SLD (Second-Level Domain) configuration in WHMCS defines how the domain name portion before the TLD is handled, including validation and character restrictions.

## SLD Configuration

### Basic Settings

**Configuration > Domains > SLD Settings**

```php
// SLD settings
[
    'min_length' => 2,
    'max_length' => 63,
    'allow_international' => true,
    'validation' => 'standard'
]
```

## Character Rules

### Allowed Characters

```php
// SLD character rules
[
    'allowed_chars' => 'a-z0-9-',
    'allow_numeric_only' => false,
    'allow_single_char' => false,
    'no_start_end_hyphen' => true
]
```

## SLD Validation

### Validation Rules

```php
// Validate SLD
[
    'sld' => 'example',
    'valid' => true,
    'errors' => []
]

[
    'sld' => '-bad-',
    'valid' => false,
    'errors' => ['Cannot start or end with hyphen']
]
```

## Reserved Names

### Reserved SLDs

```php
// Block reserved names
[
    'reserved' => [
        'www', 'mail', 'ftp', 'admin',
        'localhost', 'webmail', 'smtp',
        'pop', 'ns1', 'ns2'
    ],
    'block_premium' => false
]
```

## SLD Requirements

### Requirements

```php
// SLD requirements
[
    'min_length' => 2,
    'max_length' => 63,
    'no_special_chars' => true,
    'alphanumeric_only' => false
]
```

## API Functions

```php
// Validate SLD
$result = localAPI('ValidateSLD', [
    'sld' => 'example',
    'tld' => 'com'
]);
```

## Best Practices

1. **Clear rules**: Define character restrictions
2. **Reserve names**: Block system names
3. **Length limits**: Set appropriate bounds
4. **Validation**: Check before registration

## Related Documentation

- [IDN Domains](./whmcs-idn-domains.md)
- [Domain Registration](./whmcs-domain-registration.md)
- [Domain Pricing](./whmcs-domain-pricing.md)