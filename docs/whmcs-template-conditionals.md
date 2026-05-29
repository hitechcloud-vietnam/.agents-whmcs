# WHMCS Template Conditionals

## Overview

Conditional statements in WHMCS templates control what content is displayed based on data values. Smarty uses `{if}`, `{elseif}`, `{else}`, and `{/if}` tags.

## Basic Syntax

### Simple If

```smarty
{if $condition}
    Content to display
{/if}
```

### If-Else

```smarty
{if $condition}
    True content
{else}
    False content
{/if}
```

### If-Elseif-Else

```smarty
{if $condition1}
    Condition 1 content
{elseif $condition2}
    Condition 2 content
{elseif $condition3}
    Condition 3 content
{else}
    Default content
{/if}
```

## Comparison Operators

### Equality

```smarty
{if $status eq "active"}
    Active
{/if}

{if $status == "active"}
    Active
{/if}
```

### Inequality

```smarty
{if $status neq "cancelled"}
    Not cancelled
{/if}

{if $status != "cancelled"}
    Not cancelled
{/if}
```

### Greater Than/Less Than

```smarty
{if $count gt 0}
    Has items
{/if}

{if $count lt 10}
    Below threshold
{/if}

{if $count gte 5}
    At least 5
{/if}

{if $count lte 100}
    At most 100
{/if}
```

## Logical Operators

### AND

```smarty
{if $active && $verified}
    Active and verified
{/if}

{if $status eq "active" AND $balance gt 0}
    Active with balance
{/if}
```

### OR

```smarty
{if $type eq "hosting" || $type eq "server"}
    Hosting or server
{/if}

{if $status eq "pending" OR $status eq "active"}
    Pending or active
{/if}
```

### NOT

```smarty
{if !$disabled}
    Not disabled
{/if}

{if not $isGuest}
    Registered user
{/if}
```

### Combined

```smarty
{if ($a && $b) || $c}
    Complex logic
{/if}

{if $active && (!$disabled || $admin)}
    Active and (not disabled or admin)
{/if}
```

## isset and empty

### isset

Check if variable exists.

```smarty
{if isset($variable)}
    Variable exists
{/if}

{if isset($array.key)}
    Key exists in array
{/if}
```

### empty

Check if variable is empty.

```smarty
{if empty($name)}
    No name provided
{/if}

{if not empty($email)}
    Email: {$email}
{/if}
```

### === null

```smarty
{if $value === null}
    Value is null
{/if}
```

## String Comparisons

### Contains

```smarty
{if $text|strstr:'keyword'}
    Contains keyword
{/if}

{if $domain|strpos:'.co.uk' !== false}
    UK domain
{/if}
```

### Starts/Ends With

```smarty
{if $email|substr:0:4 eq "www."}
    Starts with www.
{/if}

{if $domain|substr:-4 eq ".com"}
    .com domain
{/if}
```

### Regex Match

```smarty
{if $email|preg_match:'/^[a-z]+@[a-z]+\.[a-z]+$/'}
    Valid email format
{/if}
```

## Array Conditionals

### In Array

```smarty
{if $value|in_array:$allowed_values}
    Allowed
{/if}

{if $status|in_array:["active","pending","approved"]}
    Valid status
{/if}
```

### Array Has Items

```smarty
{if $items|count > 0}
    Has items
{/if}

{if $items}
    Has items
{/if}
```

### First/Last Item

```smarty
{foreach $items as $item}
    {if $item@first}
        First item
    {/if}
    {if $item@last}
        Last item
    {/if}
{/foreach}
```

## Object Property Checks

### Property Exists

```smarty
{if $client->companyname}
    {$client->companyname}
{/if}

{if isset($client->credit)}
    Credit: {$client->credit}
{/if}
```

### Property Value

```smarty
{if $service->status eq "Active"}
    <span class="badge-success">Active</span>
{/if}

{if $invoice->balance > 0}
    Balance due: {$invoice->balance}
{/if}
```

## Negation

```smarty
{if $condition == false}
{if not $condition}
{if $condition|not}
```

## Inline Conditionals

### Ternary Operator

```smarty
{$status eq "active" ? "Active" : "Inactive"}
{assign var="label" value=$active ? "Enabled" : "Disabled"}
```

### Multiple Values

```smarty
{$status}
{if $status eq "Active"}active{elseif $status eq "Suspended"}warning{else}default{/if}
```

## Complex Examples

### Product Status Display

```smarty
<div class="status status-{$service->status|lower}">
    {if $service->status eq "Active"}
        <span class="icon check"></span> Active
    {elseif $service->status eq "Suspended"}
        <span class="icon pause"></span> Suspended
    {elseif $service->status eq "Terminated"}
        <span class="icon x"></span> Terminated
    {else}
        <span class="icon clock"></span> Pending
    {/if}
</div>
```

### Invoice Status

```smarty
<div class="invoice-status">
    {if $invoice->status eq "Paid"}
        <span class="label label-success">Paid</span>
    {elseif $invoice->status eq "Unpaid"}
        {if $invoice->duedate|substr:0:10 lt $smarty.now|date_format:"Y-m-d"}
            <span class="label label-danger">Overdue</span>
        {else}
            <span class="label label-warning">Unpaid</span>
        {/if}
    {elseif $invoice->status eq "Cancelled"}
        <span class="label label-default">Cancelled</span>
    {/if}
</div>
```

### Table Row Alternation

```smarty
<table>
    {foreach $items as $item}
        <tr class="{if $item@iteration is even}even{else}odd{/if}">
            <td>{$item.name}</td>
            <td>{$item.value}</td>
        </tr>
    {/foreach}
</table>
```

### Nested Conditions

```smarty
{if $client}
    <div class="client-info">
        {if $client->companyname}
            <h2>{$client->companyname}</h2>
            <p>Contact: {$client->firstname} {$client->lastname}</p>
        {else}
            <h2>{$client->firstname} {$client->lastname}</h2>
        {/if}
        
        {if $client->credit > 0}
            <div class="credit-balance">
                Credit: {$client->credit|currency_format}
            </div>
        {/if}
    </div>
{/if}
```

## See Also

- [Smarty Syntax](../whmcs-smarty-syntax.md)
- [Template Loops](../whmcs-template-loops.md)