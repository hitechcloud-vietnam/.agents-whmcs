# WHMCS Variable Modifiers

## Overview

Variable modifiers in WHMCS/Smarty are used to transform and format variable output. They are applied using the pipe `|` character after a variable.

## Syntax

```smarty
{$variable|modifier}
{$variable|modifier:param}
{$variable|modifier1|modifier2}
{$var|modifier1:param1:param2|modifier2}
```

## String Modifiers

### upper

Convert to uppercase.

```smarty
{$name|upper}  {* JOHN DOE *}
```

### lower

Convert to lowercase.

```smarty
{$name|lower}  {* john doe *}
```

### capitalize

Capitalize first letter of each word.

```smarty
{$name|capitalize}  {* John Doe *}
```

### nl2br

Convert newlines to HTML breaks.

```smarty
{$text|nl2br}
```

### replace

Find and replace in string.

```smarty
{$text|replace:'search':'replace'}
{$description|replace:'old text':'new text'}
```

### truncate

Truncate string to specified length.

```smarty
{$text|truncate:100}
{$text|truncate:100:"..."}
{$text|truncate:100:"...":true}  {* word break *}
{$text|truncate:30:"...":true:true}  {* middle *}
```

### substr

Get substring.

```smarty
{$string|substr:0:10}  {* first 10 chars *}
{$string|substr:5}     {* from char 5 *}
```

### strstr

Find string after needle.

```smarty
{$email|strstr:'@'}  {* @domain.com *}
```

### trim

Remove whitespace from ends.

```smarty
{$text|trim}
```

### esc / escape

Escape for HTML output.

```smarty
{$user_input|escape}
{$html|noescape}  {* Don't escape *}
```

## Date/Time Modifiers

### date_format

Format date/time.

```smarty
{$timestamp|date_format}
{$timestamp|date_format:"%Y-%m-%d"}
{$timestamp|date_format:"%B %d, %Y"}
{$timestamp|date_format:"%d/%m/%Y %H:%M"}
```

Format specifiers:
- `%Y` - 4-digit year
- `%y` - 2-digit year
- `%m` - Month (01-12)
- `%B` - Full month name
- `%b` - Short month name
- `%d` - Day (01-31)
- `%A` - Full weekday
- `%a` - Short weekday
- `%H` - Hour (24-hour)
- `%I` - Hour (12-hour)
- `%M` - Minutes
- `%S` - Seconds

### time_ago

Convert to relative time.

```smarty
{$timestamp|time_ago}  {* 2 hours ago, 3 days ago, etc. *}
```

## Number Modifiers

### number_format

Format number with decimals and separators.

```smarty
{$amount|number_format}
{$amount|number_format:2}
{$amount|number_format:2:",":"."}  {* German format *}
```

### currency_format

Format as currency.

```smarty
{$amount|currency_format}
{$amount|currency_format:$currencyid}
```

### percentage

Format as percentage.

```smarty
{$value|percentage}      {* 25% *}
{$value|percentage:2}    {* 25.00% *}
```

### quantize

Round to specified precision.

```smarty
{$value|quantize:0.01}
{$value|quantize:5}
```

## Array Modifiers

### count

Count array elements.

```smarty
{$items|count}
```

### json_encode

Convert array to JSON.

```smarty
{$array|json_encode}
```

### json_decode

Convert JSON to array (in PHP only).

### array_search

Search in array.

### in_array

Check if value exists.

```smarty
{if $value|in_array:$allowed_values}
    Allowed
{/if}
```

### array_key_exists

Check if key exists.

```smarty
{if $key|array_key_exists:$array}
    Key exists
{/if}
```

## Conditional Modifiers

### default

Provide default value.

```smarty
{$name|default:"Valued Customer"}
{$company|default:""}
```

### ternary

Ternary operation.

```smarty
{$active|ternary:'Active':'Inactive'}
{$value|ternary:'Yes':'No':'N/A'}
```

### coalesce

Return first non-empty value.

```smarty
{$var1|coalesce:$var2:$var3:"default"}
```

## Type Modifiers

### intval / floatval

Convert to integer/float.

```smarty
{$price|intval}
{$amount|floatval}
```

### string

Convert to string.

```smarty
{$value|string}
```

### bool

Convert to boolean.

```smarty
{$value|bool}
```

## URL Modifiers

### urlencode

URL encode.

```smarty
{$text|urlencode}
```

### rawurlencode

RFC 3986 URL encode.

```smarty
{$text|rawurlencode}
```

### explode

Split string to array.

```smarty
{$tags|explode:','}
```

### join

Join array to string.

```smarty
{$items|join:','}
{$array|join:' - '}
```

## WHMCS-Specific Modifiers

### sha1

Generate SHA1 hash.

```smarty
{$value|sha1}
```

### md5

Generate MD5 hash.

```smarty
{$value|md5}
```

### base64_encode

Base64 encode.

```smarty
{$data|base64_encode}
```

### base64_decode

Base64 decode.

```smarty
{$data|base64_decode}
```

## Chaining Modifiers

```smarty
{* Multiple modifiers *}
{$text|lower|replace:' ':'-'|truncate:50}

{* With parameters *}
{$price|currency_format|upper}

{* Complex example *}
{$name|trim|capitalize|default:"Unknown"|truncate:30:"..."}
```

## Custom Modifiers

Register in PHP:

```php
<?php
$smarty->registerPlugin('modifier', 'myModifier', function($value, $param1 = null) {
    return $value . $param1;
});
```

Use in template:

```smarty
{$text|myModifier:'suffix'}
```

## See Also

- [Smarty Syntax](../whmcs-smarty-syntax.md)
- [Template Conditionals](../whmcs-template-conditionals.md)