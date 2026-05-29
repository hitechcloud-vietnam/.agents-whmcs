# WHMCS Responsive Design

## Overview

Create responsive WHMCS templates that work across all devices.

## Responsive Grid System

```css
/* Bootstrap 3 Grid */
.container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 15px;
}

.row {
    margin: 0 -15px;
    display: flex;
    flex-wrap: wrap;
}

/* Column sizes */
.col-xs-1, .col-sm-1, .col-md-1, .col-lg-1 { width: 8.33%; }
.col-xs-2, .col-sm-2, .col-md-2, .col-lg-2 { width: 16.66%; }
.col-xs-3, .col-sm-3, .col-md-3, .col-lg-3 { width: 25%; }
.col-xs-4, .col-sm-4, .col-md-4, .col-lg-4 { width: 33.33%; }
.col-xs-6, .col-sm-6, .col-md-6, .col-lg-6 { width: 50%; }
.col-xs-12, .col-sm-12, .col-md-12, .col-lg-12 { width: 100%; }

/* Mobile first - extra small screens */
@media (max-width: 767px) {
    .container { padding: 0 10px; }
    .col-xs-6 { width: 50%; }
    .col-xs-12 { width: 100%; }
}

/* Small screens - tablets */
@media (min-width: 768px) and (max-width: 991px) {
    .col-sm-4 { width: 33.33%; }
    .col-sm-6 { width: 50%; }
}

/* Medium screens - desktop */
@media (min-width: 992px) and (max-width: 1199px) {
    .col-md-3 { width: 25%; }
}

/* Large screens */
@media (min-width: 1200px) {
    .col-lg-4 { width: 33.33%; }
}
```

## Responsive Navigation

```css
/* Mobile navigation */
.navbar-toggle {
    display: none;
    padding: 10px;
    background: #333;
    border: none;
    cursor: pointer;
}

@media (max-width: 767px) {
    .navbar-toggle {
        display: block;
    }
    
    .navbar-collapse {
        display: none;
        width: 100%;
        background: #f8f9fa;
    }
    
    .navbar-collapse.active {
        display: block;
    }
    
    .navbar-nav {
        flex-direction: column;
        width: 100%;
    }
    
    .navbar-nav li {
        width: 100%;
        border-bottom: 1px solid #ddd;
    }
    
    .navbar-nav li a {
        padding: 15px;
        display: block;
    }
}

/* Desktop navigation */
@media (min-width: 768px) {
    .navbar-collapse {
        display: flex !important;
    }
}
```

## Responsive Tables

```css
/* Horizontal scroll for tables on mobile */
.table-responsive {
    overflow-x: auto;
    -webkit-overflow-scrolling: touch;
}

@media (max-width: 767px) {
    /* Stack table cells */
    .table {
        display: block;
    }
    
    .table thead {
        display: none;
    }
    
    .table tbody {
        display: block;
    }
    
    .table tbody tr {
        display: block;
        margin-bottom: 15px;
        border: 1px solid #ddd;
        border-radius: 4px;
    }
    
    .table tbody td {
        display: flex;
        justify-content: space-between;
        padding: 10px 15px;
        border: none;
        border-bottom: 1px solid #eee;
    }
    
    .table tbody td::before {
        content: attr(data-label);
        font-weight: 600;
        color: #666;
    }
}
```

```smarty
<table class="table table-striped table-responsive">
    <thead>
        <tr>
            <th>Invoice #</th>
            <th>Amount</th>
            <th>Status</th>
        </tr>
    </thead>
    <tbody>
        {foreach from=$invoices item=invoice}
            <tr>
                <td data-label="Invoice #">{$invoice->id}</td>
                <td data-label="Amount">{$invoice->total|currency}</td>
                <td data-label="Status">{$invoice->status}</td>
            </tr>
        {/foreach}
    </tbody>
</table>
```

## Responsive Cards

```css
/* Card grid */
.card-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 20px;
}

@media (max-width: 767px) {
    .card-grid {
        grid-template-columns: 1fr;
    }
}

.card {
    background: white;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    overflow: hidden;
}

.card-header {
    padding: 15px 20px;
    background: #f8f9fa;
    border-bottom: 1px solid #eee;
}

.card-body {
    padding: 20px;
}

.card-footer {
    padding: 15px 20px;
    background: #f8f9fa;
    border-top: 1px solid #eee;
}
```

## Responsive Forms

```css
@media (max-width: 767px) {
    .form-group {
        margin-bottom: 15px;
    }
    
    .form-label {
        display: block;
        margin-bottom: 5px;
    }
    
    .form-control {
        width: 100%;
        font-size: 16px; /* Prevents zoom on iOS */
    }
    
    .btn {
        width: 100%;
        margin-bottom: 10px;
    }
    
    .btn-group .btn {
        width: auto;
    }
}
```

## Responsive Typography

```css
:root {
    --font-size-base: 16px;
    --heading-scale: 1.25;
}

h1 { font-size: 2rem; }
h2 { font-size: 1.75rem; }
h3 { font-size: 1.5rem; }
h4 { font-size: 1.25rem; }
h5 { font-size: 1rem; }
h6 { font-size: 0.875rem; }

@media (min-width: 768px) {
    :root {
        --font-size-base: 14px;
    }
    
    h1 { font-size: 2.5rem; }
    h2 { font-size: 2rem; }
}

/* Better text on mobile */
@media (max-width: 767px) {
    body {
        font-size: 16px; /* Prevents zoom on focus */
        -webkit-text-size-adjust: 100%;
    }
}
```

## Viewport Units

```css
/* Using vw/vh for full-height sections */
.hero-section {
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
}

/* Sidebar on desktop, drawer on mobile */
.sidebar {
    position: fixed;
    top: 0;
    left: 0;
    width: 250px;
    height: 100vh;
    background: #333;
    z-index: 1000;
}

.main-content {
    margin-left: 250px;
}

@media (max-width: 767px) {
    .sidebar {
        transform: translateX(-100%);
        transition: transform 0.3s ease;
    }
    
    .sidebar.open {
        transform: translateX(0);
    }
    
    .main-content {
        margin-left: 0;
    }
}
```

## Images and Media

```css
/* Responsive images */
img {
    max-width: 100%;
    height: auto;
}

/* Object fit for backgrounds */
.hero-bg {
    background-size: cover;
    background-position: center;
    background-repeat: no-repeat;
}

/* Video embeds */
.video-container {
    position: relative;
    padding-bottom: 56.25%; /* 16:9 */
    height: 0;
    overflow: hidden;
}

.video-container iframe {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}
```

## Touch-Friendly Elements

```css
/* Touch targets minimum 44x44px */
.btn, .nav-link, .form-control {
    min-height: 44px;
    padding: 10px 15px;
}

.checkbox, .radio {
    min-height: 44px;
    display: flex;
    align-items: center;
}

/* Hover states for touch devices */
@media (hover: none) {
    .btn:hover {
        opacity: 0.9;
    }
    
    .dropdown:hover .dropdown-menu {
        display: none;
    }
    
    .dropdown.open .dropdown-menu {
        display: block;
    }
}
```

## Breakpoints Reference

| Breakpoint | Width | Device Type |
|------------|-------|-------------|
| xs | < 576px | Mobile phones |
| sm | 576 - 767px | Large phones |
| md | 768 - 991px | Tablets |
| lg | 992 - 1199px | Small desktops |
| xl | >= 1200px | Large desktops |

## Best Practices

1. **Mobile first** - Start with mobile styles, enhance for desktop
2. **Use breakpoints** - Test at all screen sizes
3. **Touch targets** - Minimum 44x44px for interactive elements
4. **Responsive images** - Use max-width: 100%
5. **Test real devices** - Emulators aren't always accurate

## Related Documentation

- [WHMCS CSS Customization](/docs/whmcs-css-customization.md)
- [WHMCS Client Area](/docs/whmcs-client-area.md)