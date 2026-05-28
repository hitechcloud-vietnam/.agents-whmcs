# WHMCS Client Area Theming

**Version:** 8.x
**Updated:** 2026-05-28
**Related Skills:** `admin-area-customization`, `smarty-template-reference`, `caching-strategies`

## Overview

The WHMCS client area (portal) theming system uses Smarty templates, CSS, and JavaScript. This guide covers creating custom themes, modifying templates, and implementing responsive designs.

## Theme Directory Structure

```
whmcs/
├── templates/
│   ├── your-theme/
│   │   ├── assets/
│   │   │   ├── css/
│   │   │   │   ├── style.css
│   │   │   │   ├── responsive.css
│   │   │   │   └── components.css
│   │   │   ├── js/
│   │   │   │   ├── main.js
│   │   │   │   └── custom.js
│   │   │   ├── images/
│   │   │   └── fonts/
│   │   ├── layouts/
│   │   │   ├── default.tpl
│   │   │   ├── full-width.tpl
│   │   │   └── minimal.tpl
│   │   ├── templates/
│   │   │   ├── clientareahome.tpl
│   │   │   ├── clientareadomaindetails.tpl
│   │   │   └── ...
│   │   ├── includes/
│   │   │   ├── header.tpl
│   │   │   ├── sidebar.tpl
│   │   │   └── footer.tpl
│   │   └── template.php
│   └── default/
└── templates/
```

## Creating a New Theme

### Step 1: Create Theme Directory

```bash
mkdir -p whmcs/templates/your-theme/assets/{css,js,images,fonts}
mkdir -p whmcs/templates/your-theme/{layouts,templates,includes}
```

### Step 2: Create template.php

```php
<?php
/**
 * Theme Configuration
 *
 * @package WHMCS
 * @subpackage Theme
 */

use WHMCS\View\Theme\ThemeConfigurationInterface;

if (!defined('WHMCS')) {
    die('Direct access not permitted');
}

return [
    'name' => 'Your Custom Theme',
    'author' => 'Your Name',
    'description' => 'A modern, responsive WHMCS theme',
    'version' => '1.0.0',

    // Layouts
    'layouts' => [
        'default' => 'layouts/default.tpl',
        'full_width' => 'layouts/full-width.tpl',
        'minimal' => 'layouts/minimal.tpl',
    ],

    // Features
    'supports' => [
        'cart',           // Shopping cart
        'clientArea',     // Client portal
        'signup',         // Signup forms
        'general',        // General pages
        'downloads',      // Downloads section
        'knowledgebase',  // KB articles
    ],

    // Hook files to include
    'hookFiles' => [
        'includes/hooks.php',
    ],

    // CSS files to include (loaded automatically)
    'cssFiles' => [
        'assets/css/style.css',
        'assets/css/components.css',
        'assets/css/responsive.css',
    ],

    // JavaScript files to include
    'jsFiles' => [
        'assets/js/main.js',
    ],

    // Template variables
    'variables' => [
        'theme_version' => '1.0.0',
        'enable_animations' => true,
        'primary_color' => '#007bff',
    ],
];
```

## Layout Templates

### Default Layout

```smarty
{* templates/your-theme/layouts/default.tpl *}
<!DOCTYPE html>
<html lang="{$language}" dir="{if $rtl}rtl{else}ltr{/if}">
<head>
    <meta charset="{$charset}">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="description" content="{$company_name} - Client Portal">

    <title>{$pageTitle}</title>

    {* Favicon *}
    <link rel="icon" type="image/x-icon" href="{$base_path}/favicon.ico">

    {* Google Fonts *}
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">

    {* Theme CSS *}
    {foreach $templateCssFiles as $cssFile}
        <link rel="stylesheet" href="{$base_path}/templates/{$template}/{$cssFile}">
    {/foreach}

    {* Page-specific CSS *}
    {$headoutput}
</head>
<body class="{$templatefile} {if $loggedin}logged-in{/if}">

    {* Navigation Header *}
    {include file="$template/includes/header.tpl"}

    {* Main Content *}
    <main class="main-content">
        <div class="container">
            <div class="row">

                {* Sidebar (if applicable) *}
                {if $showSidebar}
                    <aside class="col-md-3 sidebar">
                        {include file="$template/includes/sidebar.tpl"}
                    </aside>
                {/if}

                {* Primary Content *}
                <div class="{if $showSidebar}col-md-9{else}col-md-12{/if} primary-content">
                    {* Breadcrumbs *}
                    {if $breadcrumbs}
                        <nav aria-label="breadcrumb">
                            <ol class="breadcrumb">
                                {foreach $breadcrumbs as $crumb}
                                    <li class="breadcrumb-item {if $crumb@last}active{/if}">
                                        {if !$crumb@last}
                                            <a href="{$crumb.href}">{$crumb.title}</a>
                                        {else}
                                            {$crumb.title}
                                        {/if}
                                    </li>
                                {/foreach}
                            </ol>
                        </nav>
                    {/if}

                    {* Flash Messages *}
                    {include file="$template/includes/alerts.tpl"}

                    {* Main Template Content *}
                    {if $templatefile}
                        {include file="$template/templates/{$templatefile}.tpl"}
                    {/if}
                </div>
            </div>
        </div>
    </main>

    {* Footer *}
    {include file="$template/includes/footer.tpl"}

    {* Theme JavaScript *}
    {foreach $templateJsFiles as $jsFile}
        <script src="{$base_path}/templates/{$template}/{$jsFile}"></script>
    {/foreach}

    {* Page-specific JavaScript *}
    {$footoutput}
</body>
</html>
```

## Header Template

```smarty
{* templates/your-theme/includes/header.tpl *}
<header class="site-header">
    <nav class="navbar navbar-expand-lg navbar-light">
        <div class="container">
            {* Logo *}
            <a class="navbar-brand" href="{$WEB_ROOT}/">
                <img src="{$company_logo_url}" alt="{$company_name}" height="40">
            </a>

            {* Mobile Toggle *}
            <button class="navbar-toggler" type="button" data-toggle="collapse"
                    data-target="#mainNav" aria-controls="mainNav">
                <span class="navbar-toggler-icon"></span>
            </button>

            {* Navigation *}
            <div class="collapse navbar-collapse" id="mainNav">
                <ul class="navbar-nav mr-auto">
                    <li class="nav-item">
                        <a class="nav-link" href="{$WEB_ROOT}/clientarea.php">
                            <i class="fa fa-home"></i> {$LANG.home}
                        </a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{$WEB_ROOT}/clientarea.php?action=services">
                            <i class="fa fa-server"></i> {$LANG.navservices}
                        </a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{$WEB_ROOT}/clientarea.php?action=domains">
                            <i class="fa fa-globe"></i> {$LANG.navdomains}
                        </a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{$WEB_ROOT}/supporttickets.php">
                            <i class="fa fa-life-ring"></i> {$LANG.navtickets}
                        </a>
                    </li>
                </ul>

                {* User Menu (logged in) *}
                {if $loggedin}
                    <ul class="navbar-nav">
                        <li class="nav-item dropdown">
                            <a class="nav-link dropdown-toggle" href="#" data-toggle="dropdown">
                                <i class="fa fa-user-circle"></i>
                                {$client_first_name} {$client_last_name}
                            </a>
                            <div class="dropdown-menu dropdown-menu-right">
                                <a class="dropdown-item" href="{$WEB_ROOT}/clientarea.php?action=account">
                                    <i class="fa fa-cog"></i> {$LANG.accountdetails}
                                </a>
                                <a class="dropdown-item" href="{$WEB_ROOT}/clientarea.php?action=security">
                                    <i class="fa fa-lock"></i> {$LANG.navsecurity}
                                </a>
                                <div class="dropdown-divider"></div>
                                <a class="dropdown-item" href="{$WEB_ROOT}/logout.php">
                                    <i class="fa fa-sign-out"></i> {$LANG.logout}
                                </a>
                            </div>
                        </li>
                    </ul>
                {else}
                    <div class="navbar-nav">
                        <a href="{$WEB_ROOT}/login.php" class="btn btn-outline-primary btn-sm">
                            {$LANG.login}
                        </a>
                        {if $conditonal_allow_signup}
                            <a href="{$WEB_ROOT}/register.php" class="btn btn-primary btn-sm ml-2">
                                {$LANG.register}
                            </a>
                        {/if}
                    </div>
                {/if}
            </div>
        </div>
    </nav>
</header>
```

## Footer Template

```smarty
{* templates/your-theme/includes/footer.tpl *}
<footer class="site-footer">
    <div class="container">
        <div class="row">
            {* Company Info *}
            <div class="col-md-4">
                <h5>{$company_name}</h5>
                <p class="text-muted">{$company_address|nl2br}</p>
            </div>

            {* Quick Links *}
            <div class="col-md-2">
                <h5>{$LANG.navservices}</h5>
                <ul class="list-unstyled">
                    <li><a href="{$WEB_ROOT}/clientarea.php">{$LANG.home}</a></li>
                    <li><a href="{$WEB_ROOT}/clientarea.php?action=services">My Services</a></li>
                    <li><a href="{$WEB_ROOT}/clientarea.php?action=domains">My Domains</a></li>
                </ul>
            </div>

            {* Support *}
            <div class="col-md-2">
                <h5>{$LANG.support}</h5>
                <ul class="list-unstyled">
                    <li><a href="{$WEB_ROOT}/supporttickets.php">Tickets</a></li>
                    <li><a href="{$WEB_ROOT}/knowledgebase.php">Knowledgebase</a></li>
                    <li><a href="{$WEB_ROOT}/downloads.php">Downloads</a></li>
                </ul>
            </div>

            {* Contact *}
            <div class="col-md-4">
                <h5>{$LANG.contactus}</h5>
                <ul class="list-unstyled text-muted">
                    <li><i class="fa fa-phone"></i> {$company_phone}</li>
                    <li><i class="fa fa-envelope"></i> {$company_email}</li>
                </ul>
            </div>
        </div>

        {* Copyright *}
        <div class="footer-bottom text-center">
            <p class="text-muted mb-0">
                &copy; {date('Y')} {$company_name}. All rights reserved.
            </p>
        </div>
    </div>
</footer>
```

## Custom CSS

```css
/* templates/your-theme/assets/css/style.css */

/* CSS Variables */
:root {
    --primary-color: #007bff;
    --primary-hover: #0056b3;
    --secondary-color: #6c757d;
    --success-color: #28a745;
    --danger-color: #dc3545;
    --warning-color: #ffc107;
    --info-color: #17a2b8;
    --light-bg: #f8f9fa;
    --dark-bg: #343a40;
    --border-radius: 8px;
    --box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    --transition-speed: 0.3s;
}

/* Base Styles */
body {
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
    font-size: 16px;
    line-height: 1.6;
    color: #333;
    background-color: #fff;
}

/* Typography */
h1, h2, h3, h4, h5, h6 {
    font-weight: 600;
    margin-bottom: 1rem;
}

a {
    color: var(--primary-color);
    transition: color var(--transition-speed);
}

a:hover {
    color: var(--primary-hover);
    text-decoration: none;
}

/* Navigation */
.site-header {
    background: #fff;
    box-shadow: var(--box-shadow);
    position: sticky;
    top: 0;
    z-index: 1000;
}

.navbar {
    padding: 0.75rem 0;
}

.navbar-brand img {
    height: 40px;
    width: auto;
}

.navbar-nav .nav-link {
    padding: 0.5rem 1rem;
    color: #333;
    font-weight: 500;
    transition: color var(--transition-speed);
}

.navbar-nav .nav-link:hover {
    color: var(--primary-color);
}

/* Cards */
.card {
    border: none;
    border-radius: var(--border-radius);
    box-shadow: var(--box-shadow);
    transition: transform var(--transition-speed), box-shadow var(--transition-speed);
}

.card:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.15);
}

.card-header {
    background: var(--primary-color);
    color: #fff;
    border-radius: var(--border-radius) var(--border-radius) 0 0;
    padding: 1rem 1.25rem;
}

/* Buttons */
.btn {
    border-radius: var(--border-radius);
    padding: 0.5rem 1.5rem;
    font-weight: 500;
    transition: all var(--transition-speed);
}

.btn-primary {
    background: var(--primary-color);
    border-color: var(--primary-color);
}

.btn-primary:hover {
    background: var(--primary-hover);
    border-color: var(--primary-hover);
}

/* Tables */
.table {
    border-collapse: separate;
    border-spacing: 0;
}

.table thead th {
    background: var(--primary-color);
    color: #fff;
    border: none;
    padding: 1rem;
    font-weight: 600;
}

.table tbody tr:hover {
    background: var(--light-bg);
}

/* Forms */
.form-control {
    border-radius: var(--border-radius);
    padding: 0.75rem 1rem;
    border: 1px solid #ddd;
    transition: border-color var(--transition-speed), box-shadow var(--transition-speed);
}

.form-control:focus {
    border-color: var(--primary-color);
    box-shadow: 0 0 0 3px rgba(0, 123, 255, 0.25);
}

/* Alerts */
.alert {
    border-radius: var(--border-radius);
    border: none;
}

/* Badges */
.badge {
    padding: 0.35em 0.65em;
    font-weight: 500;
}

/* Utility Classes */
.shadow-sm {
    box-shadow: var(--box-shadow);
}

.rounded-lg {
    border-radius: calc(var(--border-radius) * 1.5);
}

/* Footer */
.site-footer {
    background: var(--dark-bg);
    color: #fff;
    padding: 3rem 0 1.5rem;
    margin-top: 4rem;
}

.site-footer h5 {
    color: #fff;
    margin-bottom: 1.5rem;
}

.site-footer a {
    color: rgba(255, 255, 255, 0.7);
}

.site-footer a:hover {
    color: #fff;
}

.footer-bottom {
    border-top: 1px solid rgba(255, 255, 255, 0.1);
    padding-top: 1.5rem;
    margin-top: 2rem;
}
```

## Responsive CSS

```css
/* templates/your-theme/assets/css/responsive.css */

/* Extra Small Devices */
@media (max-width: 575.98px) {
    body {
        font-size: 14px;
    }

    .site-header .navbar-brand img {
        height: 30px;
    }

    .main-content {
        padding: 1rem 0;
    }

    .card {
        margin-bottom: 1rem;
    }

    .table-responsive {
        font-size: 13px;
    }
}

/* Small Devices */
@media (min-width: 576px) and (max-width: 767.98px) {
    .navbar-collapse {
        background: #fff;
        padding: 1rem;
        margin-top: 0.5rem;
        border-radius: var(--border-radius);
    }
}

/* Medium Devices */
@media (min-width: 768px) and (max-width: 991.98px) {
    .sidebar {
        margin-bottom: 2rem;
    }
}

/* Large Devices */
@media (min-width: 992px) {
    .site-header {
        padding: 0;
    }
}
```

## JavaScript

```javascript
// templates/your-theme/assets/js/main.js

(function() {
    'use strict';

    // Initialize on DOM ready
    document.addEventListener('DOMContentLoaded', function() {
        initNavigation();
        initForms();
        initModals();
        initTooltips();
    });

    // Mobile Navigation
    function initNavigation() {
        const navbarToggles = document.querySelectorAll('.navbar-toggler');

        navbarToggles.forEach(function(toggle) {
            toggle.addEventListener('click', function() {
                const target = document.querySelector(this.getAttribute('data-target'));
                this.classList.toggle('active');
                target.classList.toggle('show');
            });
        });
    }

    // Form Validation
    function initForms() {
        const forms = document.querySelectorAll('.needs-validation');

        forms.forEach(function(form) {
            form.addEventListener('submit', function(event) {
                if (!form.checkValidity()) {
                    event.preventDefault();
                    event.stopPropagation();
                }
                form.classList.add('was-validated');
            }, false);
        });
    }

    // Confirm Actions
    function initModals() {
        document.querySelectorAll('[data-confirm]').forEach(function(el) {
            el.addEventListener('click', function(e) {
                const message = this.getAttribute('data-confirm') || 'Are you sure?';
                if (!confirm(message)) {
                    e.preventDefault();
                }
            });
        });
    }

    // Tooltips
    function initTooltips() {
        const tooltipTriggerList = [].slice.call(
            document.querySelectorAll('[data-toggle="tooltip"]')
        );

        tooltipTriggerList.forEach(function(tooltipTriggerEl) {
            new bootstrap.Tooltip(tooltipTriggerEl);
        });
    }

    // AJAX Loading States
    window.showLoading = function(element) {
        element.classList.add('loading');
        element.disabled = true;
    };

    window.hideLoading = function(element) {
        element.classList.remove('loading');
        element.disabled = false;
    };

})();
```

## Custom Client Area Pages

### Custom Homepage Widget

```php
// Add to template/includes/hooks.php

use WHMCS\Custom\Widget\HomepageWidgetInterface;

add_hook('ClientAreaHomepage', 1, function($vars) {
    $client = $vars['client'];

    // Get custom data
    $recentInvoices = $client->invoices()
        ->where('status', '!=', 'Paid')
        ->orderBy('duedate', 'asc')
        ->limit(3)
        ->get();

    return [
        'templateFile' => 'templates/your-theme/includes/widgets/custom-status.tpl',
        'vars' => [
            'recentInvoices' => $recentInvoices,
        ],
    ];
});
```

## Theme Hooks

```php
// templates/your-theme/includes/hooks.php

// Pre-render hook
add_hook('PreTemplateRender', 1, function($vars) {
    // Add custom CSS class to body
    return ['bodyClass' => 'custom-theme-v1'];
});

// Client area header
add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    return '<link rel="stylesheet" href="custom-header.css">';
});

// Footer output
add_hook('ClientAreaFooterOutput', 1, function($vars) {
    return '<script>console.log("Theme loaded");</script>';
});
```

## Best Practices

1. **Inherit from Default Theme**: Copy and modify default theme files
2. **Use CSS Variables**: Enable easy customization and theming
3. **Mobile-First Design**: Start with mobile styles, add media queries for larger screens
4. **Minimize Custom JavaScript**: Prefer WHMCS built-in functionality
5. **Template Variables Check**: Always check if variables exist
6. **Cache Assets**: Minify CSS/JS in production
7. **RTL Support**: Include RTL stylesheet if supporting RTL languages

## Related Documentation

- [Admin Area Customization](admin-area-customization.md)
- [Smarty Template Reference](smarty-template-reference.md)
- [Caching Strategies](caching-strategies.md)
