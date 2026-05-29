# WHMCS Smarty Templates

## Overview

Smarty is WHMCS's templating engine for generating dynamic HTML content.

## Basic Syntax

```smarty
{* This is a comment *}

{* Variable output *}
{$client->fullName}
{$product->name}

{* Function call *}
{foreach from=$products item=product}
    <div>{$product.name}</div>
{/foreach}

{* Conditional *}
{if $client->status === 'Active'}
    <span class="badge badge-success">Active</span>
{else}
    <span class="badge badge-secondary">Inactive</span>
{/if}
```

## Template Variables

### Available Variables

```smarty
{* Client variables *}
{$client->id}
{$client->email}
{$client->firstname}
{$client->lastname}
{$client->companyname}
{$client->fullName}
{$client->status}
{$client->created_at}

{* Service variables *}
{$service->id}
{$service->domain}
{$service->username}
{$service->password} {* Encrypted *}
{$service->status}
{$service->nextduedate}

{* Invoice variables *}
{$invoice->id}
{$invoice->total}
{$invoice->status}
{$invoice->duedate}
{$invoice->datecreated}
```

## Smarty Modifiers

```smarty
{* String modifiers *}
{$client->email|lower}
{$client->fullName|upper}
{$client->companyname|escape}
{$client->firstname|truncate:20:"..."}

{* Date formatting *}
{$service->nextduedate|date_format:"%d %b %Y"}
{$invoice->datecreated|date_format}

{* Number formatting *}
{$invoice->total|string_format:"%.2f"}
{$amount|currency}

{* Array modifiers *}
{count($items)}
{implode(', ', $tags)}
```

## Control Structures

### Foreach Loop

```smarty
{foreach from=$invoices item=invoice}
    <tr>
        <td>{$invoice.id}</td>
        <td>{$invoice.total|currency}</td>
        <td>{$invoice.status}</td>
        <td>{$invoice.duedate|date_format}</td>
    </tr>
{foreachelse}
    <tr>
        <td colspan="4">No invoices found</td>
    </tr>
{/foreach}
```

### For Loop

```smarty
{for $i=1 to 10}
    <option value="{$i}">{$i}</option>
{/for}
```

### Section Loop

```smarty
{section name=idx loop=$items}
    <div class="item">
        {$items[idx].name}
    </div>
{/section}
```

## Template Inheritance

```smarty
{* layout.tpl - Base template *}
<html>
<head>
    <title>{block name="title"}Default Title{/block}</title>
    {block name="head"}{/block}
</head>
<body>
    <div class="header">
        {block name="header"}Default Header{/block}
    </div>
    
    <div class="content">
        {block name="content"}Default Content{/block}
    </div>
    
    <div class="footer">
        {block name="footer"}Default Footer{/block}
    </div>
</body>
</html>

{* page.tpl - Child template *}
{extends file="layout.tpl"}

{block name="title"}Page Title{/block}
{block name="content"}
    <p>Page content here</p>
{/block}
```

## Template Functions

### Include

```smarty
{* Include another template *}
{include file="modules/addons/yourmodule/templates/partial.tpl"}

{* Include with variables *}
{include file="table.tpl" items=$invoices title="Invoice List"}
```

### Capture

```smarty
{capture name="sidebar"}
    <div class="sidebar">
        {foreach from=$sidebarItems item=item}
            <a href="{$item.url}">{$item.name}</a>
        {/foreach}
    </div>
{/capture}

<div class="main-content">
    {$smarty.capture.sidebar}
</div>
```

### Literal

```smarty
{literal}
<script>
    // JavaScript code that shouldn't be parsed
    document.getElementById('test').innerHTML = '{$client->name}';
</script>
{/literal}
```

## Custom Functions

### Registering Custom Functions

```php
<?php
// In module or hooks
$whmcs->smarty->registerFunction('widget', function($params) {
    return renderWidget($params['name'], $params['data'] ?? []);
});

$whmcs->smarty->registerFunction('formatCurrency', function($params) {
    return format_currency($params['amount'], $params['currency'] ?? null);
});
```

### Custom Block Function

```php
<?php
$smarty->registerBlock('markdown', function($params, $content, $template, &$repeat) {
    if (!$repeat) {
        return parseMarkdown($content);
    }
});
```

```smarty
{markdown}
# Heading
This is **bold** text.
{/markdown}
```

## Template Best Practices

```smarty
{* Use strict comparison *}
{if $variable === 'value'}
{/if}

{* Check variable exists *}
{if isset($variable)}
{/if}

{* Check not empty *}
{if !empty($items)}
{/if}

{* Use else for multiple conditions *}
{if $status === 'Active'}
    <span>Active</span>
{elseif $status === 'Suspended'}
    <span>Suspended</span>
{else}
    <span>Unknown</span>
{/if}

{* Escape user data *}
{$userInput|escape} {* For HTML *}
{$userInput|escape:'htmlall'} {* For all *}
```

## Common Patterns

### Data Table

```smarty
<table class="table">
    <thead>
        <tr>
            <th>ID</th>
            <th>Client</th>
            <th>Amount</th>
            <th>Status</th>
            <th>Actions</th>
        </tr>
    </thead>
    <tbody>
        {foreach from=$invoices item=invoice}
            <tr>
                <td>#{$invoice.id}</td>
                <td>{$invoice.client->fullName}</td>
                <td>{$invoice.total|currency}</td>
                <td>
                    <span class="badge badge-{$invoice.status|lower}">
                        {$invoice.status}
                    </span>
                </td>
                <td>
                    <a href="viewinvoice.php?id={$invoice.id}">View</a>
                </td>
            </tr>
        {/foreach}
    </tbody>
</table>
```

### Pagination

```smarty
{if $totalPages > 1}
    <nav class="pagination">
        {if $currentPage > 1}
            <a href="?page={$currentPage - 1}">Previous</a>
        {/if}
        
        {for $page=1 to $totalPages}
            <a href="?page={$page}" 
               class="{if $page == $currentPage}active{/if}">
                {$page}
            </a>
        {/for}
        
        {if $currentPage < $totalPages}
            <a href="?page={$currentPage + 1}">Next</a>
        {/if}
    </nav>
{/if}
```

## Related Documentation

- [WHMCS Client Area](/docs/whmcs-client-area.md)
- [WHMCS CSS Customization](/docs/whmcs-css-customization.md)