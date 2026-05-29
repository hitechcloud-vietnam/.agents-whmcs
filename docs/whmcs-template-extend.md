# WHMCS Template Extend

## Overview

The `{extends}` tag in WHMCS templates is used for template inheritance. It allows child templates to extend a parent template and override specific blocks.

## Basic Syntax

### Extending a Template

```smarty
{extends file="parent.tpl"}

{block name="content"}
    This overrides the parent's content block
{/block}
```

## How Inheritance Works

### 1. Create Parent Template

```smarty
{* base.tpl *}
<!DOCTYPE html>
<html>
<head>
    <title>{block name="title"}Site{/block}</title>
</head>
<body>
    {block name="header"}
        <header>Default Header</header>
    {/block}
    
    <main>
        {block name="content"}{/block}
    </main>
    
    {block name="footer"}
        <footer>&copy; 2024</footer>
    {/block}
</body>
</html>
```

### 2. Create Child Template

```smarty
{* page.tpl *}
{extends file="base.tpl"}

{block name="title"}
    Custom Page Title
{/block}

{block name="content"}
    <h1>Welcome</h1>
    <p>This is my custom content.</p>
{/block}
```

## Block Options

### append

Add to end of parent block content.

```smarty
{block name="scripts" append}
    <script src="custom.js"></script>
{/block}
```

### prepend

Add to beginning of parent block content.

```smarty
{block name="head" prepend}
    <meta name="custom" content="value">
{/block}
```

### hide

Hide block from output but keep for processing.

```smarty
{block name="debug" hide}
    <div class="debug">{$debug}</div>
{/block}
```

### nocache

Exclude from template caching.

```smarty
{block name="user-info" nocache}
    Logged in as: {$user->name}
{/block}
```

## Multiple Levels

### Level 1: base.tpl

```smarty
<html>
<head>
    {block name="head"}
        <meta charset="UTF-8">
    {/block}
</head>
<body>
    {block name="body"}{/block}
</body>
</html>
```

### Level 2: two-col.tpl

```smarty
{extends file="base.tpl"}

{block name="body"}
    <div class="container">
        <aside>{block name="sidebar"}Sidebar{/block}</aside>
        <main>{block name="main"}Main{/block}</main>
    </div>
{/block}
```

### Level 3: product.tpl

```smarty
{extends file="two-col.tpl"}

{block name="sidebar"}
    <nav class="product-nav">
        <a href="#overview">Overview</a>
        <a href="#features">Features</a>
        <a href="#pricing">Pricing</a>
    </nav>
{/block}

{block name="main"}
    <h1>{$product->name}</h1>
    <div class="product-info">...</div>
{/block}
```

## Dynamic Inheritance

### Variable Template

```smarty
{extends file=$layout_file}
```

### Conditional Extension

```smarty
{if $layout eq 'minimal'}
    {extends file="minimal.tpl"}
{elseif $layout eq 'full'}
    {extends file="full.tpl"}
{/if}
```

## Combining with Include

### Include Within Extends

```smarty
{extends file="base.tpl"}

{block name="navigation"}
    {include file="nav.tpl"}
{/block}

{block name="content"}
    <div class="content">
        {$content}
    </div>
{/block}
```

## WHMCS-Specific Patterns

### Client Area Layout

```smarty
{extends file="clientarealayout.tpl"}

{block name="pagetitle"}
    {$pagetitle} - {$companyname}
{/block}

{block name="clientsidenav"}
    {include file="$template/includes/clientmenu.tpl"}
{/block}

{block name="content"}
    <div class="client-content">
        {$content}
    </div>
{/block}
```

### Custom Page Template

```smarty
{extends file="base.tpl"}

{block name="title"}
    {$page_title} | {$companyname}
{/block}

{block name="breadcrumb"}
    <nav class="breadcrumb">
        <a href="{$WEB_ROOT}">Home</a> &raquo;
        {$page_title}
    </nav>
{/block}

{block name="content"}
    <article class="custom-page">
        {$page_content}
    </article>
{/block}

{block name="sidebar"}
    {include file="page-sidebar.tpl"}
{/block}
```

## Best Practices

### Do

- Keep inheritance shallow (2-3 levels max)
- Use descriptive block names
- Provide fallback content in parent blocks
- Document complex inheritance chains

### Don't

- Create deeply nested inheritance
- Override blocks unnecessarily
- Mix {extends} with {include} inappropriately

## See Also

- [Template Inheritance](../whmcs-template-inheritance.md)
- [Template Blocks](../whmcs-template-blocks.md)
- [Template Include](../whmcs-template-include.md)