# WHMCS Client Area Theme Development Workflow

## Purpose

Complete guide to developing custom WHMCS client area themes. Covers theme structure, Smarty templates, asset management, responsive design, Bootstrap customization, and performance optimization.

## Prerequisites

- WHMCS 7.0+ installation
- PHP 7.4+ knowledge
- HTML/CSS/JavaScript skills
- Understanding of Smarty templating
- Basic Bootstrap knowledge

## Workflow Steps

### Step 1: Understanding WHMCS Theme Architecture

```
Theme Directory Structure:

templates/
├── six/
│   ├── template.php          # Theme configuration
│   ├── header.tpl            # Global header
│   ├── footer.tpl             # Global footer
│   ├── homepage.tpl           # Homepage template
│   ├── login.tpl             # Login page
│   ├── clientregister.tpl    # Registration
│   ├── password.tpl           # Password reset
│   ├── viewinvoice.tpl        # Invoice viewing
│   ├── invoices.tpl          # Invoice list
│   ├── supporttickets.tpl     # Tickets
│   ├── knowledgebase.tpl     # KB articles
│   ├── announcements.tpl     # News/announcements
│   └── {module}/
│       └── templates/        # Module templates
│
├── yourcustomtheme/
│   ├── template.php
│   ├── overrides/            # Override specific templates
│   ├── templates/
│   ├── assets/
│   │   ├── css/
│   │   ├── js/
│   │   └── fonts/
│   └── templates_c/          # Compiled templates (auto-created)
```

### Step 2: Creating Theme Configuration

```php
<?php
/**
 * WHMCS Custom Theme Configuration
 *
 * Place in templates/yourtheme/template.php
 */

use WHMCS\View\Asset;
use WHMCS\View\Markup\Markup Registrar;

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Theme registration
 */
function yourtheme_meta(): array
{
    return [
        'name' => 'Your Custom Theme',
        'author' => 'Your Company',
        'description' => 'Custom WHMCS client area theme',
        'version' => '1.0.0',
        'requires' => '7.0.0',
        'supports' => [
            'client_area',
            'signup',
            'login',
        ],
    ];
}

/**
 * Register theme assets
 */
function yourtheme_assets(): array
{
    return [
        'css' => [
            'custom' => 'assets/css/custom.min.css',
            'variables' => 'assets/css/variables.css',
            'dark-mode' => 'assets/css/dark-mode.css',
        ],
        'js' => [
            'app' => 'assets/js/app.js',
            'charts' => 'assets/js/charts.js',
        ],
        'fonts' => [
            'inter' => 'https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap',
        ],
    ];
}

/**
 * Theme configuration options
 */
function yourtheme_config(): array
{
    return [
        'Logo' => [
            'Type' => 'text',
            'Name' => 'Logo URL',
            'Default' => 'assets/images/logo.svg',
            'Description' => 'Enter the URL to your logo',
        ],
        'Primary Color' => [
            'Type' => 'text',
            'Name' => 'Primary Color (HEX)',
            'Default' => '#0066cc',
            'Description' => 'Enter primary color in HEX format',
        ],
        'Dark Mode' => [
            'Type' => 'yesno',
            'Name' => 'Enable Dark Mode',
            'Default' => 'no',
        ],
        'Custom CSS' => [
            'Type' => 'textarea',
            'Name' => 'Custom CSS',
            'Default' => '',
            'Description' => 'Add custom CSS rules',
        ],
    ];
}

/**
 * Pre-template hook for common variables
 */
add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    $configVars = \DI::make('config');

    return <<<HTML
<style>
    :root {
        --primary-color: {$configVars->get('yourtheme_primary_color', '#0066cc')};
        --secondary-color: {$configVars->get('yourtheme_secondary_color', '#6c757d')};
        --accent-color: {$configVars->get('yourtheme_accent_color', '#28a745')};
    }
</style>
HTML;
});
```

### Step 3: Building Templates with Smarty

```smarty
{* templates/yourtheme/header.tpl *}

<!DOCTYPE html>
<html lang="{$LANG.locale|sanitize:'html':'path'}" {if $templatefile == 'login'}data-whmcs-theme="login"{/if}>
<head>
    <meta charset="{$charset|default:'UTF-8'}">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="robots" content="noindex,nofollow">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">

    <title>{$pageTitle} - {$companyname}</title>

    <link rel="icon" type="image/x-icon" href="{$base_url}/favicon.ico">

    {* Theme assets *}
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
    <link rel="stylesheet" href="{$base_url}/templates/{$template}/assets/css/variables.css">
    <link rel="stylesheet" href="{$base_url}/templates/{$template}/assets/css/bootstrap.min.css">
    <link rel="stylesheet" href="{$base_url}/templates/{$template}/assets/css/main.min.css">

    {if isset($customCss)}
        <style>{$customCss}</style>
    {/if}

    {* Insert header output from hooks *}
    {$headoutput}

    <body class="whmcs {(($loggedin) ? 'client-area' : 'login-area')} {if $templatefile}template-{$templatefile}{/if}">
        {* Skip to main content link *}
        <a href="#main-content" class="skip-to-content">
            {lang key='skipToMainContent'}
        </a>

        {* Top navigation bar *}
        <header class="site-header" role="banner">
            <nav class="navbar navbar-expand-lg" role="navigation">
                <div class="container">
                    <a class="navbar-brand" href="{$base_url}/">
                        <img src="{$theme_logo}" alt="{$companyname}" height="40">
                    </a>

                    <button class="navbar-toggler" type="button"
                            data-toggle="collapse"
                            data-target="#mainNav"
                            aria-controls="mainNav"
                            aria-expanded="false"
                            aria-label="Toggle navigation">
                        <span class="navbar-toggler-icon"></span>
                    </button>

                    <div class="collapse navbar-collapse" id="mainNav">
                        {* Main navigation *}
                        <ul class="navbar-nav mr-auto">
                            <li class="nav-item">
                                <a class="nav-link" href="{$base_url}/index.php">
                                    <i class="fa fa-home"></i> {lang key='home'}
                                </a>
                            </li>
                            <li class="nav-item">
                                <a class="nav-link" href="{$base_url}/cart.php">
                                    <i class="fa fa-shopping-cart"></i> {lang key='order'}
                                </a>
                            </li>
                            {if $loggedin}
                                <li class="nav-item">
                                    <a class="nav-link" href="{$base_url}/clientarea.php">
                                        <i class="fa fa-user"></i> {lang key='myaccount'}
                                    </a>
                                </li>
                            {/if}
                        </ul>

                        {* Login/Account dropdown *}
                        <ul class="navbar-nav">
                            {if $loggedin}
                                <li class="nav-item dropdown">
                                    <a class="nav-link dropdown-toggle" href="#"
                                       data-toggle="dropdown"
                                       aria-haspopup="true">
                                        <i class="fa fa-user-circle"></i>
                                        {$client->firstname} {$client->lastname}
                                    </a>
                                    <div class="dropdown-menu dropdown-menu-right">
                                        <a class="dropdown-item" href="{$base_url}/clientarea.php">
                                            <i class="fa fa-tachometer"></i> Dashboard
                                        </a>
                                        <a class="dropdown-item" href="{$base_url}/clientarea.php?action=details">
                                            <i class="fa fa-user"></i> Profile
                                        </a>
                                        <div class="dropdown-divider"></div>
                                        <a class="dropdown-item" href="{$base_url}/logout.php">
                                            <i class="fa fa-sign-out"></i> {lang key='logout'}
                                        </a>
                                    </div>
                                </li>
                            {else}
                                <li class="nav-item">
                                    <a class="nav-link" href="{$base_url}/login.php">
                                        <i class="fa fa-sign-in"></i> {lang key='login'}
                                    </a>
                                </li>
                                <li class="nav-item">
                                    <a class="nav-link btn btn-primary text-white" href="{$base_url}/register.php">
                                        {lang key='register'}
                                    </a>
                                </li>
                            {/if}
                        </ul>
                    </div>
                </div>
            </nav>
        </header>

        {* Breadcrumb navigation *}
        {if isset($breadcrumbs) && count($breadcrumbs) > 0}
            <nav class="breadcrumb-nav" aria-label="Breadcrumb">
                <div class="container">
                    <ol class="breadcrumb">
                        <li class="breadcrumb-item">
                            <a href="{$base_url}/">
                                <i class="fa fa-home"></i>
                            </a>
                        </li>
                        {foreach $breadcrumbs as $crumb}
                            <li class="breadcrumb-item {if $crumb.active}active{/if}">
                                {if $crumb.active}
                                    {$crumb.label}
                                {else}
                                    <a href="{$crumb.link}">{$crumb.label}</a>
                                {/if}
                            </li>
                        {/foreach}
                    </ol>
                </div>
            </nav>
        {/if}

        {* Flash messages *}
        <main id="main-content" class="main-content" role="main" tabindex="-1">
            <div class="container">
                {foreach $flash messages as $message}
                    <div class="alert alert-{$message.type} alert-dismissible" role="alert">
                        <button type="button" class="close" data-dismiss="alert" aria-label="Close">
                            <span aria-hidden="true">&times;</span>
                        </button>
                        {$message.message}
                    </div>
                {/foreach}
```

```smarty
{* templates/yourtheme/footer.tpl *}

            </div>
        </main>

        {* Footer *}
        <footer class="site-footer" role="contentinfo">
            <div class="container">
                <div class="row">
                    <div class="col-md-4">
                        <h4>{$companyname}</h4>
                        <p>{$companyaddress|nl2br}</p>
                    </div>
                    <div class="col-md-2">
                        <h5>{lang key='footerNavLabel'}</h5>
                        <ul class="footer-links">
                            <li><a href="{$base_url}/about.php">About Us</a></li>
                            <li><a href="{$base_url}/contact.php">Contact</a></li>
                            <li><a href="{$base_url}/privacy.php">Privacy Policy</a></li>
                            <li><a href="{$base_url}/terms.php">Terms of Service</a></li>
                        </ul>
                    </div>
                    <div class="col-md-2">
                        <h5>{lang key='support'}</h5>
                        <ul class="footer-links">
                            <li><a href="{$base_url}/supporttickets.php">Tickets</a></li>
                            <li><a href="{$base_url}/knowledgebase.php">Knowledge Base</a></li>
                            <li><a href="{$base_url}/announcements.php">Announcements</a></li>
                        </ul>
                    </div>
                    <div class="col-md-4">
                        <h5>{lang key='connect'}</h5>
                        <div class="social-links">
                            <a href="{$social_links.twitter|default:'#'}" class="social-link" aria-label="Twitter">
                                <i class="fa fa-twitter"></i>
                            </a>
                            <a href="{$social_links.facebook|default:'#'}" class="social-link" aria-label="Facebook">
                                <i class="fa fa-facebook"></i>
                            </a>
                            <a href="{$social_links.linkedin|default:'#'}" class="social-link" aria-label="LinkedIn">
                                <i class="fa fa-linkedin"></i>
                            </a>
                        </div>
                    </div>
                </div>
                <div class="footer-bottom">
                    <p>&copy; {date('Y')} {$companyname}. {lang key='allRightsReserved'}</p>
                </div>
            </div>
        </footer>

        {* Back to top button *}
        <button class="back-to-top" aria-label="Back to top" title="Back to top">
            <i class="fa fa-chevron-up"></i>
        </button>

        {* Theme JavaScript *}
        <script src="{$base_url}/templates/{$template}/assets/js/vendor/jquery.min.js"></script>
        <script src="{$base_url}/templates/{$template}/assets/js/vendor/bootstrap.bundle.min.js"></script>
        <script src="{$base_url}/templates/{$template}/assets/js/app.min.js"></script>

        {* Hook output for footer *}
        {$footeroutput}

        {if $templatefile == 'viewinvoice'}
            <script src="{$base_url}/templates/{$template}/assets/js/invoice.js"></script>
        {/if}
    </body>
</html>
```

### Step 4: Creating Custom Client Area Pages

```smarty
{* templates/yourtheme/clientarea.tpl *}

{include file="$template/header.tpl"}

<div class="client-area-wrapper">
    <div class="row">
        {* Sidebar navigation *}
        <aside class="col-md-3 sidebar-nav">
            <div class="card">
                <div class="card-body">
                    <div class="user-profile-header">
                        <div class="avatar-circle">
                            {if $client->email}
                                <span>{$client->firstname|substr:0:1}{$client->lastname|substr:0:1}</span>
                            {else}
                                <i class="fa fa-user"></i>
                            {/if}
                        </div>
                        <div class="user-info">
                            <h5>{$client->firstname} {$client->lastname}</h5>
                            <p class="text-muted">{$client->email}</p>
                        </div>
                    </div>

                    <nav class="nav-pills flex-column">
                        <a class="nav-link {if $currentpage == 'dashboard'}active{/if}" 
                           href="clientarea.php">
                            <i class="fa fa-tachometer"></i> Dashboard
                        </a>
                        <a class="nav-link {if $currentpage == 'services'}active{/if}"
                           href="clientarea.php?action=services">
                            <i class="fa fa-server"></i> {lang key='myProducts'}
                        </a>
                        <a class="nav-link {if $currentpage == 'domains'}active{/if}"
                           href="clientarea.php?action=domains">
                            <i class="fa fa-globe"></i> {lang key='myDomains'}
                        </a>
                        <a class="nav-link {if $currentpage == 'invoices'}active{/if}"
                           href="clientarea.php?action=invoices">
                            <i class="fa fa-file-text-o"></i> {lang key='myInvoices'}
                        </a>
                        <a class="nav-link {if $currentpage == 'tickets'}active{/if}"
                           href="clientarea.php?action=supporttickets">
                            <i class="fa fa-ticket"></i> {lang key='supportTickets'}
                        </a>
                        <a class="nav-link {if $currentpage == 'billing'}active{/if}"
                           href="clientarea.php?action=billing">
                            <i class="fa fa-credit-card"></i> Billing
                        </a>
                        <a class="nav-link {if $currentpage == 'profile'}active{/if}"
                           href="clientarea.php?action=details">
                            <i class="fa fa-user-circle"></i> {lang key='myDetails'}
                        </a>
                        <a class="nav-link {if $currentpage == 'password'}nav-link{/if}"
                           href="clientarea.php?action=security">
                            <i class="fa fa-lock"></i> Security
                        </a>
                        {if $condtl.logout2fa}
                            <a class="nav-link" href="clientarea.php?action=logout2fa">
                                <i class="fa fa-shield"></i> 2FA
                            </a>
                        {/if}
                    </nav>
                </div>
            </div>

            {* Quick stats card *}
            <div class="card mt-3">
                <div class="card-header">
                    <h5 class="mb-0">Quick Stats</h5>
                </div>
                <div class="card-body">
                    <ul class="stats-list">
                        <li>
                            <span class="stat-label">Active Services</span>
                            <span class="stat-value">{$stats.activeServices}</span>
                        </li>
                        <li>
                            <span class="stat-label">Domains</span>
                            <span class="stat-value">{$stats.totalDomains}</span>
                        </li>
                        <li>
                            <span class="stat-label">Open Tickets</</span>
                            <span class="stat-value">{$stats.openTickets}</span>
                        </li>
                        <li>
                            <span class="stat-label">Balance</span>
                            <span class="stat-value">{$client->credit|货币格式:'USD'}</span>
                        </li>
                    </ul>
                </div>
            </div>
        </aside>

        {* Main content area *}
        <section class="col-md-9 main-content-area">
            {* Page title *}
            <div class="page-header">
                <h1>{$pageTitle}</h1>
                {if isset($pageSubtitle)}
                    <p class="text-muted">{$pageSubtitle}</p>
                {/if}
            </div>

            {* Content based on current page *}
            {switch $currentpage}
                {case "dashboard"}
                    {include file="$template/partials/dashboard.tpl"}
                {case "services"}
                    {include file="$template/partials/services.tpl"}
                {case "invoices"}
                    {include file="$template/partials/invoices.tpl"}
                {case "tickets"}
                    {include file="$template/partials/tickets.tpl"}
                {case "profile"}
                    {include file="$template/partials/profile.tpl"}
                {default}
                    {$content}
            {/switch}
        </section>
    </div>
</div>

{include file="$template/footer.tpl"}
```

### Step 5: CSS Architecture and Variables

```css
/* assets/css/variables.css */

:root {
    /* Colors */
    --primary-color: #0066cc;
    --primary-hover: #0052a3;
    --primary-light: #e6f0f7;
    --secondary-color: #6c757d;
    --secondary-hover: #5a6268;
    --accent-color: #28a745;
    --accent-hover: #218838;
    --danger-color: #dc3545;
    --warning-color: #ffc107;
    --info-color: #17a2b8;
    --success-color: #28a745;

    /* Grays */
    --gray-100: #f8f9fa;
    --gray-200: #e9ecef;
    --gray-300: #dee2e6;
    --gray-400: #ced4da;
    --gray-500: #adb5bd;
    --gray-600: #6c757d;
    --gray-700: #495057;
    --gray-800: #343a40;
    --gray-900: #212529;

    /* Typography */
    --font-family-base: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
    --font-family-heading: 'Inter', var(--font-family-base);
    --font-size-base: 1rem;
    --font-size-sm: 0.875rem;
    --font-size-lg: 1.25rem;
    --font-weight-normal: 400;
    --font-weight-medium: 500;
    --font-weight-semibold: 600;
    --font-weight-bold: 700;
    --line-height-base: 1.6;
    --line-height-heading: 1.3;

    /* Spacing */
    --spacing-xs: 0.25rem;
    --spacing-sm: 0.5rem;
    --spacing-md: 1rem;
    --spacing-lg: 1.5rem;
    --spacing-xl: 2rem;
    --spacing-2xl: 3rem;

    /* Border Radius */
    --border-radius-sm: 0.25rem;
    --border-radius: 0.5rem;
    --border-radius-lg: 1rem;
    --border-radius-pill: 50rem;

    /* Shadows */
    --shadow-sm: 0 0.125rem 0.25rem rgba(0, 0, 0, 0.04);
    --shadow: 0 0.5rem 1rem rgba(0, 0, 0, 0.1);
    --shadow-lg: 0 1rem 3rem rgba(0, 0, 0, 0.15);

    /* Transitions */
    --transition-fast: 0.15s ease-in-out;
    --transition: 0.3s ease-in-out;
    --transition-slow: 0.5s ease-in-out;

    /* Breakpoints */
    --breakpoint-sm: 576px;
    --breakpoint-md: 768px;
    --breakpoint-lg: 992px;
    --breakpoint-xl: 1200px;

    /* Container max-widths */
    --container-sm: 540px;
    --container-md: 720px;
    --container-lg: 960px;
    --container-xl: 1140px;
}

/* Dark mode */
[data-theme="dark"] {
    --body-bg: #1a1a2e;
    --body-color: #e4e4e7;
    --card-bg: #16213e;
    --card-border: #2a3f5f;
    --primary-color: #4d94ff;
    --primary-hover: #6aabff;
}

/* Minimalize default styles reset */
.whmcs-body {
    background-color: var(--gray-100);
    color: var(--gray-900);
    font-family: var(--font-family-base);
    line-height: var(--line-height-base);
}

/* Global card styling */
.card {
    background-color: #fff;
    border: 1px solid var(--gray-200);
    border-radius: var(--border-radius);
    box-shadow: var(--shadow-sm);
    transition: box-shadow var(--transition);
}

.card:hover {
    box-shadow: var(--shadow);
}

.card-body {
    padding: var(--spacing-lg);
}

/* Button styles */
.btn {
    font-weight: var(--font-weight-medium);
    padding: var(--spacing-sm) var(--spacing-lg);
    border-radius: var(--border-radius);
    transition: all var(--transition-fast);
}

.btn-primary {
    background-color: var(--primary-color);
    border-color: var(--primary-color);
}

.btn-primary:hover {
    background-color: var(--primary-hover);
    border-color: var(--primary-hover);
    transform: translateY(-1px);
}

.btn-outline-primary {
    color: var(--primary-color);
    border-color: var(--primary-color);
}

.btn-outline-primary:hover {
    background-color: var(--primary-color);
    border-color: var(--primary-color);
    color: #fff;
}

/* Form styles */
.form-control {
    border: 1px solid var(--gray-300);
    border-radius: var(--border-radius-sm);
    padding: var(--spacing-sm) var(--spacing-md);
    transition: border-color var(--transition-fast), box-shadow var(--transition-fast);
}

.form-control:focus {
    border-color: var(--primary-color);
    box-shadow: 0 0 0 3px rgba(0, 102, 204, 0.15);
}

/* Table styles */
.table {
    background-color: #fff;
    border-radius: var(--border-radius);
    overflow:
    hidden;
}

.table thead th {
    background-color: var(--gray-100);
    border-bottom: 2px solid var(--gray-200);
    font-weight: var(--font-weight-semibold);
    text-transform: uppercase;
    font-size: var(--font-size-sm);
    letter-spacing: 0.05em;
    color: var(--gray-600);
}

.table tbody tr:hover {
    background-color: var(--gray-100);
}

/* Badge variants */
.badge-success {
    background-color: #d4edda;
    color: #155724;
}

.badge-warning {
    background-color: #fff3cd;
    color: #856404;
}

.badge-danger {
    background-color: #f8d7da;
    color: #721c24;
}

/* Nav styles */
.nav-pills .nav-link {
    color: var(--gray-700);
    padding: var(--spacing-sm) var(--spacing-md);
    border-radius: var(--border-radius);
    transition: all var(--transition-fast);
}

.nav-pills .nav-link:hover {
    background-color: var(--gray-100);
}

.nav-pills .nav-link.active {
    background-color: var(--primary-color);
    color: #fff;
}

/* Responsive helpers */
@media (max-width: 768px) {
    .sidebar-nav {
        margin-bottom: var(--spacing-lg);
    }

    .page-header h1 {
        font-size: 1.5rem;
    }
}

/* Back to top button */
.back-to-top {
    position: fixed;
    bottom: 2rem;
    right: 2rem;
    width: 48px;
    height: 48px;
    border-radius: var(--border-radius-pill);
    background-color: var(--primary-color);
    color: #fff;
    border: none;
    box-shadow: var(--shadow);
    cursor: pointer;
    opacity: 0;
    visibility: hidden;
    transition: all var(--transition);
    z-index: 1000;
}

.back-to-top.visible {
    opacity: 1;
    visibility: visible;
}

.back-to-top:hover {
    transform: translateY(-3px);
    background-color: var(--primary-hover);
}
```

### Step 6: JavaScript Module

```javascript
// assets/js/app.js

/**
 * WHMCS Custom Theme JavaScript
 */

const ThemeApp = {
    // Initialize app
    init() {
        this.initBackToTop();
        this.initDarkMode();
        this.initTooltips();
        this.initSmoothScroll();
        this.initFlashMessages();
        this.initFormValidation();
        this.initLazyLoad();
        this.initMobileNav();
    },

    // Back to top button
    initBackToTop() {
        const $btn = $('.back-to-top');

        $(window).scroll(function() {
            if ($(this).scrollTop() > 300) {
                $btn.addClass('visible');
            } else {
                $btn.removeClass('visible');
            }
        });

        $btn.click(function() {
            $('html, body').animate({
                scrollTop: 0
            }, 300);
        });
    },

    // Dark mode toggle
    initDarkMode() {
        const $toggle = $('.dark-mode-toggle');
        const theme = localStorage.getItem('theme') || 'light';

        if (theme === 'dark') {
            document.documentElement.setAttribute('data-theme', 'dark');
            $toggle.prop('checked', true);
        }

        $toggle.change(function() {
            const newTheme = $(this).prop('checked') ? 'dark' : 'light';
            localStorage.setItem('theme', newTheme);
            document.documentElement.setAttribute('data-theme', newTheme);
        });
    },

    // Bootstrap tooltips
    initTooltips() {
        $('[data-toggle="tooltip"]').tooltip();
        $('[data-toggle="popover"]').popover();
    },

    // Smooth scroll for anchor links
    initSmoothScroll() {
        $('a[href^="#"]').on('click', function(e) {
            const target = $(this.getAttribute('href'));
            if (target.length) {
                e.preventDefault();
                $('html, body').stop().animate({
                    scrollTop: target.offset().top - 100
                }, 300);
            }
        });
    },

    // Auto-dismiss flash messages
    initFlashMessages() {
        setTimeout(function() {
            $('.alert:not(.alert-persistent)').fadeOut('slow', function() {
                $(this).remove();
            });
        }, 5000);
    },

    // Form validation enhancement
    initFormValidation() {
        $('form[data-validate="true"]').each(function() {
            $(this).validate({
                errorElement: 'div',
                errorClass: 'invalid-feedback',
                highlight: function(element) {
                    $(element).addClass('is-invalid');
                },
                unhighlight: function(element) {
                    $(element).removeClass('is-invalid');
                },
                errorPlacement: function(error, element) {
                    error.insertAfter(element.closest('.input-group, .form-group'));
                }
            });
        });
    },

    // Lazy load images
    initLazyLoad() {
        if ('IntersectionObserver' in window) {
            const observer = new IntersectionObserver(function(entries) {
                entries.forEach(function(entry) {
                    if (entry.isIntersecting) {
                        const img = entry.target;
                        img.src = img.dataset.src;
                        img.classList.remove('lazy');
                        observer.unobserve(img);
                    }
                });
            }, {
                rootMargin: '50px 0px'
            });

            document.querySelectorAll('img[data-src]').forEach(function(img) {
                observer.observe(img);
            });
        }
    },

    // Mobile navigation enhancements
    initMobileNav() {
        $('.navbar-toggler').on('click', function() {
            const expanded = $(this).attr('aria-expanded') === 'true';
            $('body').toggleClass('nav-open', expanded);
        });

        // Close nav on link click
        $('.navbar-nav a').on('click', function() {
            if ($(window).width() < 992) {
                $('.navbar-collapse').collapse('hide');
            }
        });
    },

    // Confirm delete actions
    confirmDelete(message) {
        return confirm(message || 'Are you sure you want to delete this item?');
    },

    // Copy to clipboard
    copyToClipboard(text, successMsg = 'Copied!') {
        navigator.clipboard.writeText(text).then(function() {
            WHMCS.jsNotifier.notify(successMsg);
        }).catch(function() {
            // Fallback for older browsers
            const textarea = document.createElement('textarea');
            textarea.value = text;
            document.body.appendChild(textarea);
            textarea.select();
            document.execCommand('copy');
            document.body.removeChild(textarea);
            WHMCS.jsNotifier.notify(successMsg);
        });
    }
};

// Initialize when DOM is ready
$(document).ready(function() {
    ThemeApp.init();
});

// Export for module use
window.ThemeApp = ThemeApp;
```

## Verification Checklist

```
Theme Structure:
□ Template files in correct locations
□ Assets organized in css/js/fonts folders
□ template.php configuration file present
□ All required WHMCS templates overridden
□ Smarty variables used correctly

CSS/Styling:
□ CSS variables defined for theming
□ Responsive breakpoints implemented
□ Dark mode support (if needed)
□ Print styles included
□ Custom fonts loaded properly
□ Bootstrap properly extended

JavaScript:
□ No jQuery conflicts
□ Proper event delegation
□ Progressive enhancement
□ Lazy loading implemented
□ Accessibility features working

Templates:
□ Smarty syntax correct
□ Lang keys used for text
□ XSS prevention (escaping)
□ SEO meta tags included
□ ARIA attributes on interactive elements

Accessibility:
□ Skip links included
□ Semantic HTML structure
□ Keyboard navigation support
□ Focus indicators visible
□ Screen reader friendly

Performance:
□ CSS/JS minified for production
□ Images optimized
□ Fonts preloaded
□ Critical CSS inlined
□ Lazy loading for off-screen content
```

## WHMCS ClassDocs References

- [Smarty Template Guide](https://developers.whmcs.com/customizing-ui/smarty/)
- [Template Variables](https://developers.whmcs.com/customizing-ui/template-variables/)
- [Asset Management](https://developers.whmcs.com/customizing-ui/assets/)
- [Hooks: ClientAreaHeader](https://developers.whmcs.com/advanced/hooks-system/)
