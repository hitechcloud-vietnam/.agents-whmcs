# WHMCS Internationalization Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for making WHMCS modules internationally compatible.

## When to Use

- Supporting multiple currencies
- Handling multiple timezones
- Supporting RTL languages

## i18n Patterns

### Currency Handling

```php
function formatAmount(float $amount, string $currency = 'VND'): string {
    $symbols = [
        'VND' => '₫',
        'USD' => '$',
        'EUR' => '€',
    ];

    $symbol = $symbols[$currency] ?? '';

    switch ($currency) {
        case 'VND':
            return $symbol . number_format($amount, 0, ',', '.');
        case 'USD':
        case 'EUR':
            return $symbol . number_format($amount, 2, '.', ',');
        default:
            return number_format($amount, 2) . ' ' . $currency;
    }
}
```

### Date/Timezone Handling

```php
function formatDateTime(string $datetime, string $timezone = 'Asia/Ho_Chi_Minh'): string {
    $dt = new DateTime($datetime, new DateTimeZone('UTC'));
    $dt->setTimezone(new DateTimeZone($timezone));

    return $dt->format('Y-m-d H:i:s');
}

function convertToTimezone(string $datetime, string $fromTz, string $toTz): string {
    $dt = new DateTime($datetime, new DateTimeZone($fromTz));
    $dt->setTimezone(new DateTimeZone($toTz));
    return $dt->format('Y-m-d H:i:s');
}
```

### RTL Support

```smarty
<div class="module-content" dir="{$LANG.direction|default:'ltr'}">
    {if $LANG.direction eq 'rtl'}
    <style>
        .module-container { direction: rtl; }
        .btn { margin-left: 10px; margin-right: 0; }
    </style>
    {/if}
</div>
```

---

**Related Skills:**
- whmcs-multilanguage-support
- whmcs-template-styling
- whmcs-clientarea-builder