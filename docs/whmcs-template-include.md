# WHMCS Template Include

## Overview

The `{include}` tag in WHMCS templates allows you to include and reuse template files. This promotes code reuse and maintains consistency across templates.

## Basic Syntax

### Simple Include

```smarty
{include file="header.tpl"}
```

### With Path Variable

```smarty
{include file="$template/includes/header.tpl"}
```

### With Variables

```smarty
{include file="button.tpl" type="submit" text="Click Me" class="btn-primary"}
```

## Include Paths

### Relative Paths

```smarty
{include file="includes/header.tpl"}
{include file="../shared/button.tpl"}
```

### Absolute Template Path

```smarty
{include file="$template/includes/header.tpl"}
```

### Full Path

```smarty
{include file="/path/to/templates/header.tpl"}
```

## Variables in Includes

### Passing Variables

```smarty
{include file="item.tpl" item=$product title="Product"}
```

### Multiple Variables

```smarty
{include file="card.tpl" 
    title=$item.name 
    description=$item.desc 
    price=$item.price
    image=$item.image
}
```

### Dynamic Variable Names

```smarty
{include file="{$template_file}.tpl"}
{include file="{$type}-form.tpl"}
```

## Variable Scoping

### Local Scope (default)

```smarty
{include file="item.tpl" item=$product}

{* item is available in item.tpl only *}
```

### Shared Scope

```smarty
{include file="item.tpl" item=$product scope=parent}

{* item is shared with parent template *}
```

### Assign Result

```smarty
{include file="complex.tpl" item=$data assign="result"}

{$result|upper}
```

## Common Use Cases

### Headers and Footers

```smarty
{include file="$template/includes/header.tpl"}

<div class="content">
    {$content}
</div>

{include file="$template/includes/footer.tpl"}
```

### Reusable Components

```smarty
{* Button component *}
{include file="components/button.tpl" type="submit" text="Submit"}

{* Alert component *}
{include file="components/alert.tpl" type="success" message="Success!"}

{* Card component *}
{include file="components/card.tpl" 
    title="Feature" 
    content="Description here"
}
```

### Sidebar

```smarty
{include file="$template/includes/sidebar.tpl"}

<div class="main-content">
    {block name="content"}
        Main content
    {/block}
</div>
```

## Include with Conditional Logic

### Conditional Include

```smarty
{if $show_sidebar}
    {include file="sidebar.tpl"}
{/if}
```

### Dynamic File Selection

```smarty
{if $theme eq "dark"}
    {include file="header-dark.tpl"}
{else}
    {include file="header-light.tpl"}
{/if}
```

### Variable File

```smarty
{include file="{$view}.tpl"}

{* If $view = "grid", includes grid.tpl *}
```

## Include in Loops

```smarty
{foreach $products as $product}
    {include file="product-card.tpl" product=$product}
{/foreach}

{* With index *}
{foreach $products as $index => $product}
    {include file="product-row.tpl" product=$product index=$index}
{/foreach}
```

## Nested Includes

### Multi-level Nesting

```smarty
{* main.tpl *}
{include file="layout.tpl"}

{* layout.tpl *}
{include file="header.tpl"}
{include file="content.tpl"}
{include file="footer.tpl"}

{* content.tpl *}
{include file="sidebar.tpl"}
<div class="content-area">
    {block name="content"}{/block}
</div>
```

### Shared Components

```smarty
{* card.tpl *}
<div class="card">
    <div class="card-header">
        {include file="title.tpl" title=$title}
    </div>
    <div class="card-body">
        {$content}
    </div>
</div>

{* title.tpl *}
<h3 class="card-title">{$title}</h3>
```

## Include with Captured Variables

```smarty
{capture name="custom_sidebar"}
    {include file="custom-widgets.tpl"}
{/capture}

{include file="layout-with-sidebar.tpl" sidebar=$smarty.capture.custom_sidebar}
```

## Performance Considerations

### Caching

```smarty
{include file="static-content.tpl" cached}
{include file="dynamic-content.tpl" nocache}
```

### Caching with Key

```smarty
{include file="user-specific.tpl" cache_lifetime=3600 cache_key=$user.id}
```

### Static vs Dynamic

```smarty
{* Static content - include once *}
{include file="footer.tpl"}

{* Dynamic content - consider caching *}
{include file="recent-items.tpl" nocache}
```

## WHMCS-Specific Patterns

### Standard Includes

```smarty
{include file="$template/includes/tabbedNavigation.tpl"}

{include file="$template/includes/confirm.tpl" 
    title="Confirm Action"
    message="Are you sure?"
    yes="Continue"
    no="Cancel"
}

{include file="$template/includes/panel.tpl"
    title="Panel Title"
}
```

### Client Area Includes

```smarty
{include file="$template/clientareadetails.tpl"}

{include file="$template/includes/service-sidebar.tpl" service=$service}
```

### Hook Integration

```smarty
{hook file="output_output.tpl" point="ClientAreaPage"}
```

## Best Practices

1. **Use relative paths** - Easier to move templates
2. **Pass necessary variables** - Don't rely on global scope
3. **Name include files clearly** - `header.tpl`, `footer.tpl`
4. **Keep includes small** - Single responsibility
5. **Consider caching** - Static content can be cached

## See Also

- [Smarty Syntax](../whmcs-smarty-syntax.md)
- [Template Inheritance](../whmcs-template-inheritance.md)
- [Template Extend](../whmcs-template-extend.md)