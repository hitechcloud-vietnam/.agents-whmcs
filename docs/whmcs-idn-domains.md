# WHMCS IDN Domains

## Overview

IDN (Internationalized Domain Name) support in WHMCS allows registration of domains with non-ASCII characters, supporting multiple languages and scripts.

## IDN Configuration

### Enable IDN

**Configuration > Domains > IDN Support**

```php
// IDN settings
[
    'enable_idn' => true,
    'allowed_scripts' => ['latin', 'cyrillic', 'greek', 'arabic', 'cjk'],
    'punycode_required' => true
]
```

## Supported Scripts

### Language Support

| Script | Languages | Example |
|--------|-----------|---------|
| Latin | English, European | cafe.com |
| Cyrillic | Russian, Bulgarian | музыка.рф |
| Greek | Greek | παραδειγμα.gr |
| Arabic | Arabic | موقع.امارات |
| CJK | Chinese, Japanese, Korean | 中文.cn |

## IDN Conversion

### Punycode Conversion

```php
// Convert to punycode
[
    'idn_domain' => 'münchen.de',
    'punycode' => 'xn--mnchen-3ya.de',
    'registrar_format' => 'xn--mnchen-3ya.de'
]
```

## IDN Registration

### Register IDN

```php
// Register international domain
[
    'domain' => 'موقع.امارات',
    'punycode' => 'xn--4sb8ac.com',
    'script' => 'arabic',
    'language' => 'ar'
]
```

## IDN Pricing

### Same as Standard

```php
// Pricing for IDN
[
    'tld' => 'de',
    'idn_price' => 9.95,          // Same as standard
    'registration' => 9.95
]
```

## IDN Validation

### Validate Input

```php
// Validate IDN domain
[
    'domain' => 'пример.рф',
    'valid' => true,
    'script' => 'cyrillic',
    'punycode' => 'xn--e1afmkfd.ru'
]
```

## API Functions

```php
// Check IDN availability
$result = localAPI('CheckIDNDomain', [
    'domain' => 'münchen.de'
]);

// Convert to punycode
$result = localAPI('ConvertToPunycode', [
    'domain' => 'münchen.de'
]);
```

## Best Practices

1. **Enable IDN**: Support international customers
2. **Validate input**: Check domain syntax
3. **Convert properly**: Use punycode for registrar
4. **Show pricing**: IDN pricing should be clear

## Related Documentation

- [Domain Registration](./whmcs-domain-registration.md)
- [SLD Configuration](./whmcs-sld-configuration.md)
- [TLD Sync](./whmcs-tld-sync.md)