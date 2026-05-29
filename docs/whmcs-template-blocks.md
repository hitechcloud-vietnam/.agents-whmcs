# WHMCS Template Blocks

## Overview

Blocks in WHMCS templates are sections that can be defined and overridden in child templates. They are fundamental to template inheritance and component organization.

## Basic Syntax

### Defining Blocks

```smarty
{block name="blockname"}
    Default block content
{/block}
```

### Block with HTML

```smarty
{block name="sidebar"}
    <aside class="sidebar">
        <h3>Quick Links</h3>
        <ul>
            <li><a href="#">Link 1</a></li>
            <li><a href="#">Link 2</a></li>
        </ul>
    </aside>
{/block}
```

## Block Options

### append

Add content after parent block.

```smarty
{block name="scripts" append}
    <script src="extra.js"></script>
{/block}
```

### prepend

Add content before parent block.

```smarty
{block name="head" prepend}
    <meta name="keywords" content="hosting, domains">
{/block}
```

### hide

Hide the block but keep for reference.

```smarty
{block name="debug" hide}
    <div class="debug">
        {$debug_info}
    </div>
{/block}
```

### nocache

Exclude from template caching.

```smarty
{block name="user-specific" nocache}
    Welcome, {$user->name}!
{/block}
```

## Block Nesting

### Nested Blocks

```smarty
{block name="content"}
    <article>
        <header>
            {block name="article-header"}
                <h1>Default Title</h1>
            {/block}
        </header>
        <div class="body">
            {block name="article-body"}
                Default content
            {/block}
        </div>
    </article>
{/block}
```

### Override Nested Blocks

```smarty
{block name="article-header"}
    <h1>Custom Title</h1>
    <p>Subtitle</p>
{/block}

{block name="article-body"}
    <p>Custom content goes here.</p>
{/block}
```

## Block Variables

### $smarty.block.* Variables

```smarty
{block name="myblock"}
    Parent content: {$smarty.block.myblock}
{/block}
```

### Accessing Parent Content

```smarty
{block name="content"}
    {$smarty.block.parent}
{/block}
```

## WHMCS-Specific Blocks

### Standard Page Blocks

```smarty
{block name="pagetitle"}
    {$pagetitle}
{/block}

{block name="breadcrumb"}
    {if $breadcrumb}
        {foreach $breadcrumb as $crumb}
            <a href="{$crumb.url}">{$crumb.label}</a>
        {/foreach}
    {/if}
{/block}

{block name="sidebar"}
    {include file="$template/includes/sidebar.tpl"}
{/block}
```

### Form Blocks

```smarty
{block name="form-fields"}
    <div class="form-group">
        <label>Name</label>
        <input type="text" name="name">
    </div>
{/block}

{block name="form-actions"}
    <button type="submit" class="btn btn-primary">
        Submit
    </button>
{/block}
```

## Block in Hooks

### Output Hook Integration

```smarty
{block name="after-content"}
    {hook name="ClientAreaPageFooter"}
{/block}
```

### Multiple Hook Points

```smarty
{hook file="output_output.tpl" point="ClientAreaPageHeader"}
{hook file="output_output.tpl" point="ClientAreaPageSidebar"}
{hook file="output_output.tpl" point="ClientAreaPageContent"}
```

## Common Block Patterns

### Alert Messages

```smarty
{block name="alerts"}
    {if $success}
        <div class="alert alert-success">
            {$success}
        </div>
    {/if}
    {if $error}
        <div class="alert alert-danger">
            {$error}
        </div>
    {/if}
    {if $warning}
        <div class="alert alert-warning">
            {$warning}
        </div>
    {/if}
{/block}
```

### Product Configuration

```smarty
{block name="product-features"}
    <ul class="features-list">
        {foreach $product->features as $feature}
            <li>{$feature}</li>
        {/foreach}
    </ul>
{/block}

{block name="product-pricing"}
    <div class="pricing-table">
        {foreach $pricing as $cycle => $price}
            <div class="price-option">
                <span class="cycle">{$cycle}</span>
                <span class="price">{$price|currency_format}</span>
            </div>
        {/foreach}
    </div>
{/block}
```

### Table Display

```smarty
{block name="table-header"}
    <thead>
        <tr>
            <th>Column 1</th>
            <th>Column 2</th>
        </tr>
    </thead>
{/block}

{block name="table-rows"}
    <tbody>
        {foreach $rows as $row}
            <tr>
                <td>{$row.col1}</td>
                <td>{$row.col2}</td>
            </tr>
        {/foreach}
    </tbody>
{/block}
```

## Dynamic Block Content

### Conditional Blocks

```smarty
{block name="status-badge"}
    {if $item->status eq 'active'}
        <span class="badge badge-success">Active</span>
    {elseif $item->status eq 'pending'}
        <span class="badge badge-warning">Pending</span>
    {else}
        <span class="badge badge-secondary">{$item->status}</span>
    {/if}
{/block}
```

### Loop-Based Blocks

```smarty
{block name="service-list"}
    {foreach $services as $service}
        <div class="service-item">
            <span class="domain">{$service->domain}</span>
            <span class="status">{$service->status}</span>
        </div>
    {/foreach}
{/block}
```

## Block Best Practices

1. **Use descriptive names** - `sidebar`, `content`, `footer`
2. **Provide defaults** - Include fallback content
3. **Avoid deep nesting** - Maximum 2-3 levels
4. **Keep blocks focused** - One purpose per block
5. **Document complex blocks** - Add comments

## See Also

- [Template Inheritance](../whmcs-template-inheritance.md)
- [Template Extend](../whmcs-template-extend.md)
- [Template Include](../whmcs-template-include.md)