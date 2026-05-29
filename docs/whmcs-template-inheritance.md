# WHMCS Template Inheritance

## Overview

Template inheritance in WHMCS allows you to create a base template that child templates extend, enabling consistent layouts across your WHMCS installation.

## How Inheritance Works

1. Define a base template with `{block}` placeholders
2. Child templates extend the base using `{extends}` or file naming
3. Override specific blocks in child templates

## Base Template Structure

### Creating base.tpl

```smarty
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{block name="title"}Default Title{/block}</title>
    {block name="head"}
    <link rel="stylesheet" href="{$baseurl}/css/styles.css">
    {/block}
</head>
<body>
    <header>
        {block name="header"}
            <nav>
                <a href="{$WEB_ROOT}/">Home</a>
                <a href="{$WEB_ROOT}/contact">Contact</a>
            </nav>
        {/block}
    </header>
    
    <main>
        {block name="content"}
            <!-- Page content goes here -->
        {/block}
    </main>
    
    <footer>
        {block name="footer"}
            <p>&copy; {year} Your Company</p>
        {/block}
    </footer>
    
    {block name="scripts"}
    <script src="{$baseurl}/js/scripts.js"></script>
    {/block}
</body>
</html>
```

## Extending Templates

### Using {extends} Tag

```smarty
{extends file="base.tpl"}

{block name="title"}
    Custom Page Title
{/block}

{block name="content"}
    <h1>Welcome to Our Site</h1>
    <p>This is custom page content.</p>
{/block}
```

### Combining Blocks

```smarty
{block name="title" append}
    - Additional Title Part
{/block}

{block name="scripts" prepend}
    <script src="custom-page.js"></script>
{/block}
```

## Block Modifiers

### append

Add content to end of parent's block.

```smarty
{block name="scripts" append}
    <script src="extra.js"></script>
{/block}
```

### prepend

Add content to beginning of parent's block.

```smarty
{block name="styles" prepend}
    <link rel="stylesheet" href="custom.css">
{/block}
```

### nocache

Prevent block from being cached.

```smarty
{block name="dynamic-content" nocache}
    {$session_content}
{/block}
```

## Multiple Levels of Inheritance

### Level 1: base.tpl

```smarty
<html>
<head>
    <title>{block name="title"}Site{/block}</title>
</head>
<body>
    {block name="content"}{/block}
</body>
</html>
```

### Level 2: two-column.tpl

```smarty
{extends file="base.tpl"}

{block name="title" prepend}
    {block name="page-title"}{/block} -
{/block}

{block name="content"}
    <div class="sidebar">
        {block name="sidebar"}
            Default sidebar content
        {/block}
    </div>
    <div class="main-content">
        {block name="main"}
            Main content here
        {/block}
    </div>
{/block}
```

### Level 3: page.tpl

```smarty
{extends file="two-column.tpl"}

{block name="page-title"}
    Product Page
{/block}

{block name="main"}
    <h1>Our Products</h1>
    <p>Product listing...</p>
{/block}
```

## WHMCS-Specific Patterns

### Client Area Layout

```smarty
{extends file="layout.tpl"}

{block name="pagetitle"}
    {$pagetitle}
{/block}

{block name="content"}
    <div class="client-area">
        {include file="$template/includes/sidebar.tpl"}
        
        <div class="main-content">
            {$content}
        </div>
    </div>
{/block}
```

### Product Detail Layout

```smarty
{extends file="two-column.tpl"}

{block name="title"}
    {$product->name} - WHMCS
{/block}

{block name="breadcrumb"}
    <nav class="breadcrumb">
        <a href="cart.php">Cart</a> >
        <a href="cart.php?a=view">{$product->name}</a>
    </nav>
{/block}

{block name="main"}
    <div class="product-detail">
        <h1>{$product->name}</h1>
        {$product->description}
    </div>
{/block}
```

## Dynamic Template Selection

### Based on Client Group

```php
<?php
// In PHP hook
$client = Menu::context('client');
if ($client && $client->groupid == 2) {
    $template = 'premium';
} else {
    $template = 'standard';
}
```

### Based on Page Type

```php
<?php
// Dynamically select template file
if ($templatefile === 'viewinvoice') {
    $templatefile = 'custom-invoice';
}
```

## Include vs Inheritance

### When to Use Include

- Reusable components (headers, footers, sidebars)
- Small, self-contained sections
- Content that doesn't need customization

```smarty
{include file="header.tpl"}
<div class="content">
    {$content}
</div>
{include file="footer.tpl"}
```

### When to Use Inheritance

- Page layouts with multiple regions
- Content that needs customization
- Consistent page structure across the site

```smarty
{extends file="two-column.tpl"}
{block name="main"}
    Custom content
{/block}
```

## Best Practices

1. **Keep base templates simple** - Define only essential blocks
2. **Use descriptive block names** - `content`, `sidebar`, `scripts`
3. **Avoid deep nesting** - 2-3 levels maximum
4. **Document block purposes** - Comment complex blocks
5. **Use consistent naming** - Follow a naming convention

## See Also

- [Template Blocks](../whmcs-template-blocks.md)
- [Template Include](../whmcs-template-include.md)
- [Template Extend](../whmcs-template-extend.md)