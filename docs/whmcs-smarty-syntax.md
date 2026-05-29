# WHMCS Smarty Template Syntax

## Overview

WHMCS uses Smarty as its templating engine. Smarty is a PHP template engine that separates PHP logic from presentation, allowing designers to work with templates without knowing PHP.

## Basic Syntax

### Variable Output

```smarty
{$variable_name}
{$product.name}
{$client.firstname}
{$invoice.total|number_format:2}
```

### Comments

```smarty
{* This is a comment - not rendered in output *}
```

### PHP-like Calculations

```smarty
{$a + $b}
{$a - $b}
{$a * $b}
{$a / $b}
{$a % $b}
```

## Arrays and Objects

### Accessing Array Elements

```smarty
{* Numeric index *}
{$array.0}
{$array.1}

{* Associative key *}
{$array.key}
{$customfields.Company Name}

{* Nested access *}
{$product.pricing.ly.monthly}
```

### Array Iteration

```smarty
{foreach $items as $item}
    <li>{$item.name}</li>
{/foreach}

{foreach $items as $key => $value}
    <dt>{$key}</dt>
    <dd>{$value}</dd>
{/foreach}
```

## Modifiers

### String Modifiers

```smarty
{* Uppercase *}
{$name|upper}

{* Lowercase *}
{$name|lower}

{* Capitalize *}
{$name|capitalize}

{* Truncate *}
{$description|truncate:100:"..."}

{* Replace *}
{$text|replace:'old':'new'}

{* Escape *}
{$user_input|escape}
{$html_content|noescape}
```

### Date Modifiers

```smarty
{* Format date *}
{$date|date_format:"%Y-%m-%d"}
{$date|date_format:"%B %d, %Y"}

{* Time ago *}
{$timestamp|time_ago}
```

### Number Modifiers

```smarty
{* Currency format *}
{$amount|currency_format}
{$amount|number_format:2:".":","}

{* Percentage *}
{$value|percentage}
```

## Conditional Logic

```smarty
{if $condition}
    HTML content
{elseif $other_condition}
    Other content
{else}
    Default content
{/if}
```

### Operators

```smarty
{* Comparison *}
{if $a eq $b}
{if $a neq $b}
{if $a gt $b}
{if $a lt $b}
{if $a gte $b}
{if $a lte $b}

{* Logical *}
{if $a && $b}
{if $a || $b}
{if !$a}

{* Containment *}
{if $value in $array}
```

## Loops

### For Loop

```smarty
{for $i=0 to 10}
    <li>Item {$i}</li>
{/for}

{for $i=1 to $total_pages}
    <a href="?page={$i}">{$i}</a>
{/for}
```

### While Loop

```smarty
{while $items}
    <div>{$items.pop()}</div>
{/while}
```

### Foreach Loop

```smarty
{foreach $products as $product}
    <div class="product">
        <h3>{$product.name}</h3>
        <p>{$product.description}</p>
        <span>{$product.price|currency_format}</span>
    </div>
{/foreach}
```

## Template Includes

### Including Templates

```smarty
{include file="template.tpl"}
{include file="header.tpl" title="Page Title"}
{include file="footer.tpl" company_name=$company}
```

### Capture

```smarty
{capture name="sidebar"}
    <aside>
        <h3>Sidebar</h3>
        <p>Sidebar content</p>
    </aside>
{/capture}

<div class="main">
    {$smarty.capture.sidebar}
</div>
```

## Built-in Functions

### Literal

```smarty
{literal}
    <script>
        function example() {
            // JavaScript code with {$variables} won't be parsed
        }
    </script>
{/literal}
```

### Strip

```smarty
{strip}
<div class="container">
    <p>Whitespace    will    be    removed</p>
</div>
{/strip}
```

## Whitespace Control

```smarty
{* Remove whitespace around *}
{assign var="x" value="y"}{*$no whitespace*}}

{* Whitespace trimming *}
{foreach $items as $item key=key item=item}
    <li>{$item}</li>
{/foreach}
```

## See Also

- [Template Variables](../whmcs-template-variables.md)
- [Template Functions](../whmcs-template-functions.md)
- [Smarty Documentation](https://www.smarty.net/docs/en/)