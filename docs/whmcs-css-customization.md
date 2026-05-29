# WHMCS CSS Customization

## Overview

Customize WHMCS appearance using CSS while maintaining compatibility with updates.

## Adding Custom CSS

### Via Admin Area

Go to **Configuration > System Settings > Custom CSS** and add your styles.

### Via Module/Addon

```php
<?php
function yourmodule_output(array $vars): void
{
    // Add CSS file
    echo '<link rel="stylesheet" href="' . $vars['modulename'] . '/assets/css/custom.css">';
}
```

### Via Hooks

```php
<?php
add_hook('AdminAreaHeaderOutput', 1, function($vars) {
    return '<style>
        .custom-header {
            background: #1a1a2e;
            color: white;
        }
    </style>';
});

add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    return '<link rel="stylesheet" href="/modules/addons/yourmodule/assets/css/style.css">';
});
```

## Common Customizations

### Brand Colors

```css
/* Primary colors */
:root {
    --brand-primary: #0066cc;
    --brand-secondary: #6c757d;
    --brand-success: #28a745;
    --brand-danger: #dc3545;
    --brand-warning: #ffc107;
}

/* Override Bootstrap colors */
.btn-primary {
    background-color: var(--brand-primary);
    border-color: var(--brand-primary);
}

.btn-primary:hover {
    background-color: #0052a3;
    border-color: #0052a3;
}

.bg-primary {
    background-color: var(--brand-primary) !important;
}

.text-primary {
    color: var(--brand-primary) !important;
}
```

### Client Area Styling

```css
/* Header customization */
#header {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

#header .navbar-brand {
    font-weight: bold;
    font-size: 1.5rem;
}

/* Navigation */
.nav.navbar-nav li a {
    transition: color 0.3s ease;
}

.nav.navbar-nav li a:hover {
    color: var(--brand-primary);
}

/* Cards */
.card {
    border: none;
    border-radius: 12px;
    box-shadow: 0 4px 6px rgba(0,0,0,0.1);
    transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.card:hover {
    transform: translateY(-5px);
    box-shadow: 0 8px 15px rgba(0,0,0,0.1);
}

/* Buttons */
.btn {
    border-radius: 8px;
    padding: 0.5rem 1.5rem;
    font-weight: 500;
    transition: all 0.3s ease;
}

.btn-primary {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    border: none;
}

.btn-primary:hover {
    transform: scale(1.05);
    box-shadow: 0 4px 15px rgba(102, 126, 234, 0.4);
}
```

### Form Styling

```css
/* Form inputs */
.form-control {
    border: 2px solid #e9ecef;
    border-radius: 8px;
    padding: 0.75rem 1rem;
    transition: border-color 0.3s ease, box-shadow 0.3s ease;
}

.form-control:focus {
    border-color: var(--brand-primary);
    box-shadow: 0 0 0 3px rgba(0, 102, 204, 0.1);
}

/* Labels */
.form-label {
    font-weight: 600;
    color: #495057;
    margin-bottom: 0.5rem;
}

/* Validation states */
.is-invalid {
    border-color: var(--brand-danger);
}

.invalid-feedback {
    color: var(--brand-danger);
    font-size: 0.875rem;
}
```

### Table Styling

```css
/* Modern tables */
.table {
    border-collapse: separate;
    border-spacing: 0;
}

.table thead th {
    background: #f8f9fa;
    border-bottom: 2px solid #dee2e6;
    font-weight: 600;
    text-transform: uppercase;
    font-size: 0.75rem;
    letter-spacing: 0.5px;
}

.table tbody tr {
    transition: background-color 0.2s ease;
}

.table tbody tr:hover {
    background-color: #f8f9fa;
}

.table td {
    vertical-align: middle;
    padding: 1rem;
}

/* Striped rows */
.table-striped tbody tr:nth-of-type(odd) {
    background-color: rgba(0, 102, 204, 0.03);
}
```

### Responsive Design

```css
/* Mobile-first responsive */
@media (max-width: 768px) {
    .table-responsive {
        border-radius: 8px;
        overflow: hidden;
    }
    
    .card {
        margin-bottom: 1rem;
    }
    
    .btn {
        width: 100%;
        margin-bottom: 0.5rem;
    }
}

@media (min-width: 769px) {
    .btn {
        width: auto;
    }
}
```

### Dark Mode

```css
@media (prefers-color-scheme: dark) {
    body {
        background-color: #1a1a2e;
        color: #e0e0e0;
    }
    
    .card {
        background-color: #16213e;
        color: #e0e0e0;
    }
    
    .table {
        color: #e0e0e0;
    }
    
    .table thead th {
        background-color: #0f3460;
        color: #ffffff;
    }
    
    .form-control {
        background-color: #1a1a2e;
        border-color: #2d3748;
        color: #e0e0e0;
    }
}
```

## CSS Architecture

### BEM Methodology

```css
/* Block */
.invoice-card {
    padding: 1.5rem;
    border-radius: 8px;
    background: white;
}

/* Element */
.invoice-card__header {
    border-bottom: 1px solid #eee;
    padding-bottom: 1rem;
    margin-bottom: 1rem;
}

.invoice-card__total {
    font-size: 1.5rem;
    font-weight: bold;
    color: var(--brand-primary);
}

/* Modifier */
.invoice-card--overdue {
    border-left: 4px solid var(--brand-danger);
}

.invoice-card--paid {
    border-left: 4px solid var(--brand-success);
}
```

### CSS Variables

```css
:root {
    /* Typography */
    --font-family-base: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
    --font-size-base: 1rem;
    --line-height-base: 1.5;
    
    /* Spacing */
    --spacing-xs: 0.25rem;
    --spacing-sm: 0.5rem;
    --spacing-md: 1rem;
    --spacing-lg: 1.5rem;
    --spacing-xl: 2rem;
    
    /* Colors */
    --color-primary: #0066cc;
    --color-secondary: #6c757d;
    --color-success: #28a745;
    --color-danger: #dc3545;
    --color-warning: #ffc107;
    
    /* Shadows */
    --shadow-sm: 0 1px 2px rgba(0,0,0,0.05);
    --shadow-md: 0 4px 6px rgba(0,0,0,0.1);
    --shadow-lg: 0 10px 15px rgba(0,0,0,0.1);
    
    /* Border radius */
    --radius-sm: 4px;
    --radius-md: 8px;
    --radius-lg: 12px;
    --radius-full: 9999px;
}
```

## Best Practices

1. **Use CSS variables** - Easier to maintain and theme
2. **Follow BEM** - Structured naming convention
3. **Use !important sparingly** - Only for overrides
4. **Test responsive** - Mobile and desktop
5. **Support dark mode** - Consider color scheme preferences

## Related Documentation

- [WHMCS Smarty Templates](/docs/whmcs-smarty-templates.md)
- [WHMCS Responsive Design](/docs/whmcs-responsive-design.md)