# WHMCS Domain Park Page

## Overview

Domain parking in WHMCS displays a placeholder page for domains that aren't actively used, often showing ads or search results to generate revenue.

## Parking Configuration

### Enable Parking

**Configuration > Domains > Parking**

```php
// Parking settings
[
    'enable_parking' => true,
    'parked_page_template' => 'default',
    'show_ads' => true,
    'ad_provider' => 'google_adsense',
    'ad_slot_id' => 'xxx'
]
```

## Parking Templates

### Template Options

```php
// Parking page template
[
    'template' => 'search',
    'search_provider' => 'google',
    'show_categories' => true,
    'show_related' => true
]
```

## Parking Setup

### Park Domain

```php
// Park domain
[
    'domain_id' => 1,
    'parked' => true,
    'template' => 'search',
    'categories' => ['business', 'technology'],
    'ads_enabled' => true
]
```

## Parking Features

### Page Features

```php
// Parking page features
[
    'search_box' => true,
    'categories' => true,
    'related_links' => true,
    'ads' => true,
    'contact_link' => true
]
```

## Unparking

### Activate Domain

```php
// Remove parking
[
    'domain_id' => 1,
    'parked' => false,
    'point_to_hosting' => true
]
```

## API Functions

```php
// Park domain
$result = localAPI('ParkDomain', [
    'domainid' => 1
]);

// Unpark domain
$result = localAPI('UnparkDomain', [
    'domainid' => 1
]);
```

## Best Practices

1. **Monetize unused**: Generate revenue from spare domains
2. **Good templates**: Use professional parking templates
3. **SEO-friendly**: Parked pages should be crawlable
4. **Clear contact**: Include way to purchase

## Related Documentation

- [Domain Forwarding](./whmcs-domain-forwarding.md)
- [Domain Registration](./whmcs-domain-registration.md)
- [Domain Sync](./whmcs-domain-sync.md)