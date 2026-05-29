# WHMCS Responsive Breakpoints Workflow

## Purpose
Configure and implement responsive design breakpoints in WHMCS.

## Prerequisites
- WHMCS installation
- CSS knowledge
- Template access
- Responsive design understanding

## Step-by-Step Process

### Step 1: Define Breakpoint System

**Create breakpoints.css:**
```css
/* /whmcs/templates/your_template/css/breakpoints.css */

/* 
 * Breakpoint System:
 * - Mobile First approach
 * - Breakpoints: sm, md, lg, xl, xxl
 */

/* Base (Mobile) - No media query needed */

/* Small devices (landscape phones, 576px and up) */
@media (min-width: 576px) {
    :root {
        --container-sm: 540px;
    }
}

/* Medium devices (tablets, 768px and up) */
@media (min-width: 768px) {
    :root {
        --container-md: 720px;
    }
}

/* Large devices (desktops, 992px and up) */
@media (min-width: 992px) {
    :root {
        --container-lg: 960px;
    }
}

/* Extra large devices (large desktops, 1200px and up) */
@media (min-width: 1200px) {
    :root {
        --container-xl: 1140px;
    }
}

/* XXL devices (large desktops, 1400px and up) */
@media (min-width: 1400px) {
    :root {
        --container-xxl: 1320px;
    }
}
```

### Step 2: Hook Breakpoints Into Template

```php
<?php
/**
 * Add responsive CSS
 */
use WHMCS\View\Asset;

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    return '<link rel="stylesheet" href="' . 
           Asset::url('/templates/' . $vars['template'] . '/css/breakpoints.css') . '">';
});
```

### Step 3: Grid System Implementation

```css
/* Grid System */
.row {
    display: flex;
    flex-wrap: wrap;
    margin-left: calc(var(--space-4) * -0.5);
    margin-right: calc(var(--space-4) * -0.5);
}

.col {
    flex: 1 0 0%;
    padding-left: calc(var(--space-4) * 0.5);
    padding-right: calc(var(--space-4) * 0.5);
}

/* Column sizes */
.col-auto { flex: 0 0 auto; width: auto; }
.col-1 { flex: 0 0 8.333333%; max-width: 8.333333%; }
.col-2 { flex: 0 0 16.666667%; max-width: 16.666667%; }
.col-3 { flex: 0 0 25%; max-width: 25%; }
.col-4 { flex: 0 0 33.333333%; max-width: 33.333333%; }
.col-5 { flex: 0 0 41.666667%; max-width: 41.666667%; }
.col-6 { flex: 0 0 50%; max-width: 50%; }
.col-7 { flex: 0 0 58.333333%; max-width: 58.333333%; }
.col-8 { flex: 0 0 66.666667%; max-width: 66.666667%; }
.col-9 { flex: 0 0 75%; max-width: 75%; }
.col-10 { flex: 0 0 83.333333%; max-width: 83.333333%; }
.col-11 { flex: 0 0 91.666667%; max-width: 91.666667%; }
.col-12 { flex: 0 0 100%; max-width: 100%; }

/* Responsive columns */
@media (min-width: 768px) {
    .col-md-auto { flex: 0 0 auto; width: auto; }
    .col-md-1 { flex: 0 0 8.333333%; max-width: 8.333333%; }
    .col-md-2 { flex: 0 0 16.666667%; max-width: 16.666667%; }
    .col-md-3 { flex: 0 0 25%; max-width: 25%; }
    .col-md-4 { flex: 0 0 33.333333%; max-width: 33.333333%; }
    .col-md-6 { flex: 0 0 50%; max-width: 50%; }
    .col-md-8 { flex: 0 0 66.666667%; max-width: 66.666667%; }
    .col-md-12 { flex: 0 0 100%; max-width: 100%; }
}

@media (min-width: 992px) {
    .col-lg-auto { flex: 0 0 auto; width: auto; }
    .col-lg-2 { flex: 0 0 16.666667%; max-width: 16.666667%; }
    .col-lg-3 { flex: 0 0 25%; max-width: 25%; }
    .col-lg-4 { flex: 0 0 33.333333%; max-width: 33.333333%; }
    .col-lg-6 { flex: 0 0 50%; max-width: 50%; }
    .col-lg-8 { flex: 0 0 66.666667%; max-width: 66.666667%; }
    .col-lg-9 { flex: 0 0 75%; max-width: 75%; }
    .col-lg-12 { flex: 0 0 100%; max-width: 100%; }
}
```

### Step 4: WHMCS Component Responsive Styles

```css
/* WHMCS Navigation Responsive */
.navbar-collapse {
    display: none;
}

.navbar-collapse.show,
.navbar-collapse.collapsing {
    display: block;
}

@media (min-width: 992px) {
    .navbar-collapse {
        display: flex !important;
    }
    
    .navbar-toggle {
        display: none;
    }
}

/* WHMCS Tables Responsive */
.table-responsive {
    overflow-x: auto;
}

@media (max-width: 767px) {
    .table-responsive {
        font-size: 0.875rem;
    }
    
    .table th,
    .table td {
        padding: 0.5rem;
        white-space: nowrap;
    }
}

/* WHMCS Forms Responsive */
@media (max-width: 575px) {
    .form-group {
        margin-bottom: 1rem;
    }
    
    .form-actions {
        display: flex;
        flex-direction: column;
    }
    
    .form-actions .btn {
        width: 100%;
        margin-bottom: 0.5rem;
    }
}

/* WHMCS Cards Responsive */
@media (min-width: 768px) {
    .product-grid {
        display: grid;
        grid-template-columns: repeat(2, 1fr);
        gap: 1.5rem;
    }
}

@media (min-width: 992px) {
    .product-grid {
        grid-template-columns: repeat(3, 1fr);
    }
}

@media (min-width: 1200px) {
    .product-grid {
        grid-template-columns: repeat(4, 1fr);
    }
}
```

### Step 5: Display Utilities

```css
/* Display utilities */
.d-none { display: none !important; }
.d-block { display: block !important; }
.d-flex { display: flex !important; }
.d-grid { display: grid !important; }
.d-inline { display: inline !important; }
.d-inline-block { display: inline-block !important; }
.d-inline-flex { display: inline-flex !important; }

/* Responsive display */
@media (min-width: 576px) {
    .d-sm-none { display: none !important; }
    .d-sm-block { display: block !important; }
    .d-sm-flex { display: flex !important; }
}

@media (min-width: 768px) {
    .d-md-none { display: none !important; }
    .d-md-block { display: block !important; }
    .d-md-flex { display: flex !important; }
}

@media (min-width: 992px) {
    .d-lg-none { display: none !important; }
    .d-lg-block { display: block !important; }
    .d-lg-flex { display: flex !important; }
}

@media (min-width: 1200px) {
    .d-xl-none { display: none !important; }
    .d-xl-block { display: block !important; }
    .d-xl-flex { display: flex !important; }
}
```

### Step 6: WHMCS Sidebar Responsive

```css
/* Sidebar Behavior */
.main-content {
    width: 100%;
}

.sidebar {
    width: 100%;
    margin-bottom: 2rem;
}

@media (min-width: 992px) {
    .main-wrapper {
        display: flex;
    }
    
    .main-content {
        flex: 1;
        max-width: calc(100% - 280px);
    }
    
    .sidebar {
        width: 260px;
        flex-shrink: 0;
        margin-bottom: 0;
    }
}

/* Mobile Navigation Menu */
.mobile-menu {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: var(--color-background);
    z-index: 1000;
    transform: translateX(-100%);
    transition: transform 0.3s ease;
}

.mobile-menu.active {
    transform: translateX(0);
}
```

### Step 7: Image Responsive

```css
/* Responsive Images */
img {
    max-width: 100%;
    height: auto;
}

/* Picture element for art direction */
.picture-source {
    display: none;
}

@media (min-width: 768px) {
    .picture-source.mobile {
        display: none;
    }
    .picture-source.tablet {
        display: block;
    }
}

@media (min-width: 1200px) {
    .picture-source.mobile,
    .picture-source.tablet {
        display: none;
    }
    .picture-source.desktop {
        display: block;
    }
}

/* Lazy loading placeholder */
img.lazy {
    background-color: #f0f0f0;
    min-height: 200px;
}
```

### Step 8: WHMCS Checkout Responsive

```css
/* Checkout Page Responsive */
.checkout-container {
    display: flex;
    flex-direction: column;
    gap: 2rem;
}

.checkout-summary {
    order: 2;
}

.checkout-form {
    order: 1;
}

@media (min-width: 992px) {
    .checkout-container {
        flex-direction: row;
        align-items: flex-start;
    }
    
    .checkout-form {
        flex: 1;
        max-width: 600px;
    }
    
    .checkout-summary {
        width: 340px;
        flex-shrink: 0;
        position: sticky;
        top: 2rem;
    }
}

/* Cart Items Responsive */
.cart-item {
    display: flex;
    flex-direction: column;
    padding: 1rem 0;
    border-bottom: 1px solid var(--color-border);
}

@media (min-width: 576px) {
    .cart-item {
        flex-direction: row;
        align-items: center;
        justify-content: space-between;
    }
    
    .cart-item-details {
        flex: 1;
    }
    
    .cart-item-actions {
        display: flex;
        gap: 0.5rem;
    }
}
```

### Step 9: WHMCS Dashboard Responsive

```css
/* Client Dashboard Responsive */
.dashboard-stats {
    display: grid;
    grid-template-columns: 1fr;
    gap: 1rem;
}

@media (min-width: 576px) {
    .dashboard-stats {
        grid-template-columns: repeat(2, 1fr);
    }
}

@media (min-width: 992px) {
    .dashboard-stats {
        grid-template-columns: repeat(4, 1fr);
    }
}

/* Service Cards */
.service-card {
    padding: 1.5rem;
}

@media (max-width: 767px) {
    .service-card {
        padding: 1rem;
    }
    
    .service-header {
        flex-direction: column;
        align-items: flex-start;
    }
    
    .service-actions {
        width: 100%;
        margin-top: 1rem;
    }
}

/* Invoice Responsive */
.invoice-table {
    font-size: 0.875rem;
}

@media (max-width: 767px) {
    .invoice-table {
        display: block;
        overflow-x: auto;
    }
    
    .invoice-totals {
        margin-top: 1rem;
    }
}
```

### Step 10: Test Responsive Design

```javascript
// Test breakpoints
function testBreakpoints() {
    const breakpoints = [
        { name: 'xs', width: 320 },
        { name: 'sm', width: 576 },
        { name: 'md', width: 768 },
        { name: 'lg', width: 992 },
        { name: 'xl', width: 1200 },
        { name: 'xxl', width: 1400 }
    ];
    
    const currentWidth = window.innerWidth;
    
    breakpoints.forEach(bp => {
        const isActive = currentWidth >= bp.width;
        console.log(`${bp.name}: ${isActive ? 'active' : 'inactive'} (${bp.width}px)`);
    });
}

window.addEventListener('resize', debounce(testBreakpoints, 200));

// Visual breakpoint indicator
(function() {
    const indicator = document.createElement('div');
    indicator.style.cssText = 'position:fixed;bottom:10px;right:10px;background:#000;color:#fff;padding:5px 10px;font-size:12px;z-index:9999;';
    document.body.appendChild(indicator);
    
    function updateIndicator() {
        const width = window.innerWidth;
        let breakpoint = 'xs';
        if (width >= 1400) breakpoint = 'xxl';
        else if (width >= 1200) breakpoint = 'xl';
        else if (width >= 992) breakpoint = 'lg';
        else if (width >= 768) breakpoint = 'md';
        else if (width >= 576) breakpoint = 'sm';
        indicator.textContent = breakpoint + ' (' + width + 'px)';
    }
    
    updateIndicator();
    window.addEventListener('resize', updateIndicator);
})();
```

## Best Practices
- Use mobile-first approach
- Test all breakpoints during development
- Use relative units (%, rem) over fixed pixels
- Test on actual devices when possible
- Consider touch targets (minimum 44px)
- Maintain readability at all sizes
- Prioritize content hierarchy
- Use flexbox and grid for layouts
- Test navigation on mobile devices
- Consider performance on mobile
