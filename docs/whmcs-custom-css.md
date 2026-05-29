# WHMCS Custom CSS

## Overview

Custom CSS in WHMCS allows you to customize the appearance of templates without modifying core template files. This enables branding consistency and visual improvements.

## Adding Custom CSS

### Method 1: Template Folder

Create a custom.css file in your template folder:

```
templates/your-template/
    custom.css
    style.css
    ...
```

### Method 2: Include in Header

```php
<?php
// In hooks/custom-css.php

add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    return '<link rel="stylesheet" href="' . 
           \WHMCS\Config\Setting::getValue('Template') . 
           '/custom.css">';
});
```

### Method 3: Inline Styles

```php
<?php
add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    $customCSS = '
        .btn-primary { background: #ff6600; }
        .navbar { border-bottom: 2px solid #ff6600; }
    ';
    return '<style>' . $customCSS . '</style>';
});
```

## Common Customizations

### Brand Colors

```css
/* Primary Brand Color */
.btn-primary,
.btn-success,
.bg-primary {
    background-color: #ff6600;
    border-color: #ff5500;
}

.btn-primary:hover,
.btn-primary:focus {
    background-color: #ff5500;
    border-color: #ff4400;
}

.text-primary,
a,
.link {
    color: #ff6600;
}

/* Link Hover */
a:hover {
    color: #ff5500;
}

/* Focus States */
.form-control:focus,
.btn:focus {
    border-color: #ff6600;
    box-shadow: 0 0 0 3px rgba(255, 102, 0, 0.25);
}
```

### Header Customization

```css
/* Navigation Bar */
.navbar {
    background-color: #ffffff;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.navbar-brand {
    font-size: 24px;
    font-weight: bold;
    color: #333 !important;
}

.navbar-nav > li > a {
    padding: 15px 20px;
    transition: color 0.3s ease;
}

.navbar-nav > li > a:hover,
.navbar-nav > li > a:focus {
    color: #ff6600;
}

/* Sticky Header */
.header-sticky {
    position: fixed;
    top: 0;
    width: 100%;
    z-index: 1000;
}
```

### Button Styles

```css
/* Primary Button */
.btn-primary {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    border: none;
    padding: 12px 30px;
    border-radius: 25px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 1px;
    transition: all 0.3s ease;
}

.btn-primary:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 15px rgba(102, 126, 234, 0.4);
}

/* Secondary Button */
.btn-secondary {
    background: transparent;
    border: 2px solid #333;
    color: #333;
}

.btn-secondary:hover {
    background: #333;
    color: #fff;
}

/* Icon Button */
.btn-icon {
    width: 40px;
    height: 40px;
    padding: 0;
    border-radius: 50%;
    display: inline-flex;
    align-items: center;
    justify-content: center;
}
```

### Form Styling

```css
/* Input Fields */
.form-control {
    border: 2px solid #e0e0e0;
    border-radius: 8px;
    padding: 12px 15px;
    transition: border-color 0.3s ease, box-shadow 0.3s ease;
}

.form-control:focus {
    border-color: #ff6600;
    box-shadow: 0 0 0 3px rgba(255, 102, 0, 0.1);
}

/* Labels */
.form-group label {
    font-weight: 600;
    color: #333;
    margin-bottom: 8px;
    display: block;
}

/* Custom Checkbox */
.custom-checkbox {
    display: flex;
    align-items: center;
    cursor: pointer;
}

.custom-checkbox input {
    width: 20px;
    height: 20px;
    margin-right: 10px;
}
```

### Card Components

```css
/* Product Cards */
.product-card {
    background: #fff;
    border-radius: 16px;
    box-shadow: 0 4px 20px rgba(0,0,0,0.08);
    transition: transform 0.3s ease, box-shadow 0.3s ease;
    overflow: hidden;
}

.product-card:hover {
    transform: translateY(-8px);
    box-shadow: 0 12px 40px rgba(0,0,0,0.12);
}

.product-card .card-header {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: #fff;
    padding: 30px;
    text-align: center;
}

.product-card .card-body {
    padding: 30px;
}

/* Feature List */
.product-card .features {
    list-style: none;
    padding: 0;
    margin: 0;
}

.product-card .features li {
    padding: 10px 0;
    border-bottom: 1px solid #f0f0f0;
}

.product-card .features li:last-child {
    border-bottom: none;
}

.product-card .features li::before {
    content: "✓";
    color: #4caf50;
    margin-right: 10px;
    font-weight: bold;
}
```

### Table Styling

```css
/* Data Tables */
.data-table {
    border-collapse: separate;
    border-spacing: 0;
}

.data-table thead th {
    background: #f8f9fa;
    border-bottom: 2px solid #dee2e6;
    font-weight: 600;
    text-transform: uppercase;
    font-size: 12px;
    letter-spacing: 0.5px;
}

.data-table tbody tr {
    transition: background-color 0.2s ease;
}

.data-table tbody tr:hover {
    background-color: #f8f9fa;
}

.data-table td {
    padding: 15px;
    vertical-align: middle;
}

/* Status Badges */
.status-badge {
    padding: 6px 12px;
    border-radius: 20px;
    font-size: 12px;
    font-weight: 600;
    text-transform: uppercase;
}

.status-active {
    background: #d4edda;
    color: #155724;
}

.status-pending {
    background: #fff3cd;
    color: #856404;
}

.status-suspended {
    background: #f8d7da;
    color: #721c24;
}
```

### Responsive Adjustments

```css
/* Mobile Adjustments */
@media (max-width: 768px) {
    .btn {
        width: 100%;
        margin-bottom: 10px;
    }
    
    .table-responsive {
        border: none;
    }
    
    .product-card {
        margin-bottom: 20px;
    }
    
    .navbar-collapse {
        max-height: 100vh;
        overflow-y: auto;
    }
}

/* Tablet Adjustments */
@media (min-width: 769px) and (max-width: 1024px) {
    .col-md-4 {
        width: 50%;
    }
}
```

### Animation Effects

```css
/* Fade In Animation */
@keyframes fadeIn {
    from {
        opacity: 0;
        transform: translateY(20px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.fade-in {
    animation: fadeIn 0.5s ease forwards;
}

/* Pulse Animation */
@keyframes pulse {
    0% {
        box-shadow: 0 0 0 0 rgba(255, 102, 0, 0.4);
    }
    70% {
        box-shadow: 0 0 0 10px rgba(255, 102, 0, 0);
    }
    100% {
        box-shadow: 0 0 0 0 rgba(255, 102, 0, 0);
    }
}

.pulse {
    animation: pulse 2s infinite;
}

/* Loading Spinner */
.spinner {
    width: 40px;
    height: 40px;
    border: 4px solid #f3f3f3;
    border-top: 4px solid #ff6600;
    border-radius: 50%;
    animation: spin 1s linear infinite;
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}
```

### WHMCS-Specific Classes

```css
/* Client Area */
#main-body {
    background-color: #f5f5f5;
}

.client-area {
    padding: 40px 0;
}

/* Sidebar */
.sidebar {
    background: #fff;
    padding: 20px;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.05);
}

.sidebar .nav > li > a {
    padding: 12px 15px;
    border-radius: 4px;
}

.sidebar .nav > li > a:hover {
    background: #f0f0f0;
}

/* Invoice */
.invoice-details {
    background: #fff;
    padding: 30px;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.05);
}

/* Service Status */
.service-status-indicator {
    display: inline-block;
    width: 10px;
    height: 10px;
    border-radius: 50%;
    margin-right: 8px;
}

.service-status-indicator.active {
    background: #4caf50;
}

.service-status-indicator.suspended {
    background: #ff9800;
}

.service-status-indicator.terminated {
    background: #f44336;
}
```

## Best Practices

1. **Use specific selectors** - Avoid !important when possible
2. **Keep styles organized** - Group by component
3. **Use CSS variables** for easy theming
4. **Test responsive** designs on mobile
5. **Minify CSS** for production

## See Also

- [Bootstrap Components](../whmcs-bootstrap-components.md)
- [Font Awesome Icons](../whmcs-fontawesome-icons.md)
- [Template Inheritance](../whmcs-template-inheritance.md)