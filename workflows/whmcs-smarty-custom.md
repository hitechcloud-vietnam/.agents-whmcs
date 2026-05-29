# WHMCS Smarty Customization Workflow

## Purpose
Customize WHMCS using Smarty template modifications safely.

## Prerequisites
- WHMCS installation
- Basic Smarty/PHP knowledge
- Template file access

## Step-by-Step Process

### Step 1: Understanding WHMCS Smarty Variables

**Common System Variables:**
```smarty
{$baseweburl}         {* Base URL *}
{$WEB_ROOT}           {* Web root *}
{$template}           {* Current template name *}
{$client}              {* Logged in client data *}
{$loggedin}            {* Login status *}
{$cartitems}          {* Cart items array *}
{$languages}          {* Available languages *}
```

### Step 2: Create Custom Smarty Plugin

**Location:** `/whmcs/includes/smartyplugins/`

**Function Example (modifier):**
```php
<?php
/**
 * Custom Smarty modifier: format_currency
 * Usage: {$amount|format_currency:'USD'}
 */
function smarty_modifier_format_currency($string, $currency = 'USD') {
    $formatter = new NumberFormatter($currency, NumberFormatter::CURRENCY);
    return $formatter->formatCurrency($string, $currency);
}
```

**Block Function Example:**
```php
<?php
/**
 * Custom Smarty block: highlight
 * Usage: {highlight}text to highlight{/highlight}
 */
function smarty_block_highlight($params, $content, &$smarty) {
    $color = isset($params['color']) ? $params['color'] : '#ffff00';
    return '<span style="background-color: ' . $color . ';">' . $content . '</span>';
}
```

### Step 3: Modify Template Files Safely

**Best Practice: Template Overrides**
```
1. Copy original template to custom folder
2. Make modifications in custom folder
3. Set custom folder as active template
```

### Step 4: Common Template Modifications

**Display Logic:**
```smarty
{* Check if user is logged in *}
{if $loggedin}
    <p>Welcome back, {$client.firstname}!</p>
{else}
    <p>Please <a href="login.php">log in</a></p>
{/if}

{* Loop through products *}
{foreach from=$products item=product}
    <div class="product">
        <h3>{$product.name}</h3>
        <p>{$product.description}</p>
        <span class="price">{$product.price}</span>
    </div>
{/foreach}
```

**Conditional Classes:**
```smarty
<div class="container {if $loggedin}user-logged-in{else}user-guest{/if}">
    {* Content *}
</div>

{* Active navigation *}
<nav>
    <a href="/" {if $current_page == 'home'}class="active"{/if}>Home</a>
    <a href="/about" {if $current_page == 'about'}class="active"{/if}>About</a>
</nav>
```

**Date Formatting:**
```smarty
{* Using PHP date format *}
{$date_created|date_format:"%Y-%m-%d"}
{$last_login|date_format:"%B %d, %Y %H:%M"}

{* Custom formatting *}
{assign var="formatted_date" value=$date_start|date_format:"%d/%m/%Y"}
```

### Step 5: Create Custom Template Variables

**Hook to add variables:**
```php
<?php
/**
 * Add custom template variables
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    return [
        'custom_greeting' => getCustomGreeting(),
        'featured_products' => getFeaturedProducts(),
        'site_notifications' => getSiteNotifications()
    ];
});
```

**Use in Template:**
```smarty
{if $custom_greeting}
    <div class="alert">{$custom_greeting}</div>
{/if}

{foreach from=$featured_products item=product}
    <div class="featured">
        <img src="{$product.image}" alt="{$product.name}">
        <h4>{$product.name}</h4>
    </div>
{/foreach}
```

### Step 6: Smarty Plugins Directory

**Custom Plugin Types:**

1. **Modifiers** (single value transformation)
2. **Blocks** (content with start/end tags)
3. **Functions** (dynamic content generation)
4. **Prefilters** (modify template before compile)
5. **Postfilters** (modify template after compile)
6. **Outputfilters** (modify output)

**Registering Custom Plugins:**
```php
<?php
// In /whmcs/includes/smarty_setup.php or hook file
use WHMCS\Smarty\Extension;

add_hook('SmartyAppSetup', 1, function($smarty) {
    $smarty->registerPlugin('modifier', 'my_custom_modifier', 'smarty_modifier_my_custom_modifier');
});
```

### Step 7: Debug Smarty

**Enable Debug Mode:**
```smarty
{debug}
{* Opens Smarty debug window *}
```

**Display Variables:**
```smarty
<pre>{$smarty.dump}</pre>
{* Dumps all available variables *}

<pre>{print_r($variable, true)|var_dump}</pre>
{* Debug specific variable *}
```

### Step 8: Template Security

**Always Escape Output:**
```smarty
{* XSS Prevention *}
{$user_input|escape}
{$user_input|htmlspecialchars}
{$user_input|htmlentities}

{* URL Encoding *}
{$string|urlencode}

{* JavaScript Escaping *}
{$string|jsescape}
```

## Common Smarty Snippets

**Pagination:**
```smarty
{assign var="totalPages" value=($totalItems / $itemsPerPage)|ceil}
{assign var="currentPage" value=$smarty.get.page|default:1}

<div class="pagination">
    {if $currentPage > 1}
        <a href="?page={$currentPage - 1}">Previous</a>
    {/if}
    
    {for $i=1 to $totalPages}
        <a href="?page={$i}" {if $i == $currentPage}class="active"{/if}>{$i}</a>
    {/for}
    
    {if $currentPage < $totalPages}
        <a href="?page={$currentPage + 1}">Next</a>
    {/if}
</div>
```

**String Manipulation:**
```smarty
{* Truncate text *}
{$description|truncate:100:'...':true}

{* String replace *}
{$text|replace:'old':'new'}

{* Uppercase/Lowercase *}
{$text|upper}
{$text|lower}

{* String length *}
{$text|strlen}
```

## Best Practices
- Never modify core WHMCS files
- Use child templates for customizations
- Always escape user-generated content
- Clear template cache after changes
- Test in multiple browsers
- Document all custom modifications
- Keep Smarty code clean and readable
