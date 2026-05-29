# WHMCS Template Functions

## Overview

Smarty functions in WHMCS templates are used to perform operations, retrieve data, and control template output. Functions are wrapped in `{funcname}` or `{funcname ...}` syntax.

## Output Functions

### lang

Language string output.

```smarty
{lang key="welcome"}
{lang key="invoice.total" template="invoices"}
{lang key="global" print="yes"}
```

### hook

Execute a hook point.

```smarty
{hook file="output_output.tpl" point="ClientAreaPage"}
{hook point="ClientAreaPageViewInvoice" invoice=$invoice}
```

### include

Include another template file.

```smarty
{include file="$template/includes/header.tpl"}
{include file="table.tpl" loop=$items}
{include file="javascript.tpl" inline}
```

## Variable Functions

### assign

Assign a variable.

```smarty
{assign var="name" value="John"}
{assign var="total" value=$subtotal + $tax}
{assign var="message" value="Hello {$name}"}
```

### capture

Capture output into a variable.

```smarty
{capture name="sidebar_content"}
    <aside>
        <h3>Sidebar</h3>
        <ul>
            <li>Item 1</li>
            <li>Item 2</li>
        </ul>
    </aside>
{/capture}

<div class="content">
    {$smarty.capture.sidebar_content}
</div>
```

### foreach / foreachelse

Iterate over arrays.

```smarty
{foreach $products as $product}
    <div class="product">
        {$product.name}
    </div>
{/foreach}

{foreach $items as $key => $value}
    <dt>{$key}</dt>
    <dd>{$value}</dd>
{foreachelse}
    <p>No items found</p>
{/foreach}
```

### for

For loop iteration.

```smarty
{for $i=0; $i < 10; $i++}
    Item {$i}
{/for}

{for $page=1 to $total_pages}
    <a href="?page={$page}">{$page}</a>
{/for}
```

### section / sectionelse

Named iteration (alternative to foreach).

```smarty
{section name=i loop=$products}
    {$products[i].name}
{sectionelse}
    No products
{/section}
```

### while

While loop.

```smarty
{while $items}
    {$items|shift}
{/while}
```

## Conditional Functions

### if / elseif / else

Conditional logic.

```smarty
{if $condition}
    Content if true
{elseif $other}
    Content if other is true
{else}
    Default content
{/if}
```

### ldelim / rdelim

Output literal delimiters.

```smarty
{ldelim}variable{rdelim}  {* Outputs: {variable} *}
```

## HTML Generation Functions

### form

Generate form tags.

```smarty
{form method="post" action="clientarea.php"}
{form method="post" id="myform" class="custom-form"}
```

### input

Generate input fields.

```smarty
{input type="text" name="field" value=$value class="form-control"}
{input type="hidden" name="token" value=$token}
```

### textarea

Generate textarea.

```smarty
{textarea name="message" rows="5" cols="50"}
    {$default_text}
{/textarea}
```

### select

Generate select dropdown.

```smarty
{select name="country"}
    {foreach $countries as $code => $name}
        <option value="{$code}"{if $code == $selected} selected{/if}>
            {$name}
        </option>
    {/foreach}
{/select}
```

### button

Generate buttons.

```smarty
{button type="submit" class="btn-primary" id="submit"}
    Submit
{/button}

{button type="reset" class="btn-secondary"}
    Reset
{/button}
```

### label

Generate labels.

```smarty
{label for="email" required="true"}
    Email Address
{/label}
```

## Display Functions

### dump

Debug variable output.

```smarty
{dump var=$variable}
{dump var=$array}
```

### helpicon

Display help icon.

```smarty
{helpicon text="This is helpful text"}
```

### icon

Display Font Awesome icon.

```smarty
{icon type="arrow-right"}
{icon type="check" style="fas"}
{icon type="warning" class="text-warning"}
```

### alert

Display alert box.

```smarty
{alert type="info"}
    Informational message
{/alert}

{alert type="success" dismiss=true}
    Success message
{/alert}

{alert type="warning"}
    Warning message
{/alert}

{alert type="danger"}
    Error message
{/alert}
```

## Pagination Functions

### pager

Generate pagination.

```smarty
{pager
    recordcount=$total
    limit=$limit
    page=$current_page
    maxlinks=5
    textnumlinks=3
    step=1
}
```

## Table Functions

### foreachproduct

Iterate over products with special handling.

```smarty
{foreachproduct product=$product}
    <div class="product">
        {$product->name}
    </div>
{/foreachproduct}
```

## API Functions

### call

Call a WHMCS API function.

```smarty
{call function="GetSupportDepartments"}
{call function="GetConfiguration"}
{call function="GetClientsDetails" params=$params}
```

## Utility Functions

### math

Perform calculations.

```smarty
{math equation="x + y" x=$a y=$b}
{math equation="price * qty" price=$item.price qty=$item.qty}
{math equation="(a + b) * 0.1" a=$x b=$y}
```

### counter

Output a counter.

```smarty
{counter start=1 skip=1 assign="cnt"}
{$cnt}
{counter}
```

### cycle

Cycle through values.

```smarty
{foreach $items as $item}
    <tr class="{cycle values="row1,row2"}">
        {$item}
    </tr>
{/foreach}
```

### now

Output current date/time.

```smarty
{now format="Y-m-d H:i:s"}
{now}
```

## See Also

- [Smarty Syntax](../whmcs-smarty-syntax.md)
- [Template Variables](../whmcs-template-variables.md)