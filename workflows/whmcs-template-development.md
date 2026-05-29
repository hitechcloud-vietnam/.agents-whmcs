# WHMCS Template Development Workflow

## Purpose
Create custom WHMCS templates for client and admin area

## Prerequisites
- WHMCS installed
- HTML/CSS knowledge
- Smarty template engine knowledge

## Step 1: Understand Template Structure

```
/whmcs/templates/
  /six/                 # Default template
    /account/
    /cart/
    /client/
    /home/
    /support/
    /assets/
  /your_template/
    /account/
    /cart/
    /client/
    /home/
    /support/
    /assets/
```

## Step 2: Create Template Directory

```bash
cd /var/www/whmcs/templates
cp -r six my_custom_template
```

## Step Me From: Template Configuration

Create template.php:

```php
<?php
// template.php

function myCustomTemplate_meta()
{
    return [
        'name' => 'My Custom Template',
        'author' => 'Your Name',
        'version' => '1.0.0',
        'requires' => '8.0',
        'supports' => '8.0+',
    ];
}

function myCustomTemplate_output($vars)
{
    // Custom output processing
}
```

## Step 4: Create Template Variables

### Register Variables in template.php

```php
function myCustomTemplate_output($vars)
{
    $assign = [];
    $assign['custom_variable'] = 'Custom Value';
    $assign['custom_data'] = getCustomData();
    
    return $assign;
}
```

## Step 5: Modify Header Template

Edit `/templates/my_custom_template/header.tpl`:

```smarty
<!DOCTYPE html>
<html lang="{$language}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{$companyname} - {$pagetitle}</title>
    
    {* Custom CSS *}
    <link href="{$base_url}templates/{$template}/assets/css/style.css" rel="stylesheet">
    
    {* Hook for additional head content *}
    {hook file="header"}
</head>
<body class="{$template}">
    <header class="site-header">
        <div class="container">
            <a href="{$WEB_ROOT}/" class="logo">
                <img src="{$logo}" alt="{$companyname}">
            </a>
            <nav class="main-nav">
                {$navigation}
            </nav>
        </div>
    </header>
    <main class="site-content">
```

## Step 6: Modify Footer Template

Edit `footer.tpl`:

```smarty
    </main>
    <footer class="site-footer">
        <div class="container">
            <div class="footer-content">
                <div class="footer-column">
                    <h4>{$companyname}</h4>
                    <p>{$company_address}</p>
                    <p>{$company_email}</p>
                </div>
                <div class="footer-column">
                    <h4>Quick Links</h4>
                    <ul>
                        <li><a href="{$WEB_ROOT}/supporttickets.php">Support</a></li>
                        <li><a href="{$WEB_ROOT}/knowledgebase.php">Knowledge Base</a></li>
                        <li><a href="{$WEB_ROOT}/affiliates.php">Affiliates</a></li>
                    </ul>
                </div>
            </div>
            <div class="footer-bottom">
                <p>&copy; {date('Y')} {$companyname}. All rights reserved.</p>
            </div>
        </div>
    </footer>
    
    {* Custom JS *}
    <script src="{$base_url}templates/{$template}/assets/js/main.js"></script>
    {hook file="footer"}
</body>
</html>
```

## Step 7: Create Homepage Template

Edit `home.tpl`:

```smarty
{extends file="layout.tpl"}

{block name="content"}
<div class="home-page">
    {* Hero Section *}
    <section class="hero">
        <h1>{$LANG.homepage_tagline}</h1>
        <p>{$LANG.homepage_description}</p>
        <a href="{if $loggedin}{$WEB_ROOT}/cart.php{else}{$WEB_ROOT}/register.php{/if}" class="btn btn-primary">
            {if $loggedin}Order Now{else}Get Started{/if}
        </a>
    </section>
    
    {* Features Section *}
    <section class="features">
        {foreach $featureProducts as $product}
        <div class="feature-box">
            <h3>{$product.name}</h3>
            <p>{$product.description}</p>
            <span class="price">{$product.price}</span>
        </div>
        {/foreach}
    </section>
</div>
{/block}
```

## Step 8: Create Cart Template

Edit `cart.tpl`:

```smarty
<div class="shopping-cart">
    <h2>{$LANG.cart}</h2>
    
    {if $cartitems}
    <table class="cart-table">
        <thead>
            <tr>
                <th>Product</th>
                <th>Configuration</th>
                <th>Billing Cycle</th>
                <th>Amount</th>
                <th>Remove</th>
            </tr>
        </thead>
        <tbody>
            {foreach $cartitems as $item}
            <tr>
                <td>{$item.name}</td>
                <td>{$item.configoption}</td>
                <td>{$item.billingcycle}</td>
                <td>{$item.amount}</td>
                <td>
                    <a href="{$WEB_ROOT}/cart.php?a=remove&id={$item.id}" class="btn btn-sm btn-danger">
                        Remove
                    </a>
                </td>
            </tr>
            {/foreach}
        </tbody>
    </table>
    
    <div class="cart-summary">
        <h3>Total: {$carttotal}</h3>
        <a href="{$WEB_ROOT}/cart.php?a=checkout" class="btn btn-primary">
            Checkout
        </a>
    </div>
    {else}
    <p>Your cart is empty.</p>
    {/if}
</div>
```

## Step 9: Create Custom CSS

```css
/* assets/css/style.css */

:root {
    --primary-color: #0073aa;
    --secondary-color: #23282d;
    --accent-color: #00a0d2;
    --font-family: 'Inter', sans-serif;
}

body {
    font-family: var(--font-family);
    color: var(--secondary-color);
}

.site-header {
    background: var(--primary-color);
    padding: 1rem 0;
}

.site-header .logo img {
    height: 50px;
}

.hero {
    background: linear-gradient(135deg, var(--primary-color), var(--accent-color));
    color: white;
    padding: 4rem 0;
    text-align: center;
}

.btn {
    padding: 0.75rem 1.5rem;
    border-radius: 4px;
    text-decoration: none;
    display: inline-block;
}

.btn-primary {
    background: var(--primary-color);
    color: white;
}
```

## Step 10: Create Custom JavaScript

```javascript
// assets/js/main.js

document.addEventListener('DOMContentLoaded', function() {
    // Initialize custom functionality
    initCustomForms();
    initAnimations();
});

function initCustomForms() {
    const forms = document.querySelectorAll('.custom-validation');
    forms.forEach(form => {
        form.addEventListener('submit', validateForm);
    });
}

function initAnimations() {
    const observer = new IntersectionObserver((entries) => {
        entries.forEach(entry => {
            if (entry.isIntersecting) {
                entry.target.classList.add('visible');
            }
        });
    });
    
    document.querySelectorAll('.animate-on-scroll').forEach(el => {
        observer.observe(el);
    });
}
```

## Step 11: Register Template Hooks

```php
// hooks.php

add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    return '<link href="template.css" rel="stylesheet">';
});

add_hook('ClientAreaFooterOutput', 1, function($vars) {
    return '<script src="template.js"></script>';
});
```

## Step 12: Activate Template

Navigate to: Setup > Client Area Design > Choose Theme

Select "My Custom Template"

## Step 13: Test Template

1. Browse all pages
2. Test responsive design
3. Check functionality
4. Validate HTML/CSS
5. Test form submissions

## Template Development Checklist

- [ ] Template directory created
- [ ] Template configuration created
- [ ] Header template customized
- [ ] Footer template customized
- [ ] Homepage template created
- [ ] Cart template created
- [ ] CSS styles created
- [ ] JavaScript created
- [ ] Hooks registered
- [ ] Template activated
- [ ] Template tested
