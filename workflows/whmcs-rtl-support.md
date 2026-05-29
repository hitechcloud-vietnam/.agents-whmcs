# WHMCS RTL Support Workflow

## Purpose
Implement RTL (Right-to-Left) language support in WHMCS for Arabic, Hebrew, and other RTL languages.

## Prerequisites
- WHMCS installation
- Template access
- CSS knowledge
- Understanding of RTL layout requirements

## Step-by-Step Process

### Step 1: Detect RTL Languages

**Create rtl-detection.php hook:**
```php
<?php
/**
 * Detect RTL language and set template variable
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $rtlLanguages = ['arabic', 'farsi', 'hebrew', 'urdu', 'pashto', 'azerbaijani'];
    
    $currentLang = isset($_SESSION['Language']) ? strtolower($_SESSION['Language']) : 'english';
    $isRtl = in_array($currentLang, $rtlLanguages);
    
    // Also check for _rtl suffix in language files
    if (!$isRtl && strpos($currentLang, '_rtl') !== false) {
        $isRtl = true;
    }
    
    return [
        'isRTL' => $isRtl,
        'direction' => $isRtl ? 'rtl' : 'ltr'
    ];
});
```

### Step 2: HTML Direction Setup

**Add to template head:**
```html
<html dir="{$direction}" lang="{$language}">
```

**Or via hook:**
```php
<?php
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $direction = $vars['isRTL'] ? 'rtl' : 'ltr';
    $lang = $vars['language'] ?? 'en';
    
    return '<script>
        document.documentElement.dir = "' . $direction . '";
        document.documentElement.lang = "' . $lang . '";
    </script>';
});
```

### Step 3: Create RTL CSS File

**Create rtl.css:**
```css
/* /whmcs/templates/your_template/css/rtl.css */

/* Direction-aware utilities */
[dir="rtl"] {
    /* Text alignment */
    .text-left { text-align: right; }
    .text-right { text-align: left; }
    
    /* Float direction */
    .float-left { float: right; }
    .float-right { float: left; }
    
    /* Margins */
    .ml-auto { margin-left: auto; margin-right: 0; }
    .mr-auto { margin-right: auto; margin-left: 0; }
    .ml-1 { margin-left: 0; margin-right: 0.25rem; }
    .mr-1 { margin-right: 0; margin-left: 0.25rem; }
    .ml-2 { margin-left: 0; margin-right: 0.5rem; }
    .mr-2 { margin-right: 0; margin-left: 0.5rem; }
    .ml-3 { margin-left: 0; margin-right: 0.75rem; }
    .mr-3 { margin-right: 0; margin-left: 0.75rem; }
    .ml-4 { margin-left: 0; margin-right: 1rem; }
    .mr-4 { margin-right: 0; margin-left: 1rem; }
    
    /* Padding */
    .pl-1 { padding-left: 0; padding-right: 0.25rem; }
    .pr-1 { padding-right: 0; padding-left: 0.25rem; }
    .pl-2 { padding-left: 0; padding-right: 0.5rem; }
    .pr-2 { padding-right: 0; padding-left: 0.5rem; }
    .pl-3 { padding-left: 0; padding-right: 0.75rem; }
    .pr-3 { padding-right: 0; padding-left: 0.75rem; }
    .pl-4 { padding-left: 0; padding-right: 1rem; }
    .pr-4 { padding-right: 0; padding-left: 1rem; }
    
    /* Border radius */
    .rounded-left { border-radius: 0 0.25rem 0.25rem 0; }
    .rounded-right { border-radius: 0.25rem 0 0 0.25rem; }
}

/* Border utilities */
[dir="rtl"] .border-left { border-left: none; border-right: 1px solid; }
[dir="rtl"] .border-right { border-right: none; border-left: 1px solid; }
```

### Step 4: Flexbox RTL Support

```css
/* Flexbox direction */
[dir="rtl"] {
    /* Reverse flex direction */
    .flex-row { flex-direction: row-reverse; }
    
    /* Justify content */
    .justify-content-start { justify-content: flex-end; }
    .justify-content-end { justify-content: flex-start; }
    
    /* Spacing */
    .me-1 { margin-right: 0.25rem; margin-left: 0; }
    .ms-1 { margin-left: 0.25rem; margin-right: 0; }
    .pe-1 { padding-right: 0.25rem; padding-left: 0; }
    .ps-1 { padding-left: 0.25rem; padding-right: 0; }
}

/* Bootstrap-like spacing utilities */
[dir="rtl"] {
    .ms-auto { margin-right: auto; margin-left: 0; }
    .me-auto { margin-left: auto; margin-right: 0; }
}
```

### Step 5: Grid RTL Support

```css
/* Grid gutters direction */
[dir="rtl"] .row {
    margin-left: 0;
    margin-right: calc(var(--space-4) * -0.5);
}

[dir="rtl"] .row > * {
    padding-left: 0;
    padding-right: calc(var(--space-4) * 0.5);
}
```

### Step 6: WHMCS Component RTL Fixes

```css
/* Navigation */
[dir="rtl"] .navbar-nav {
    padding-right: 0;
}

[dir="rtl"] .nav-item + .nav-item {
    margin-left: 0;
    margin-right: 1rem;
}

/* Dropdown menus */
[dir="rtl"] .dropdown-menu {
    left: auto;
    right: 0;
    text-align: right;
}

[dir="rtl"] .dropdown-toggle::after {
    margin-left: 0;
    margin-right: 0.25rem;
}

/* Buttons with icons */
[dir="rtl"] .btn-icon-right {
    margin-left: 0;
    margin-right: 0.5rem;
}

[dir="rtl"] .btn-icon-left {
    margin-right: 0;
    margin-left: 0.5rem;
}

/* Forms */
[dir="rtl"] .input-group {
    flex-direction: row-reverse;
}

[dir="rtl"] .input-group-text {
    border-radius: 0 0.25rem 0.25rem 0;
}

[dir="rtl"] .form-control {
    border-radius: 0.25rem 0 0 0.25rem;
}

[dir="rtl"] .has-feedback .form-control {
    padding-left: 0;
    padding-right: 2.25rem;
}

[dir="rtl"] .form-control-feedback {
    left: auto;
    right: 0;
}

/* Cards */
[dir="rtl"] .card-header:first-child {
    border-radius: 0;
}

[dir="rtl"] .card-header:first-child.rounded-0 {
    border-radius: 0 calc(var(--border-radius) - 1px) 0 0;
}

/* Tables */
[dir="rtl"] .table .text-right {
    text-align: left;
}

[dir="rtl"] .table .text-left {
    text-align: right;
}

/* Alerts */
[dir="rtl"] .alert-dismissible {
    padding-left: 3.5rem;
    padding-right: 1.5rem;
}

[dir="rtl"] .close {
    left: 0;
    right: auto;
}

/* Breadcrumbs */
[dir="rtl"] .breadcrumb {
    padding-right: 0;
}

[dir="rtl"] .breadcrumb-item + .breadcrumb-item::before {
    padding-left: 0.5rem;
    padding-right: 0;
    content: "\\f053"; /* FontAwesome RTL arrow */
}
```

### Step 7: Icons Direction

```css
/* RTL icon transformations */
[dir="rtl"] .icon-arrow-right {
    transform: scaleX(-1);
}

[dir="rtl"] .icon-chevron-right {
    transform: scaleX(-1);
}

[dir="rtl"] .icon-angle-right {
    transform: scaleX(-1);
}

/* Caret direction */
[dir="rtl"] .caret {
    margin-left: 0;
    margin-right: 0.5rem;
}

/* Checkbox/Radio fix */
[dir="rtl"] .custom-control-inline {
    margin-left: 0;
    margin-right: 1rem;
}
```

### Step 8: Hook RTL CSS Into Template

```php
<?php
/**
 * Load RTL CSS when needed
 */
use WHMCS\View\Asset;

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $css = '';
    $template = $vars['template'];
    
    // Load RTL CSS for RTL languages
    if (!empty($vars['isRTL'])) {
        $css .= '<link rel="stylesheet" href="' . 
                Asset::url('/templates/' . $template . '/css/rtl.css') . '">';
    }
    
    return $css;
});
```

### Step 9: Smarty Template Changes

```smarty
{* Set direction in HTML tag *}
<html dir="{$direction}" lang="{$LANG}">

{* Conditional content *}
<div class="row" {if $direction == 'rtl'}dir="rtl"{/if}>
    <div class="col-md-6">
        {if $direction == 'ltr'}
            Left content
        {else}
            Right content (RTL)
        {/if}
    </div>
</div>

{* RTL-aware icons *}
<i class="fa {if $direction == 'rtl'}fa-chevron-left{else}fa-chevron-right{/if}"></i>

{* Text alignment *}
<p class="{if $direction == 'rtl'}text-right{else}text-left{/if}">
    {$description}
</p>
```

### Step 10: Font Support for RTL

```css
/* Arabic font stack */
[dir="rtl"] body,
[dir="rtl"] input,
[dir="rtl"] select,
[dir="rtl"] textarea {
    font-family: 'Noto Sans Arabic', 'Arabic Typesetting', Tahoma, Arial, sans-serif;
}

/* Hebrew font stack */
[lang="he"] body,
[lang="he"] input,
[lang="he"] select,
[lang="he"] textarea {
    font-family: 'David Libre', 'Frank Ruhl Libre', 'Heebo', Arial, sans-serif;
}

/* Urdu font stack */
[lang="ur"] body,
[lang="ur"] input,
[lang="ur"] select,
[lang="ur"] textarea {
    font-family: 'Noto Nastaliq Urdu', 'Jameel Noori Nastaleeq', Arial, sans-serif;
}
```

### Step 11: WHMCS-Specific RTL Fixes

```css
/* WHMCS Client Area */
[dir="rtl"] #header {
    text-align: right;
}

[dir="rtl"] .header-nav {
    padding-right: 0;
}

[dir="rtl"] .header-nav > li {
    float: right;
}

/* Cart and Checkout */
[dir="rtl"] .cart-summary {
    border-left: none;
    border-right: 1px solid var(--color-border);
}

/* Domain Checker */
[dir="rtl"] .domain-availability {
    text-align: right;
}

/* Invoice */
[dir="rtl"] .invoice-table .text-right {
    text-align: left;
}

/* Support Tickets */
[dir="rtl"] .ticket-reply {
    margin-left: 0;
    margin-right: 3rem;
}

[dir="rtl"] .ticket-reply.client {
    margin-right: 0;
    margin-left: 3rem;
}

/* Pricing Tables */
[dir="rtl"] .pricing-features {
    text-align: right;
}

[dir="rtl"] .pricing-features li::before {
    margin-right: 0;
    margin-left: 0.5rem;
}
```

### Step 12: Test RTL Layout

```javascript
// Manual RTL testing
function toggleRTL() {
    const html = document.documentElement;
    const currentDir = html.getAttribute('dir');
    const newDir = currentDir === 'rtl' ? 'ltr' : 'rtl';
    
    html.setAttribute('dir', newDir);
    localStorage.setItem('test-rtl', newDir);
}

// Add to browser console for testing
console.log('Run toggleRTL() to switch between LTR and RTL for testing');
```

## Best Practices
- Use CSS logical properties where supported
- Test with actual RTL languages (Arabic, Hebrew)
- Use font stacks designed for RTL scripts
- Mirror icons that indicate direction
- Consider number formatting in RTL (LTR numbers preferred)
- Test all form elements in RTL mode
- Ensure text alignment follows reading direction
- Check third-party integrations for RTL support
- Provide RTL documentation for translators
