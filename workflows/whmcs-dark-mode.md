# WHMCS Dark Mode Setup Workflow

## Purpose
Implement dark mode support in WHMCS for improved user experience.

## Prerequisites
- WHMCS installation
- CSS custom properties knowledge
- Template access
- JavaScript knowledge

## Step-by-Step Process

### Step 1: Create Dark Mode CSS Variables

**Create dark-mode.css:**
```css
/* /whmcs/templates/your_template/css/dark-mode.css */

/* Dark Mode Color Palette */
:root,
[data-theme="light"] {
    /* Background colors */
    --color-bg-primary: #ffffff;
    --color-bg-secondary: #f8f9fa;
    --color-bg-tertiary: #e9ecef;
    --color-bg-card: #ffffff;
    --color-bg-hover: #f1f3f4;
    --color-bg-active: #e8eaed;
    
    /* Text colors */
    --color-text-primary: #1a1a2e;
    --color-text-secondary: #4a4a68;
    --color-text-muted: #6c757d;
    --color-text-inverse: #ffffff;
    
    /* Border colors */
    --color-border: #dee2e6;
    --color-border-light: #e9ecef;
    --color-border-dark: #ced4da;
    
    /* Brand colors (adjusted for dark mode) */
    --color-primary: #4361ee;
    --color-primary-light: #738ffe;
    --color-primary-dark: #3a56d4;
    
    /* Status colors (adjusted) */
    --color-success: #10b981;
    --color-warning: #f59e0b;
    --color-error: #ef4444;
    --color-info: #3b82f6;
    
    /* Shadows (lighter for dark mode) */
    --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.2);
    --shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
    --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.25);
    --shadow-lg: 0 10px 15px rgba(0, 0, 0, 0.3);
    
    /* Input backgrounds */
    --color-input-bg: #ffffff;
    --color-input-border: #ced4da;
    --color-input-focus: #4361ee;
}

[data-theme="dark"] {
    /* Background colors */
    --color-bg-primary: #0f172a;
    --color-bg-secondary: #1e293b;
    --color-bg-tertiary: #334155;
    --color-bg-card: #1e293b;
    --color-bg-hover: #334155;
    --color-bg-active: #475569;
    
    /* Text colors */
    --color-text-primary: #f1f5f9;
    --color-text-secondary: #cbd5e1;
    --color-text-muted: #94a3b8;
    --color-text-inverse: #1a1a2e;
    
    /* Border colors */
    --color-border: #475569;
    --color-border-light: #334155;
    --color-border-dark: #64748b;
    
    /* Brand colors (adjusted for dark mode) */
    --color-primary: #60a5fa;
    --color-primary-light: #93c5fd;
    --color-primary-dark: #3b82f6;
    
    /* Status colors (adjusted for dark mode) */
    --color-success: #34d399;
    --color-warning: #fbbf24;
    --color-error: #f87171;
    --color-info: #60a5fa;
    
    /* Shadows (darker for dark mode) */
    --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.4);
    --shadow: 0 2px 4px rgba(0, 0, 0, 0.4);
    --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.5);
    --shadow-lg: 0 10px 15px rgba(0, 0, 0, 0.6);
    
    /* Input backgrounds */
    --color-input-bg: #1e293b;
    --color-input-border: #475569;
    --color-input-focus: #60a5fa;
}

/* System preference support */
@media (prefers-color-scheme: dark) {
    :root:not([data-theme="light"]) {
        /* Same as [data-theme="dark"] */
        --color-bg-primary: #0f172a;
        --color-bg-secondary: #1e293b;
        --color-bg-tertiary: #334155;
        --color-bg-card: #1e293b;
        --color-bg-hover: #334155;
        --color-bg-active: #475569;
        
        --color-text-primary: #f1f5f9;
        --color-text-secondary: #cbd5e1;
        --color-text-muted: #94a3b8;
        --color-text-inverse: #1a1a2e;
        
        --color-border: #475569;
        --color-border-light: #334155;
        --color-border-dark: #64748b;
        
        --color-primary: #60a5fa;
        --color-primary-light: #93c5fd;
        --color-primary-dark: #3b82f6;
        
        --color-success: #34d399;
        --color-warning: #fbbf24;
        --color-error: #f87171;
        --color-info: #60a5fa;
        
        --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.4);
        --shadow: 0 2px 4px rgba(0, 0, 0, 0.4);
        --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.5);
        --shadow-lg: 0 10px 15px rgba(0, 0, 0, 0.6);
        
        --color-input-bg: #1e293b;
        --color-input-border: #475569;
        --color-input-focus: #60a5fa;
    }
}
```

### Step 2: Hook Dark Mode CSS

```php
<?php
/**
 * Add dark mode CSS
 */
use WHMCS\View\Asset;

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $css = '<link rel="stylesheet" href="' . 
           Asset::url('/templates/' . $vars['template'] . '/css/dark-mode.css') . '">';
    
    // Add system preference detection script
    $css .= '<script>
        (function() {
            const theme = localStorage.getItem("whmcs-theme");
            if (theme) {
                document.documentElement.setAttribute("data-theme", theme);
            }
        })();
    </script>';
    
    return $css;
});
```

### Step 3: Apply Variables to Components

```css
/* Apply dark mode variables */
body {
    background-color: var(--color-bg-primary);
    color: var(--color-text-primary);
}

.card, .panel {
    background-color: var(--color-bg-card);
    border-color: var(--color-border);
}

a {
    color: var(--color-primary);
}

a:hover {
    color: var(--color-primary-light);
}

/* Form elements */
input, select, textarea {
    background-color: var(--color-input-bg);
    border-color: var(--color-input-border);
    color: var(--color-text-primary);
}

input:focus, select:focus, textarea:focus {
    border-color: var(--color-input-focus);
    box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.2);
}

/* Tables */
.table {
    background-color: var(--color-bg-card);
    color: var(--color-text-primary);
}

.table-striped tbody tr:nth-of-type(odd) {
    background-color: var(--color-bg-secondary);
}

/* Buttons */
.btn-primary {
    background-color: var(--color-primary);
    border-color: var(--color-primary);
    color: var(--color-text-inverse);
}

.btn-outline-secondary {
    border-color: var(--color-border);
    color: var(--color-text-secondary);
}

/* Navigation */
.navbar, .header {
    background-color: var(--color-bg-card);
    border-bottom-color: var(--color-border);
}

.navbar-light .navbar-nav .nav-link {
    color: var(--color-text-primary);
}

/* Sidebar */
.sidebar {
    background-color: var(--color-bg-secondary);
}

/* Modals */
.modal-content {
    background-color: var(--color-bg-card);
    border-color: var(--color-border);
}

.modal-header {
    border-bottom-color: var(--color-border);
}

.modal-footer {
    border-top-color: var(--color-border);
}

/* Dropdowns */
.dropdown-menu {
    background-color: var(--color-bg-card);
    border-color: var(--color-border);
}

.dropdown-item {
    color: var(--color-text-primary);
}

.dropdown-item:hover {
    background-color: var(--color-bg-hover);
}

/* Alerts */
.alert {
    border-color: transparent;
}

.alert-light {
    background-color: var(--color-bg-secondary);
    color: var(--color-text-primary);
}
```

### Step 4: Create Theme Toggle Component

**Toggle Button:**
```html
<!-- Theme Toggle -->
<button type="button" class="theme-toggle" id="themeToggle" aria-label="Toggle dark mode">
    <span class="theme-toggle-icon sun">☀️</span>
    <span class="theme-toggle-icon moon">🌙</span>
</button>
```

**Toggle CSS:**
```css
.theme-toggle {
    background: none;
    border: none;
    cursor: pointer;
    padding: 0.5rem;
    border-radius: 50%;
    transition: background-color 0.2s;
}

.theme-toggle:hover {
    background-color: var(--color-bg-hover);
}

.theme-toggle-icon {
    font-size: 1.25rem;
    display: none;
}

[data-theme="light"] .theme-toggle-icon.sun {
    display: inline;
}

[data-theme="dark"] .theme-toggle-icon.moon {
    display: inline;
}

/* Or use SVG icons */
.theme-toggle-icon svg {
    width: 20px;
    height: 20px;
}
```

**Toggle JavaScript:**
```javascript
// Theme toggle functionality
document.addEventListener('DOMContentLoaded', function() {
    const toggle = document.getElementById('themeToggle');
    if (!toggle) return;
    
    // Get current theme
    function getTheme() {
        return localStorage.getItem('whmcs-theme') || 
               (window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light');
    }
    
    // Set theme
    function setTheme(theme) {
        document.documentElement.setAttribute('data-theme', theme);
        localStorage.setItem('whmcs-theme', theme);
        
        // Dispatch event for other components
        document.dispatchEvent(new CustomEvent('themeChange', { detail: { theme }}));
    }
    
    // Toggle handler
    toggle.addEventListener('click', function() {
        const current = getTheme();
        setTheme(current === 'dark' ? 'light' : 'dark');
    });
    
    // Listen for system preference changes
    window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', function(e) {
        if (!localStorage.getItem('whmcs-theme')) {
            setTheme(e.matches ? 'dark' : 'light');
        }
    });
});
```

### Step 5: Persist Theme Preference

```php
<?php
/**
 * Store theme preference in user profile
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    if ($vars['loggedin'] && isset($_COOKIE['whmcs-theme'])) {
        $theme = $_COOKIE['whmcs-theme'];
        // Could save to user preferences table
    }
    return [];
});
```

### Step 6: WHMCS-Specific Dark Mode Fixes

```css
/* WHMCS Specific Dark Mode */

/* Login page */
.login-container {
    background-color: var(--color-bg-primary);
}

.login-form {
    background-color: var(--color-bg-card);
}

/* Checkout */
.cart-sidebar {
    background-color: var(--color-bg-secondary);
}

.order-summary {
    background-color: var(--color-bg-card);
}

/* Invoices */
.invoice-view {
    background-color: var(--color-bg-primary);
}

/* Knowledgebase */
.article-content {
    background-color: var(--color-bg-card);
}

/* Support tickets */
.ticket-messages {
    background-color: var(--color-bg-primary);
}

.ticket-message {
    background-color: var(--color-bg-card);
}

/* Domain lookup */
.domain-checker {
    background-color: var(--color-bg-card);
}

/* Product pages */
.pricing-table {
    background-color: var(--color-bg-card);
}

.pricing-plan.featured {
    border-color: var(--color-primary);
}

/* Network issues */
.network-status {
    background-color: var(--color-bg-card);
}

/* Affiliates */
.affiliate-dashboard {
    background-color: var(--color-bg-primary);
}
```

### Step 7: Image Adjustments

```css
/* Images in dark mode */
@media (prefers-color-scheme: dark) {
    img[src*=".png"]:not([src*="logo"]) {
        filter: brightness(0.9) contrast(1.1);
    }
    
    /* White/light logos for dark mode */
    img.logo-light {
        display: none;
    }
    
    img.logo-dark {
        display: block;
    }
}

[data-theme="dark"] img.logo-light {
    display: none;
}

[data-theme="dark"] img.logo-dark {
    display: block;
}

/* Icon adjustments */
[data-theme="dark"] .text-muted {
    color: var(--color-text-muted) !important;
}
```

### Step 8: Smooth Transitions

```css
/* Theme transition */
body,
body * {
    transition: 
        background-color 0.3s ease,
        color 0.3s ease,
        border-color 0.3s ease;
}

/* Disable transitions on initial load */
body:not(.theme-ready) *,
body:not(.theme-ready) {
    transition: none !important;
}
```

```javascript
// Enable transitions after page load
window.addEventListener('load', function() {
    document.body.classList.add('theme-ready');
});
```

### Step 9: Favicon for Dark Mode

```html
<!-- In head -->
<link rel="icon" href="/favicon-light.svg" media="(prefers-color-scheme: light)">
<link rel="icon" href="/favicon-dark.svg" media="(prefers-color-scheme: dark)">
```

## Best Practices
- Use CSS custom properties for colors
- Test both light and dark modes thoroughly
- Consider contrast ratios (WCAG compliance)
- Provide user toggle option
- Respect system preference as default
- Store preference in localStorage and user profile
- Add smooth transitions between themes
- Update favicon for dark mode
- Test with actual screenshots
- Consider print styles (usually light)
