# WHMCS Spacing System Workflow

## Purpose
Implement a consistent spacing system in WHMCS using CSS custom properties.

## Prerequisites
- WHMCS installation
- CSS knowledge
- Template access

## Step-by-Step Process

### Step 1: Create Spacing Scale

**Create spacing.css:**
```css
/* /whmcs/templates/your_template/css/spacing.css */

:root {
    /* Base spacing unit */
    --space-unit: 0.25rem; /* 4px */
    
    /* Spacing Scale (4px base) */
    --space-0: 0;
    --space-1: calc(var(--space-unit) * 1);     /* 4px */
    --space-2: calc(var(--space-unit) * 2);     /* 8px */
    --space-3: calc(var(--space-unit) * 3);     /* 12px */
    --space-4: calc(var(--space-unit) * 4);     /* 16px */
    --space-5: calc(var(--space-unit) * 5);     /* 20px */
    --space-6: calc(var(--space-unit) * 6);     /* 24px */
    --space-8: calc(var(--space-unit) * 8);     /* 32px */
    --space-10: calc(var(--space-unit) * 10);   /* 40px */
    --space-12: calc(var(--space-unit) * 12);   /* 48px */
    --space-16: calc(var(--space-unit) * 16);   /* 64px */
    --space-20: calc(var(--space-unit) * 20);   /* 80px */
    --space-24: calc(var(--space-unit) * 24);   /* 96px */
    --space-32: calc(var(--space-unit) * 32);   /* 128px */
    
    /* Semantic Spacing */
    --space-xs: var(--space-1);
    --space-sm: var(--space-2);
    --space-md: var(--space-4);
    --space-lg: var(--space-6);
    --space-xl: var(--space-8);
    --space-2xl: var(--space-12);
    --space-3xl: var(--space-16);
    
    /* Component Spacing */
    --space-component-gap: var(--space-4);
    --space-section-gap: var(--space-8);
    --space-page-padding: var(--space-6);
}
```

### Step 2: Hook Into Template

```php
<?php
/**
 * Add spacing CSS variables
 */
use WHMCS\View\Asset;

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    return '<link rel="stylesheet" href="' . 
           Asset::url('/templates/' . $vars['template'] . '/css/spacing.css') . '">';
});
```

### Step 3: Margin Utilities

```css
/* Margin Utilities */

/* All sides */
.m-0 { margin: var(--space-0) !important; }
.m-1 { margin: var(--space-1) !important; }
.m-2 { margin: var(--space-2) !important; }
.m-3 { margin: var(--space-3) !important; }
.m-4 { margin: var(--space-4) !important; }
.m-5 { margin: var(--space-5) !important; }
.m-6 { margin: var(--space-6) !important; }
.m-8 { margin: var(--space-8) !important; }
.m-auto { margin: auto !important; }

/* Vertical (top + bottom) */
.my-0 { margin-top: var(--space-0) !important; margin-bottom: var(--space-0) !important; }
.my-1 { margin-top: var(--space-1) !important; margin-bottom: var(--space-1) !important; }
.my-2 { margin-top: var(--space-2) !important; margin-bottom: var(--space-2) !important; }
.my-4 { margin-top: var(--space-4) !important; margin-bottom: var(--space-4) !important; }
.my-6 { margin-top: var(--space-6) !important; margin-bottom: var(--space-6) !important; }
.my-8 { margin-top: var(--space-8) !important; margin-bottom: var(--space-8) !important; }
.my-auto { margin-top: auto !important; margin-bottom: auto !important; }

/* Horizontal (left + right) */
.mx-0 { margin-left: var(--space-0) !important; margin-right: var(--space-0) !important; }
.mx-auto { margin-left: auto !important; margin-right: auto !important; }

/* Individual sides */
.mt-0 { margin-top: var(--space-0) !important; }
.mt-1 { margin-top: var(--space-1) !important; }
.mt-2 { margin-top: var(--space-2) !important; }
.mt-4 { margin-top: var(--space-4) !important; }
.mt-6 { margin-top: var(--space-6) !important; }
.mt-8 { margin-top: var(--space-8) !important; }

.mb-0 { margin-bottom: var(--space-0) !important; }
.mb-1 { margin-bottom: var(--space-1) !important; }
.mb-2 { margin-bottom: var(--space-2) !important; }
.mb-4 { margin-bottom: var(--space-4) !important; }
.mb-6 { margin-bottom: var(--space-6) !important; }
.mb-8 { margin-bottom: var(--space-8) !important; }

.ml-0 { margin-left: var(--space-0) !important; }
.ml-1 { margin-left: var(--space-1) !important; }
.ml-2 { margin-left: var(--space-2) !important; }
.ml-4 { margin-left: var(--space-4) !important; }
.ml-auto { margin-left: auto !important; }

.mr-0 { margin-right: var(--space-0) !important; }
.mr-1 { margin-right: var(--space-1) !important; }
.mr-2 { margin-right: var(--space-2) !important; }
.mr-4 { margin-right: var(--space-4) !important; }
.mr-auto { margin-right: auto !important; }
```

### Step 4: Padding Utilities

```css
/* Padding Utilities */

/* All sides */
.p-0 { padding: var(--space-0) !important; }
.p-1 { padding: var(--space-1) !important; }
.p-2 { padding: var(--space-2) !important; }
.p-3 { padding: var(--space-3) !important; }
.p-4 { padding: var(--space-4) !important; }
.p-5 { padding: var(--space-5) !important; }
.p-6 { padding: var(--space-6) !important; }
.p-8 { padding: var(--space-8) !important; }

/* Vertical */
.py-0 { padding-top: var(--space-0) !important; padding-bottom: var(--space-0) !important; }
.py-1 { padding-top: var(--space-1) !important; padding-bottom: var(--space-1) !important; }
.py-2 { padding-top: var(--space-2) !important; padding-bottom: var(--space-2) !important; }
.py-4 { padding-top: var(--space-4) !important; padding-bottom: var(--space-4) !important; }
.py-6 { padding-top: var(--space-6) !important; padding-bottom: var(--space-6) !important; }
.py-8 { padding-top: var(--space-8) !important; padding-bottom: var(--space-8) !important; }

/* Horizontal */
.px-0 { padding-left: var(--space-0) !important; padding-right: var(--space-0) !important; }
.px-1 { padding-left: var(--space-1) !important; padding-right: var(--space-1) !important; }
.px-2 { padding-left: var(--space-2) !important; padding-right: var(--space-2) !important; }
.px-4 { padding-left: var(--space-4) !important; padding-right: var(--space-4) !important; }
.px-6 { padding-left: var(--space-6) !important; padding-right: var(--space-6) !important; }

/* Individual sides */
.pt-0 { padding-top: var(--space-0) !important; }
.pt-4 { padding-top: var(--space-4) !important; }
.pt-6 { padding-top: var(--space-6) !important; }
.pt-8 { padding-top: var(--space-8) !important; }

.pb-0 { padding-bottom: var(--space-0) !important; }
.pb-4 { padding-bottom: var(--space-4) !important; }
.pb-6 { padding-bottom: var(--space-6) !important; }
.pb-8 { padding-bottom: var(--space-8) !important; }

.pl-0 { padding-left: var(--space-0) !important; }
.pl-4 { padding-left: var(--space-4) !important; }

.pr-0 { padding-right: var(--space-0) !important; }
.pr-4 { padding-right: var(--space-4) !important; }
```

### Step 5: Gap Utilities

```css
/* Gap Utilities (for flexbox/grid) */
.gap-0 { gap: var(--space-0) !important; }
.gap-1 { gap: var(--space-1) !important; }
.gap-2 { gap: var(--space-2) !important; }
.gap-3 { gap: var(--space-3) !important; }
.gap-4 { gap: var(--space-4) !important; }
.gap-6 { gap: var(--space-6) !important; }
.gap-8 { gap: var(--space-8) !important; }

.gap-x-4 { column-gap: var(--space-4) !important; }
.gap-y-4 { row-gap: var(--space-4) !important; }
```

### Step 6: WHMCS Component Spacing

```css
/* WHMCS Specific Spacing */

/* Container */
.whmcs-container {
    padding-left: var(--space-page-padding);
    padding-right: var(--space-page-padding);
}

/* Cards */
.whmcs-card {
    padding: var(--space-6);
    margin-bottom: var(--space-6);
}

/* Forms */
.whmcs-form-group {
    margin-bottom: var(--space-4);
}

.whmcs-form-actions {
    margin-top: var(--space-6);
    padding-top: var(--space-6);
}

/* Navigation */
.whmcs-nav {
    padding: var(--space-4) 0;
}

.whmcs-nav-item {
    padding: var(--space-2) var(--space-4);
    margin-right: var(--space-2);
}

/* Tables */
.whmcs-table {
    border-spacing: 0;
}

.whmcs-table th,
.whmcs-table td {
    padding: var(--space-3) var(--space-4);
}

/* Buttons */
.btn {
    padding: var(--space-2) var(--space-4);
}

.btn-lg {
    padding: var(--space-3) var(--space-6);
}

.btn-sm {
    padding: var(--space-1) var(--space-3);
}

/* Sections */
.section {
    padding-top: var(--space-section-gap);
    padding-bottom: var(--space-section-gap);
}

.section-lg {
    padding-top: var(--space-2xl);
    padding-bottom: var(--space-2xl);
}
```

### Step 7: Responsive Spacing

```css
/* Responsive Spacing */

/* Small devices (landscape phones, 576px and up) */
@media (min-width: 576px) {
    .sm\:m-0 { margin: var(--space-0) !important; }
    .sm\:m-4 { margin: var(--space-4) !important; }
    .sm\:p-4 { padding: var(--space-4) !important; }
}

/* Medium devices (tablets, 768px and up) */
@media (min-width: 768px) {
    .md\:m-auto { margin: auto !important; }
    .md\:p-6 { padding: var(--space-6) !important; }
    .md\:py-8 { padding-top: var(--space-8) !important; padding-bottom: var(--space-8) !important; }
}

/* Large devices (desktops, 992px and up) */
@media (min-width: 992px) {
    .lg\:p-8 { padding: var(--space-8) !important; }
    .lg\:gap-6 { gap: var(--space-6) !important; }
}

/* Extra large devices (large desktops, 1200px and up) */
@media (min-width: 1200px) {
    .xl\:p-10 { padding: var(--space-10) !important; }
}
```

### Step 8: Template Usage Examples

```smarty
{* Card with consistent spacing *}
<div class="card p-4 mb-6">
    <h3 class="mt-0 mb-4">{$title}</h3>
    <div class="content">
        {$description}
    </div>
</div>

{* Form layout *}
<form class="whmcs-form p-4">
    <div class="whmcs-form-group">
        <label class="mb-2">Field Label</label>
        <input type="text" class="form-control">
    </div>
    <div class="whmcs-form-actions mt-6 pt-4">
        <button type="submit" class="btn btn-primary">Submit</button>
    </div>
</form>

{* Grid layout with gaps *}
<div class="products-grid" style="display: grid; grid-template-columns: repeat(auto-fill, minmax(250px, 1fr)); gap: var(--space-6);">
    {foreach from=$products item=product}
        <div class="product-card p-4">{$product.name}</div>
    {/foreach}
</div>

{* Flex layout *}
<div class="d-flex gap-4 flex-wrap">
    <div class="flex-item">Item 1</div>
    <div class="flex-item">Item 2</div>
    <div class="flex-item">Item 3</div>
</div>
```

### Step 9: Section Layout System

```css
/* Section Layout Utilities */
.section-padding {
    padding-top: var(--space-section-gap);
    padding-bottom: var(--space-section-gap);
}

.section-padding-lg {
    padding-top: var(--space-2xl);
    padding-bottom: var(--space-2xl);
}

.section-margin {
    margin-top: var(--space-section-gap);
    margin-bottom: var(--space-section-gap);
}

/* Container */
.container-custom {
    width: 100%;
    max-width: 1200px;
    margin-left: auto;
    margin-right: auto;
    padding-left: var(--space-page-padding);
    padding-right: var(--space-page-padding);
}
```

## Best Practices
- Always use spacing scale values
- Avoid arbitrary pixel values
- Use shorthand utilities for common patterns
- Maintain vertical rhythm consistency
- Use gap for flexbox/grid spacing
- Consider responsive breakpoints
- Test spacing across viewports
- Keep spacing consistent with design system
