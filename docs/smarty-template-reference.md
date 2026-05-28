# WHMCS Smarty Template Reference
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Smarty template patterns and best practices for WHMCS.

## Variable Modifiers

```smarty
{$name|escape:'html'}
{$amount|number_format:2}
{$date|date_format:'%Y-%m-%d'}
{$text|truncate:50:'...'}
{$status|upper}
{$html|strip_tags}
```

## Control Structures

```smarty
{if $condition}
    Content
{elseif $other}
    Other content
{else}
    Default
{/if}

{foreach $items as $item}
    <li>{$item.name}</li>
{/foreach}

{foreach $items as $key => $value}
    <li>{$key}: {$value}</li>
{/foreach}
```

## Commonly Used WHMCS Variables

```smarty
{$client->firstname}
{$client->lastname}
{$client->email}

{$service->domain}
{$service->username}
{$service->password}
{$service->status}

{$invoice->total}
{$invoice->paymentmethod}
```

## Template Best Practices

```smarty
{*
 * Safe template coding
 * Always escape user data
 *}

{foreach $services as $service}
    <div class="service {if $service.active}active{else}inactive{/if}">
        <h3>{$service.domain|escape}</h3>
        <p>{$service.username|escape}</p>
    </div>
{/foreach}
```

## Including Other Templates

```smarty
{include file="$templatepath/header.tpl"}

{include file="{$module_template_path}/service_row.tpl"}
```

## Custom Smarty Functions

```php
// Register function
$smarty->registerFunction('currencyFormat', function($amount) {
    return number_format($amount, 2);
});

// Usage
{$amount|currencyFormat}
```

---

**Related Skills:**
- whmcs-template-styling
- whmcs-clientarea-builder
