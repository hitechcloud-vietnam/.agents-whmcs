# WHMCS Client Area Theme Development

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-template-styling`, `whmcs-clientarea-builder`, `whmcs-admin-ui-builder`

---

## Overview

Customizing the WHMCS client area allows you to create a branded experience for your customers. This guide covers theme structure, template customization, asset management, and best practices for client area theming.

---

## Theme Structure

### Directory Structure

```
templates/
└── your_theme/
    ├── theme.yaml              # Theme manifest
    ├── master.blade.php         # Master template
    ├── home.blade.php           # Homepage
    ├── login.blade.php          # Login page
    ├── register.blade.php       # Registration
    ├── client/
    │   ├── index.blade.php      # Dashboard
    │   ├── services.blade.php   # Services list
    │   ├── service-details.blade.php
    │   ├── domains.blade.php    # Domains list
    │   ├── invoices.blade.php   # Invoices list
    │   ├── invoice-view.blade.php
    │   ├── tickets.blade.php    # Support tickets
    │   ├── cart.blade.php       # Shopping cart
    │   └── account.blade.php    # Account settings
    ├── includes/
    │   ├── header.blade.php
    │   ├── footer.blade.php
    │   ├── sidebar.blade.php
    │   └── navbar.blade.php
    └── assets/
        ├── css/
        │   ├── app.css
        │   └── custom.css
        ├── js/
        │   ├── app.js
        │   └── custom.js
        └── images/
            ├── logo.png
            └── hero-bg.jpg
```

### Theme Manifest

```yaml
# theme.yaml
name: "Your Custom Theme"
version: "1.0.0"
author: "Your Name"
description: "Custom WHMCS client area theme"
requires: "8.0"

# Template inheritance
extends: "six"

# Layout variations
layouts:
  default:
    master: "master.blade.php"
  minimal:
    master: "minimal.blade.php"

# Asset compilation
assets:
  css:
    - "assets/css/app.css"
    - "assets/css/custom.css"
  js:
    - "assets/js/app.js"
    - "assets/js/custom.js"

# Theme options
options:
  primary_color:
    type: "color"
    default: "#0073aa"
  logo:
    type: "image"
    default: "assets/images/logo.png"
```

---

## Master Template

### Base Template Structure

```php
{{-- master.blade.php --}}
<!DOCTYPE html>
<html lang="{{ Lang::getLocale() }}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="csrf-token" content="{{ csrf_token() }}">
    <meta name="generator" content="WHMCS">

    <title>@yield('title', $pagTitle ?? 'WHMCS')</title>

    {{-- Favicon --}}
    <link rel="icon" type="image/x-icon"
          href="{{ $baseUrl }}assets/img/favicon.ico">

    {{-- Google Fonts --}}
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=
        Inter:wght@400;500;600;700&display=swap" rel="stylesheet">

    {{-- Theme CSS --}}
    @stack('head_css')
    <link rel="stylesheet" href="{{ $baseUrl }}templates/{{ $template }}/assets/css/app.css">

    {{-- Page-specific head --}}
    @yield('head')

    {{-- Custom head output from hooks --}}
    @hook('ClientAreaHeadOutput')
</head>
<body class="@yield('body_class', '')">

    {{-- Skip to content link --}}
    <a href="#main-content" class="skip-link">
        Skip to main content
    </a>

    {{-- Header --}}
    <header class="site-header">
        @include('templates.' . $template . '.includes.header')
    </header>

    {{-- Flash messages --}}
    @if(Session::has('success'))
        <div class="alert alert-success">
            {{ Session::get('success') }}
        </div>
    @endif

    @if(Session::has('error'))
        <div class="alert alert-danger">
            {{ Session::get('error') }}
        </div>
    @endif

    {{-- Main content --}}
    <main id="main-content" class="main-content">
        <div class="container">
            @yield('content')
        </div>
    </main>

    {{-- Footer --}}
    <footer class="site-footer">
        @include('templates.' . $template . '.includes.footer')
    </footer>

    {{-- Theme JS --}}
    @stack('footer_js')
    <script src="{{ $baseUrl }}templates/{{ $template }}/assets/js/app.js"></script>

    {{-- Custom footer output from hooks --}}
    @hook('ClientAreaFooterOutput')
</body>
</html>
```

---

## Template Inheritance

### Extending Templates

```php
{{-- Extending the master template --}}
@extends('templates.' . $template . '.master')

@section('title', 'My Services')

@section('content')
<div class="page-header">
    <h1>{{ $pagTitle }}</h1>
</div>

<div class="services-grid">
    @forelse($services as $service)
        <div class="service-card">
            <h3>{{ $service->domain }}</h3>
            <span class="badge badge-{{ $service->status }}">
                {{ ucfirst($service->status) }}
            </span>
            <a href="{{ $service->detailsUrl }}" class="btn btn-primary">
                Manage
            </a>
        </div>
    @empty
        <div class="no-services">
            <p>No active services found.</p>
        </div>
    @endforelse
</div>

{{ $services->links() }}
@endsection

@push('head_css')
<style>
.services-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 1.5rem;
    margin: 2rem 0;
}
</style>
@endpush
```

### Component Sections

```php
{{-- Client dashboard sidebar --}}
@extends('templates.' . $template . '.master')

@section('sidebar')
<aside class="client-sidebar">
    <nav class="sidebar-nav">
        <a href="{{ route('client.overview') }}" class="nav-item">
            <i class="fas fa-home"></i>
            Overview
        </a>
        <a href="{{ route('client.services') }}" class="nav-item">
            <i class="fas fa-server"></i>
            Services
        </a>
        <a href="{{ route('client.domains') }}" class="nav-item">
            <i class="fas fa-globe"></i>
            Domains
        </a>
        <a href="{{ route('client.invoices') }}" class="nav-item">
            <i class="fas fa-file-invoice"></i>
            Invoices
        </a>
    </nav>
</aside>
@endsection

@section('content')
<div class="dashboard-content">
    @yield('dashboard_widgets')
</div>
@endsection
```

---

## Client Area Variables

### Available Smarty Variables

```php
<?php
/**
 * Common variables available in client area templates
 */

// User information
$client->id                    // Client ID
$client->email                 // Email address
$client->firstname             // First name
$client->lastname              // Last name
$client->companyname           // Company name
$client->phonenumber           // Phone number
$client->datecreated           // Registration date

// Services
$services                      // Array of service objects
$service->id                   // Service ID
$service->domain               // Domain name
$service->username             // Service username
$service->status               // Status (Active, Suspended, etc.)
$service->recurringamount      // Recurring amount
$service->nextduedate         // Next due date

// Invoices
$invoices                      // Array of invoice objects
$invoice->id                   // Invoice number
$invoice->total                // Total amount
$invoice->status               // Status (Paid, Unpaid, etc.)
$invoice->duedate             // Due date

// Domains
$domains                       // Array of domain objects
$domain->id                    // Domain ID
$domain->domain                // Domain name
$domain->registrationdate     // Registration date
$domain->nextduedate          // Next due date
$domain->status               // Domain status

// Support tickets
$tickets                       // Array of ticket objects
$ticket->id                    // Ticket ID
$ticket->subject              // Subject
$ticket->status               // Status
$ticket->priority             // Priority

// System variables
$pagTitle                      // Page title
$template                      // Current template name
$baseUrl                       // WHMCS base URL
$systemUrl                    // System URL
$lang                          // Current language
$currency                      // Currency object
```

---

## Styling

### CSS Variables

```css
/* assets/css/app.css */

:root {
    /* Primary colors */
    --primary-color: #0073aa;
    --primary-hover: #005a87;
    --primary-light: #e7f1f7;

    /* Secondary colors */
    --secondary-color: #505050;
    --secondary-hover: #353535;

    /* Status colors */
    --success-color: #28a745;
    --warning-color: #ffc107;
    --danger-color: #dc3545;
    --info-color: #17a2b8;

    /* Neutral colors */
    --white: #ffffff;
    --gray-100: #f8f9fa;
    --gray-200: #e9ecef;
    --gray-300: #dee2e6;
    --gray-400: #ced4da;
    --gray-500: #adb5bd;
    --gray-600: #6c757d;
    --gray-700: #495057;
    --gray-800: #333333;
    --gray-900: #212529;

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
    --spacing-2xl: 3rem;

    /* Border radius */
    --border-radius: 0.375rem;
    --border-radius-lg: 0.5rem;

    /* Shadows */
    --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.05);
    --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.1);
    --shadow-lg: 0 10px 15px rgba(0, 0, 0, 0.1);

    /* Transitions */
    --transition-speed: 0.2s;
}

/* Base styles */
body {
    font-family: var(--font-family-base);
    font-size: var(--font-size-base);
    line-height: var(--line-height-base);
    color: var(--gray-800);
    background-color: var(--gray-100);
}
```

### Component Styles

```css
/* Button styles */
.btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    padding: var(--spacing-sm) var(--spacing-lg);
    font-size: 0.875rem;
    font-weight: 500;
    border-radius: var(--border-radius);
    transition: all var(--transition-speed) ease;
    text-decoration: none;
    cursor: pointer;
    border: none;
}

.btn-primary {
    background-color: var(--primary-color);
    color: var(--white);
}

.btn-primary:hover {
    background-color: var(--primary-hover);
}

.btn-secondary {
    background-color: var(--gray-200);
    color: var(--gray-800);
}

/* Card styles */
.card {
    background: var(--white);
    border-radius: var(--border-radius-lg);
    box-shadow: var(--shadow-sm);
    padding: var(--spacing-lg);
}

.card-header {
    border-bottom: 1px solid var(--gray-200);
    padding-bottom: var(--spacing-md);
    margin-bottom: var(--spacing-md);
}

/* Status badges */
.badge {
    display: inline-block;
    padding: var(--spacing-xs) var(--spacing-sm);
    font-size: 0.75rem;
    font-weight: 600;
    border-radius: var(--border-radius);
    text-transform: uppercase;
}

.badge-active {
    background-color: #d4edda;
    color: #155724;
}

.badge-suspended {
    background-color: #fff3cd;
    color: #856404;
}

.badge-terminated {
    background-color: #f8d7da;
    color: #721c24;
}
```

---

## JavaScript Integration

### Asset Loading

```php
<?php
/**
 * Enqueue scripts and styles in hooks
 */

// hooks.php
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    return '<link rel="stylesheet"
          href="https://cdn.example.com/theme.css">';
});

add_hook('ClientAreaFooterOutput', 1, function($vars) {
    return '<script src="https://cdn.example.com/theme.js"></script>';
});
```

### Blade Component

```php
{{-- In your template --}}
@push('footer_js')
<script>
document.addEventListener('DOMContentLoaded', function() {
    // Initialize components
    initTooltips();
    initModals();
    initForms();

    // Theme-specific functionality
    Theme.init({
        apiUrl: '{{ $baseUrl }}api/v1/',
        csrfToken: '{{ csrf_token() }}',
    });
});
</script>
@endpush
```

### JavaScript Module Pattern

```javascript
// assets/js/theme.js
const Theme = (function() {
    'use strict';

    let config = {};
    let apiClient = null;

    // Initialize theme
    function init(options = {}) {
        config = {
            apiUrl: options.apiUrl || '/api/',
            csrfToken: options.csrfToken || '',
            ...options
        };

        setupAjax();
        setupEventListeners();
    }

    // Setup AJAX defaults
    function setupAjax() {
        $.ajaxSetup({
            headers: {
                'X-CSRF-TOKEN': config.csrfToken,
                'Accept': 'application/json'
            }
        });
    }

    // Event listeners
    function setupEventListeners() {
        // Confirm deletions
        $(document).on('click', '[data-confirm]', function(e) {
            if (!confirm($(this).data('confirm'))) {
                e.preventDefault();
            }
        });

        // Loading state on forms
        $('form[data-loading]').on('submit', function() {
            const $btn = $(this).find('[type="submit"]');
            $btn.prop('disabled', true)
                .data('original-text', $btn.text())
                .text('Loading...');
        });
    }

    // API helper
    async function api(endpoint, method = 'GET', data = null) {
        const options = {
            method: method,
            url: config.apiUrl + endpoint,
            headers: {
                'X-CSRF-TOKEN': config.csrfToken,
            }
        };

        if (data && method !== 'GET') {
            options.data = data;
            options.contentType = false;
            options.processData = false;
        }

        return $.ajax(options);
    }

    // Public API
    return {
        init: init,
        api: api,
        config: config
    };
})();

// Initialize
window.Theme = Theme;
```

---

## Responsive Design

### Grid System

```css
/* Responsive grid */
.container {
    width: 100%;
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 var(--spacing-md);
}

.row {
    display: flex;
    flex-wrap: wrap;
    margin: 0 calc(var(--spacing-md) * -0.5);
}

.col {
    flex: 1;
    padding: 0 calc(var(--spacing-md) * 0.5);
}

/* Column sizes */
.col-12 { flex: 0 0 100%; max-width: 100%; }
.col-6 { flex: 0 0 50%; max-width: 50%; }
.col-4 { flex: 0 0 33.333%; max-width: 33.333%; }
.col-3 { flex: 0 0 25%; max-width: 25%; }

/* Responsive breakpoints */
@media (max-width: 992px) {
    .col-lg-6 { flex: 0 0 50%; max-width: 50%; }
    .col-lg-12 { flex: 0 0 100%; max-width: 100%; }
}

@media (max-width: 768px) {
    .col-md-12 { flex: 0 0 100%; max-width: 100%; }
    .sidebar { display: none; }
}

@media (max-width: 576px) {
    .col-sm-12 { flex: 0 0 100%; max-width: 100%; }
}
```

### Mobile Navigation

```php
{{-- Mobile navigation toggle --}}
<div class="mobile-nav">
    <button class="mobile-nav-toggle" data-toggle="mobile-menu">
        <span class="hamburger"></span>
    </button>

    <nav class="mobile-menu" id="mobile-menu">
        <a href="{{ route('client.services') }}">
            Services
        </a>
        <a href="{{ route('client.domains') }}">
            Domains
        </a>
        <a href="{{ route('client.invoices') }}">
            Invoices
        </a>
    </nav>
</div>
```

---

## Template Hooks

### Available Client Area Hooks

```php
<?php
// hooks.php

// Add content to client area header
add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    return '<script>
        window.WHMCS_CLIENTAREA = {
            userId: ' . ($client->id ?? 0) . ',
            isLoggedIn: ' . ($client ? 'true' : 'false') . '
        };
    </script>';
});

// Add sidebar widgets
add_hook('ClientAreaSidebars', 1, function($vars) {
    return [
        'template' => 'widgets/quick-actions',
        'vars' => [
            'links' => [
                ['url' => '/cart.php', 'label' => 'Order New'],
                ['url' => '/supporttickets.php', 'label' => 'New Ticket'],
            ]
        ]
    ];
});

// Add custom template variables
add_hook('PredefinedSmartyVariables', 1, function($vars) {
    return [
        'theme_color' => '#0073aa',
        'show_promotions' => true,
    ];
});
```

---

## Best Practices

1. **Always extend the parent theme** - Use `six` as base for compatibility
2. **Use CSS variables** - For easy customization and child themes
3. **Minimize custom JavaScript** - Use jQuery and existing WHMCS libraries
4. **Test responsive design** - Ensure mobile compatibility
5. **Follow accessibility guidelines** - WCAG 2.1 AA compliance
6. **Use hooks for customization** - Avoid modifying template files directly
7. **Keep assets optimized** - Minify CSS/JS and compress images

---

## Related Documentation

- [Client Area Theming](client-area-theming.md)
- [Smarty Template Reference](smarty-template-reference.md)
- [Hooks Reference](hooks-reference.md)
- [Admin Area Customization](admin-area-customization.md)
