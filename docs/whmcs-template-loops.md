# WHMCS Template Loops

## Overview

Loops in WHMCS templates allow you to iterate over arrays and display repeated content. Smarty provides several loop constructs for different use cases.

## foreach Loop

The most common loop construct in WHMCS templates.

### Basic Syntax

```smarty
{foreach $items as $item}
    <div>{$item}</div>
{/foreach}
```

### Key-Value Pairs

```smarty
{foreach $array as $key => $value}
    <dt>{$key}</dt>
    <dd>{$value}</dd>
{/foreach}
```

### With Array Access

```smarty
{foreach $products as $product}
    <div class="product">
        <h3>{$product->name}</h3>
        <p>{$product->description}</p>
        <span class="price">{$product->price|currency_format}</span>
    </div>
{/foreach}
```

### foreachelse

Fallback when array is empty.

```smarty
{foreach $products as $product}
    <div>{$product->name}</div>
{foreachelse}
    <p>No products available</p>
{/foreach}
```

## foreach Properties

Access loop metadata using `$variable@property`.

### @index

Current iteration index (starts at 0).

```smarty
{foreach $items as $item}
    Item {$item@index + 1}: {$item}
{/foreach}
```

### @iteration

Current iteration count (starts at 1).

```smarty
{foreach $items as $item}
    {$item@iteration} of {$items|count}
{/foreach}
```

### @first

True for first iteration.

```smarty
{foreach $items as $item}
    {if $item@first}
        <div class="first-item">{$item}</div>
    {else}
        <div>{$item}</div>
    {/if}
{/foreach}
```

### @last

True for last iteration.

```smarty
{foreach $items as $item}
    <div{if $item@last} class="last"{/if}>{$item}</div>
{/foreach}
```

### @show

Check if loop has items.

```smarty
{if $items@show}
    <ul>
        {foreach $items as $item}
            <li>{$item}</li>
        {/foreach}
    </ul>
{/if}
```

### @total

Total number of iterations.

```smarty
{foreach $items as $item}
    Progress: {$item@iteration}/{$item@total}
{/foreach}
```

## Nested foreach

```smarty
{foreach $categories as $category}
    <h2>{$category.name}</h2>
    <ul>
        {foreach $category.products as $product}
            <li>{$product.name}</li>
        {/foreach}
    </ul>
{/foreach}
```

## for Loop

Numeric iteration.

### Basic for

```smarty
{for $i=0; $i < 10; $i++}
    Item {$i}
{/for}
```

### For with step

```smarty
{for $i=0; $i <= 100; $i+=10}
    {$i}
{/for}
```

### For with variable bounds

```smarty
{for $page=1; $page <= $total_pages; $page++}
    <a href="?page={$page}"{if $page == $current_page} class="active"{/if}>
        {$page}
    </a>
{/for}
```

### Paginated display

```smarty
{for $i=1 to $total_items}
    {if $i is multiple of 5}
        <div class="page-break"></div>
    {/if}
    Item {$i}
{/for}
```

## section Loop

Alternative loop syntax for indexed arrays.

```smarty
{section name=i loop=$products}
    <div class="product">
        {$products[i]->name}
        {$products[i]->price|currency_format}
    </div>
{/section}
```

### section Properties

```smarty
{$smarty.section.i.index}     {* Current index *}
{$smarty.section.i.iteration}  {* Current iteration *}
{$smarty.section.i.first}     {* First iteration? *}
{$smarty.section.i.last}     {* Last iteration? *}
{$smarty.section.i.total}     {* Total iterations *}
{$smarty.section.i.show}      {* Has items? *}
```

## while Loop

Loop until condition is false.

```smarty
{while $items}
    <div>{$items|shift}</div>
{/while}
```

## Break and Continue

### Breaking Out of Loops

```smarty
{foreach $items as $item}
    {if $item@index >= 5}
        {break}
    {/if}
    <div>{$item}</div>
{/foreach}
```

### Skip Iteration

```smarty
{foreach $items as $item}
    {if $item@index is even}
        {continue}
    {/if}
    <div class="odd">{$item}</div>
{/foreach}
```

## Real-World Examples

### Service Table

```smarty
<table class="services-table">
    <thead>
        <tr>
            <th>#</th>
            <th>Product</th>
            <th>Domain</th>
            <th>Status</th>
            <th>Next Due</th>
        </tr>
    </thead>
    <tbody>
        {foreach $services as $service}
            <tr class="row-{$service@iteration}">
                <td>{$service@iteration}</td>
                <td>{$service->product->name}</td>
                <td>{$service->domain}</td>
                <td>
                    <span class="status-{$service->status|lower}">
                        {$service->status}
                    </span>
                </td>
                <td>{$service->nextduedate|date_format}</td>
            </tr>
        {/foreach}
    </tbody>
</table>
```

### Breadcrumb

```smarty
<nav class="breadcrumb">
    {foreach $breadcrumbs as $crumb key=$index}
        <a href="{$crumb.url}">{$crumb.label}</a>
        {if !$crumb@last}
            <span class="separator">/</span>
        {/if}
    {/foreach}
</nav>
```

### Column Layout

```smarty
<div class="row">
    {foreach $products as $product name=products}
        <div class="col-md-4">
            <div class="product-card">
                {$product->name}
            </div>
        </div>
        {if $smarty.foreach.products.index % 3 == 2 && !$smarty.foreach.products.last}
    </div>
    <div class="row">
        {/if}
    {/foreach}
</div>
```

### Accordion

```smarty
<div class="accordion">
    {foreach $sections as $section name=sections}
        <div class="accordion-item">
            <button class="accordion-header" 
                    data-target="section-{$section@index}">
                {$section.title}
            </button>
            <div id="section-{$section@index}" 
                 class="accordion-content{if $section@first} show{/if}">
                {$section.content}
            </div>
        </div>
    {/foreach}
</div>
```

### Pagination

```smarty
{assign var="start" value=max(1, $current_page - 2)}
{assign var="end" value=min($total_pages, $current_page + 2)}

<div class="pagination">
    {if $current_page > 1}
        <a href="?page={$current_page - 1}">&laquo;</a>
    {/if}
    
    {for $i=$start to $end}
        <a href="?page={$i}"{if $i == $current_page} class="active"{/if}>
            {$i}
        </a>
    {/for}
    
    {if $current_page < $total_pages}
        <a href="?page={$current_page + 1}">&raquo;</a>
    {/if}
</div>
```

## See Also

- [Smarty Syntax](../whmcs-smarty-syntax.md)
- [Template Conditionals](../whmcs-template-conditionals.md)
- [Template Inheritance](../whmcs-template-inheritance.md)