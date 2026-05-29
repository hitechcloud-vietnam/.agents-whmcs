# WHMCS Color Variables Workflow

## Purpose
Implement and manage color system using CSS custom properties in WHMCS.

## Prerequisites
- WHMCS installation
- CSS knowledge
- Access to template files

## Step-by-Step Process

### Step 1: Create Color Variables System

**Create variables.css:**
```css
/* /whmcs/templates/your_template/css/variables.css */

/* Brand Colors */
:root {
    /* Primary Palette */
    --color-primary-50: #e3f2fd;
    --color-primary-100: #bbdefb;
    --color-primary-200: #90caf9;
    --color-primary-300: #64b5f6;
    --color-primary-400: #42a5f5;
    --color-primary-500: #2196f3;
    --color-primary-600: #1e88e5;
    --color-primary-700: #1976d2;
    --color-primary-800: #1565c0;
    --color-primary-900: #0d47a1;
    
    /* Secondary Palette */
    --color-secondary-50: #f3e5f5;
    --color-secondary-100: #e1bee7;
    --color-secondary-200: #ce93d8;
    --color-secondary-300: #ba68c8;
    --color-secondary-400: #ab47bc;
    --color-secondary-500: #9c27b0;
    --color-secondary-600: #8e24aa;
    --color-secondary-700: #7b1fa2;
    --color-secondary-800: #6a1b9a;
    --color-secondary-900: #4a148c;
    
    /* Neutral Palette */
    --color-gray-50: #fafafa;
    --color-gray-100: #f5f5f5;
    --color-gray-200: #eeeeee;
    --color-gray-300: #e0e0e0;
    --color-gray-400: #bdbdbd;
    --color-gray-500: #9e9e9e;
    --color-gray-600: #757575;
    --color-gray-700: #616161;
    --color-gray-800: #424242;
    --color-gray-900: #212121;
    
    /* Semantic Colors */
    --color-success: #10b981;
    --color-warning: #f59e0b;
    --color-error: #ef4444;
    --color-info: #3b82f6;
    
    /* Current Theme Primary (Reference) */
    --color-primary: var(--color-primary-500);
    --color-secondary: var(--color-secondary-500);
    --color-background: #ffffff;
    --color-surface: #ffffff;
    --color-text: var(--color-gray-900);
    --color-text-muted: var(--color-gray-600);
    --color-border: var(--color-gray-300);
    
    /* Opacity Levels */
    --opacity-disabled: 0.5;
    --opacity-hover: 0.8;
    
    /* Shadows */
    --shadow-sm: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
    --shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.1), 0 1px 2px 0 rgba(0, 0, 0, 0.06);
    --shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
    --shadow-lg: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
    --shadow-xl: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
}
```

### Step 2: Hook Variables Into Template

```php
<?php
/**
 * Add color variables CSS
 */
use WHMCS\View\Asset;

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $css = '<link rel="stylesheet" href="' . 
           Asset::url('/templates/' . $vars['template'] . '/css/variables.css') . '">';
    return $css;
});
```

### Step 3: Dark Mode Color Variables

**Extend variables.css:**
```css
/* Dark Mode */
@media (prefers-color-scheme: dark) {
    :root {
        --color-background: #0f172a;
        --color-surface: #1e293b;
        --color-text: #f1f5f9;
        --color-text-muted: #94a3b8;
        --color-border: #334155;
        
        /* Adjusted brand colors for dark mode */
        --color-primary: var(--color-primary-400);
        --color-secondary: var(--color-secondary-400);
    }
}

/* Manual Dark Mode Toggle */
[data-theme="dark"] {
    --color-background: #0f172a;
    --color-surface: #1e293b;
    --color-text: #f1f5f9;
    --color-text-muted: #94a3b8;
    --color-border: #334155;
    --color-primary: var(--color-primary-400);
    --color-secondary: var(--color-secondary-400);
}

[data-theme="light"] {
    --color-background: #ffffff;
    --color-surface: #ffffff;
    --color-text: var(--color-gray-900);
    --color-text-muted: var(--color-gray-600);
    --color-border: var(--color-gray-300);
    --color-primary: var(--color-primary-500);
    --color-secondary: var(--color-secondary-500);
}
```

### Step 4: Apply Colors to Components

```css
/* Base Elements */
body {
    background-color: var(--color-background);
    color: var(--color-text);
}

/* Cards */
.card {
    background-color: var(--color-surface);
    border: 1px solid var(--color-border);
    box-shadow: var(--shadow);
}

/* Buttons */
.btn-primary {
    background-color: var(--color-primary);
    color: white;
    border: none;
}

.btn-primary:hover {
    background-color: var(--color-primary-600);
}

/* Links */
a {
    color: var(--color-primary);
}

a:hover {
    color: var(--color-primary-700);
}

/* Forms */
input, select, textarea {
    background-color: var(--color-surface);
    border-color: var(--color-border);
    color: var(--color-text);
}

input:focus, select:focus, textarea:focus {
    border-color: var(--color-primary);
    box-shadow: 0 0 0 3px rgba(33, 150, 243, 0.2);
}

/* Status Colors */
.status-active { color: var(--color-success); }
.status-pending { color: var(--color-warning); }
.status-suspended { color: var(--color-error); }
.status-expired { color: var(--color-text-muted); }
```

### Step 5: Color Utility Classes

```css
/* Text Colors */
.text-primary { color: var(--color-primary) !important; }
.text-secondary { color: var(--color-secondary) !important; }
.text-success { color: var(--color-success) !important; }
.text-warning { color: var(--color-warning) !important; }
.text-error { color: var(--color-error) !important; }
.text-info { color: var(--color-info) !important; }
.text-muted { color: var(--color-text-muted) !important; }

/* Background Colors */
.bg-primary { background-color: var(--color-primary) !important; }
.bg-secondary { background-color: var(--color-secondary) !important; }
.bg-success { background-color: var(--color-success) !important; }
.bg-warning { background-color: var(--color-warning) !important; }
.bg-error { background-color: var(--color-error) !important; }
.bg-info { background-color: var(--color-info) !important; }

/* Border Colors */
.border-primary { border-color: var(--color-primary) !important; }
.border-secondary { border-color: var(--color-secondary) !important; }
.border-success { border-color: var(--color-success) !important; }
.border-warning { border-color: var(--color-warning) !important; }
.border-error { border-color: var(--color-error) !important; }
```

### Step 6: WHMCS-Specific Colors

```css
/* WHMCS Component Overrides */

/* Header */
#header, .header {
    background-color: var(--color-primary);
    border-bottom: 1px solid var(--color-primary-700);
}

/* Navigation */
.navbar-default {
    background-color: var(--color-surface);
}

.navbar-default .navbar-nav > li > a {
    color: var(--color-text);
}

.navbar-default .navbar-nav > li > a:hover,
.navbar-default .navbar-nav > li > a:focus {
    color: var(--color-primary);
}

/* Panels */
.panel {
    background-color: var(--color-surface);
    border-color: var(--color-border);
}

.panel-heading {
    background-color: var(--color-surface);
    border-bottom-color: var(--color-border);
}

/* Alerts */
.alert-success {
    background-color: #d1fae5;
    border-color: var(--color-success);
    color: #065f46;
}

.alert-warning {
    background-color: #fef3c7;
    border-color: var(--color-warning);
    color: #92400e;
}

.alert-danger {
    background-color: #fee2e2;
    border-color: var(--color-error);
    color: #991b1b;
}

/* Tables */
.table {
    color: var(--color-text);
}

.table-striped > tbody > tr:nth-of-type(odd) {
    background-color: rgba(0, 0, 0, 0.02);
}

.table-hover > tbody > tr:hover {
    background-color: rgba(0, 0, 0, 0.04);
}
```

### Step 7: Theme Switcher Hook

```php
<?php
/**
 * Theme color switcher for clients
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $theme = $_COOKIE['client_theme'] ?? 'light';
    return ['client_theme' => $theme];
});
```

**JavaScript Toggle:**
```javascript
function toggleTheme() {
    const current = document.documentElement.getAttribute('data-theme') || 'light';
    const next = current === 'light' ? 'dark' : 'light';
    
    document.documentElement.setAttribute('data-theme', next);
    document.cookie = `client_theme=${next}; path=/; max-age=${60*60*24*365}`;
}
```

### Step 8: Brand Color Generator

**PHP Helper Function:**
```php
<?php
/**
 * Generate color shades from base color
 */
function generateColorShades($hexColor) {
    $shades = [];
    $rgb = hexToRgb($hexColor);
    
    $steps = [50, 100, 200, 300, 400, 500, 600, 700, 800, 900];
    $percentages = [0.95, 0.9, 0.75, 0.6, 0.3, 0, -0.25, -0.5, -0.7, -0.85];
    
    foreach ($steps as $index => $step) {
        $percentage = $percentages[$index];
        
        if ($percentage > 0) {
            // Lighten
            $r = min(255, $rgb['r'] + (255 - $rgb['r']) * $percentage);
            $g = min(255, $rgb['g'] + (255 - $rgb['g']) * $percentage);
            $b = min(255, $rgb['b'] + (255 - $rgb['b']) * $percentage);
        } else {
            // Darken
            $r = max(0, $rgb['r'] + $rgb['r'] * $percentage);
            $g = max(0, $rgb['g'] + $rgb['g'] * $percentage);
            $b = max(0, $rgb['b'] + $rgb['b'] * $percentage);
        }
        
        $shades[$step] = rgbToHex($r, $g, $b);
    }
    
    return $shades;
}

function hexToRgb($hex) {
    $hex = str_replace('#', '', $hex);
    return [
        'r' => hexdec(substr($hex, 0, 2)),
        'g' => hexdec(substr($hex, 2, 2)),
        'b' => hexdec(substr($hex, 4, 2))
    ];
}

function rgbToHex($r, $g, $b) {
    return '#' . dechex($r) . dechex($g) . dechex($b);
}
```

## Best Practices
- Define all colors as CSS custom properties
- Use semantic names for common patterns
- Provide adequate contrast ratios (WCAG AA)
- Support both light and dark modes
- Document color usage guidelines
- Use color palette generators for consistency
- Test colors on various displays
- Consider colorblind-friendly alternatives
- Keep brand colors consistent across platforms
