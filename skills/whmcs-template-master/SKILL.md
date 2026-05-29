# WHMCS Template Master

## Overview
Master skill for WHMCS template development. Covers template structure, Smarty templates, CSS/JS integration, and responsive design patterns.

## Template Structure

```
/templates/your_template/
├── assets/
│   ├── css/
│   │   ├── main.css
│   │   ├── responsive.css
│   │   └── custom.css
│   ├── js/
│   │   ├── main.js
│   │   ├── validate.js
│   │   └── custom.js
│   ├── images/
│   │   ├── logo.png
│   │   └── favicon.ico
│   └── fonts/
├── layouts/
│   ├── homepage.tpl
│   ├── container-wide.tpl
│   ├── container-fluid.tpl
│   └── container-sidebar.tpl
├── includes/
│   ├── header.tpl
│   ├── footer.tpl
│   ├── sidebar.tpl
│   └── navigation.tpl
├── client/
│   ├── home.tpl
│   ├── billing.tpl
│   └── services.tpl
├── storefront/
│   ├── cart.tpl
│   ├── checkout.tpl
│   └── products.tpl
├── support/
│   ├── tickets.tpl
│   └── knowledgebase.tpl
├── email_templates/
│   └── (custom email templates)
├── 404.tpl
├── error.tpl
└── template.php
```

## Template Configuration

```php
<?php
// /templates/your_template/template.php

function your_template_getConfigArray()
{
    return [
        'displayName' => [
            'Type' => 'text',
            'Value' => 'Your Template Name',
            'Description' => 'Name shown in admin panel',
        ],
        'logoUrl' => [
            'Type' => 'text',
            'Description' => 'Logo URL or upload via Media Manager',
        ],
        'primaryColor' => [
            'Type' => 'text',
            'Value' => '#3498db',
            'Description' => 'Primary brand color (hex code)',
        ],
        'secondaryColor' => [
            'Type' => 'text',
            'Value' => '#2ecc71',
            'Description' => 'Secondary brand color (hex code)',
        ],
        'customCSS' => [
            'Type' => 'textarea',
            'Description' => 'Custom CSS overrides',
        ],
        'showFooter' => [
            'Type' => 'yesno',
            'Description' => 'Show footer on all pages',
        ],
        'enableRTL' => [
            'Type' => 'yesno',
            'Description' => 'Enable RTL support',
        ],
    ];
}

function your_template_output(array $vars)
{
    // Global JavaScript variables
    $vars['jsCode'] = <<<JS
        <script>
            window.whmcsBaseUrl = '{$vars['system_url']}';
            window.whmcsLang = {json_encode($vars['LANG'])};
            window.csrfToken = '{$vars['token']}';
        </script>
    JS;

    // Preload critical assets
    $vars['preloadAssets'] = [
        'css' => [
            'assets/css/main.min.css',
            'assets/css/responsive.min.css',
        ],
        'js' => [
            'assets/js/main.min.js',
        ],
    ];

    return $vars;
}
```

## Header Template

```smarty
{* /templates/your_template/includes/header.tpl *}
<!DOCTYPE html>
<html lang="{$language}" dir="{$direction}">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">

    <title>{$headtitle} | {$companyname}</title>

    {* Meta tags *}
    <meta name="description" content="{$page_description}">
    <meta name="keywords" content="{$page_keywords}">

    {* Favicon *}
    <link rel="icon" type="image/x-icon" href="{$baseUrl}/assets/images/favicon.ico">

    {* Preload critical assets *}
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>

    {* Fonts *}
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet">

    {* Main stylesheet *}
    <link href="{$baseUrl}/assets/css/main.min.css" rel="stylesheet">

    {* Responsive styles *}
    <link href="{$baseUrl}/assets/css/responsive.min.css" rel="stylesheet">

    {* Custom CSS from template settings *}
    {if $templateconfig.customCSS}
    <style>
        {$templateconfig.customCSS}
    </style>
    {/if}

    {* Template-specific CSS *}
    {$templatefile_css}

    {* Hook for additional head content *}
    {hook name="head_output"}

    {$jsCode}
</head>
<body class="page-{$pagtemplate}">

    {* Skip to content link for accessibility *}
    <a href="#main-content" class="sr-only sr-only-focusable">
        Skip to main content
    </a>

    {* Top bar *}
    <div class="top-bar bg-dark text-light">
        <div class="container">
            <div class="row">
                <div class="col-md-6">
                    <ul class="list-inline mb-0">
                        <li>
                            <i class="fa fa-phone"></i>
                            <a href="tel:{$companyphone}" class="text-light">{$companyphone}</a>
                        </li>
                        <li>
                            <i class="fa fa-envelope"></i>
                            <a href="mailto:{$companyemail}" class="text-light">{$companyemail}</a>
                        </li>
                    </ul>
                </div>
                <div class="col-md-6 text-md-right">
                    {if $loggedin}
                        <a href="{$smarty.server.SCRIPT_NAME}?m=account" class="text-light">
                            <i class="fa fa-user"></i> {$clientname}
                        </a>
                        <span class="text-muted">|</span>
                        <a href="{$smarty.server.SCRIPT_NAME}?m=logout" class="text-light">
                            <i class="fa fa-sign-out"></i> {$LANG.logout}
                        </a>
                    {else}
                        <a href="{$smarty.server.SCRIPT_NAME}?m=login" class="text-light">
                            <i class="fa fa-sign-in"></i> {$LANG.login}
                        </a>
                        <span class="text-muted">|</span>
                        <a href="{$smarty.server.SCRIPT_NAME}?m=register" class="text-light">
                            <i class="fa fa-user-plus"></i> {$LANG.register}
                        </a>
                    {/if}
                </div>
            </div>
        </div>
    </div>

    {* Main navigation *}
    <nav class="navbar navbar-expand-lg navbar-light bg-white shadow-sm sticky-top">
        <div class="container">
            {* Logo *}
            <a class="navbar-brand" href="{$BASE_PATH_HTTPS}">
                {if $templateconfig.logoUrl}
                    <img src="{$templateconfig.logoUrl}" alt="{$companyname}" height="40">
                {else}
                    <span class="h3 mb-0 text-primary font-weight-bold">{$companyname}</span>
                {/if}
            </a>

            {* Mobile toggle *}
            <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#mainNav">
                <span class="navbar-toggler-icon"></span>
            </button>

            {* Navigation links *}
            <div class="collapse navbar-collapse" id="mainNav">
                <ul class="navbar-nav ml-auto">
                    <li class="nav-item">
                        <a class="nav-link" href="{$BASE_PATH_HTTPS}">
                            <i class="fa fa-home"></i> {$LANG.home}
                        </a>
                    </li>
                    <li class="nav-item dropdown">
                        <a class="nav-link dropdown-toggle" href="#" data-toggle="dropdown">
                            {$LANG.navservices}
                        </a>
                        <div class="dropdown-menu">
                            <a class="dropdown-item" href="{$smarty.server.SCRIPT_NAME}?m=cart">{$LANG.order}</a>
                            <a class="dropdown-item" href="{$smarty.server.SCRIPT_NAME}?m=domainchecker">{$LANG.domainchecker}</a>
                            <a class="dropdown-item" href="{$smarty.server.SCRIPT_NAME}?m=cart&a=view">{$LANG.viewcart}</a>
                        </div>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{$smarty.server.SCRIPT_NAME}?m=supporttickets">
                            {$LANG.navtickets}
                        </a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{$smarty.server.SCRIPT_NAME}?m=knowledgebase">
                            {$LANG.knowledgebase}
                        </a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{$smarty.server.SCRIPT_NAME}?m=contact">
                            {$LANG.contactus}
                        </a>
                    </li>
                </ul>
            </div>
        </div>
    </nav>

    {* Breadcrumbs *}
    {if $breadcrumbs && count($breadcrumbs) > 0}
    <div class="breadcrumbs-bar">
        <div class="container">
            <ol class="breadcrumb mb-0">
                <li class="breadcrumb-item">
                    <a href="{$BASE_PATH_HTTPS}">{$LANG.home}</a>
                </li>
                {foreach $breadcrumbs as $crumb}
                    <li class="breadcrumb-item {if $crumb@last}active{/if}">
                        {if $crumb@last}
                            {$crumb.title}
                        {else}
                            <a href="{$crumb.link}">{$crumb.title}</a>
                        {/if}
                    </li>
                {/foreach}
            </ol>
        </div>
    </div>
    {/if}

    {* Main content area *}
    <main id="main-content" class="main-content">
```

## Footer Template

```smarty
{* /templates/your_template/includes/footer.tpl *}

    </main>

    {* Footer *}
    <footer class="footer bg-dark text-light pt-5 pb-3">
        <div class="container">
            <div class="row">
                {* About column *}
                <div class="col-lg-4 mb-4">
                    <h5>{$companyname}</h5>
                    <p class="text-muted">
                        {$LANG.footerabouttext}
                    </p>
                    <div class="social-links">
                        <a href="#" class="text-light mr-3"><i class="fa fa-facebook"></i></a>
                        <a href="#" class="text-light mr-3"><i class="fa fa-twitter"></i></a>
                        <a href="#" class="text-light mr-3"><i class="fa fa-linkedin"></i></a>
                        <a href="#" class="text-light"><i class="fa fa-github"></i></a>
                    </div>
                </div>

                {* Services column *}
                <div class="col-lg-2 col-md-4 mb-4">
                    <h6>{$LANG.services}</h6>
                    <ul class="list-unstyled">
                        <li><a href="{$smarty.server.SCRIPT_NAME}?m=cart" class="text-muted">{$LANG.order}</a></li>
                        <li><a href="{$smarty.server.SCRIPT_NAME}?m=cart&a=domainchecker" class="text-muted">Domains</a></li>
                        <li><a href="{$smarty.server.SCRIPT_NAME}?m=supporttickets" class="text-muted">Support</a></li>
                    </ul>
                </div>

                {* Account column *}
                <div class="col-lg-2 col-md-4 mb-4">
                    <h6>{$LANG.account}</h6>
                    <ul class="list-unstyled">
                        {if $loggedin}
                            <li><a href="{$smarty.server.SCRIPT_NAME}?m=account" class="text-muted">{$LANG.clientarea}</a></li>
                            <li><a href="{$smarty.server.SCRIPT_NAME}?m=invoices" class="text-muted">{$LANG.invoices}</a></li>
                            <li><a href="{$smarty.server.SCRIPT_NAME}?m=supporttickets&action=open" class="text-muted">{$LANG.submitticket}</a></li>
                        {else}
                            <li><a href="{$smarty.server.SCRIPT_NAME}?m=login" class="text-muted">{$LANG.login}</a></li>
                            <li><a href="{$smarty.server.SCRIPT_NAME}?m=register" class="text-muted">{$LANG.register}</a></li>
                        {/if}
                    </ul>
                </div>

                {* Legal column *}
                <div class="col-lg-2 col-md-4 mb-4">
                    <h6>{$LANG.legal}</h6>
                    <ul class="list-unstyled">
                        <li><a href="{$smarty.server.SCRIPT_NAME}?m=tos" class="text-muted">{$LANG.terms}</a></li>
                        <li><a href="{$smarty.server.SCRIPT_NAME}?m=privacy" class="text-muted">{$LANG.privacy}</a></li>
                        <li><a href="{$smarty.server.SCRIPT_NAME}?m=refundpolicy" class="text-muted">{$LANG.refundpolicy}</a></li>
                    </ul>
                </div>

                {* Contact column *}
                <div class="col-lg-2 mb-4">
                    <h6>{$LANG.contactus}</h6>
                    <ul class="list-unstyled text-muted">
                        <li><i class="fa fa-phone"></i> {$companyphone}</li>
                        <li><i class="fa fa-envelope"></i> {$companyemail}</li>
                    </ul>
                </div>
            </div>

            {* Payment methods *}
            <div class="payment-methods text-center mt-4 mb-4">
                <span class="text-muted mr-2">We accept:</span>
                <i class="fa fa-cc-visa fa-2x mx-1"></i>
                <i class="fa fa-cc-mastercard fa-2x mx-1"></i>
                <i class="fa fa-cc-amex fa-2x mx-1"></i>
                <i class="fa fa-cc-paypal fa-2x mx-1"></i>
            </div>

            {* Copyright *}
            <div class="text-center text-muted">
                <p>&copy; {date('Y')} {$companyname}. {$LANG.footercopyright}</p>
            </div>
        </div>
    </footer>

    {* Back to top button *}
    <a href="#" class="back-to-top" id="backToTop">
        <i class="fa fa-chevron-up"></i>
    </a>

    {* JavaScript libraries *}
    <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@4.6.0/dist/js/bootstrap.bundle.min.js"></script>

    {* Template JavaScript *}
    <script src="{$baseUrl}/assets/js/main.min.js"></script>

    {* Page-specific JavaScript *}
    {$templatefile_js}

    {* Hook for additional footer content *}
    {hook name="footer_output"}
</body>
</html>
```

## Client Area Homepage

```smarty
{* /templates/your_template/client/home.tpl *}
{extends file="$layout"}

{block name="content"}
<div class="client-home">
    <div class="row">
        {* Welcome message *}
        <div class="col-12">
            <div class="card mb-4">
                <div class="card-body">
                    <h2>{$LANG.welcome}, {$client.firstname}!</h2>
                    <p class="text-muted">
                        {$LANG.lastlogin}: {$client.lastlogin|date_format}
                    </p>
                </div>
            </div>
        </div>
    </div>

    <div class="row">
        {* Quick stats *}
        <div class="col-md-3">
            <div class="card stat-card">
                <div class="card-body text-center">
                    <i class="fa fa-server fa-2x text-primary"></i>
                    <h3>{$stats.services_active}</h3>
                    <p>{$LANG.services}</p>
                </div>
            </div>
        </div>

        <div class="col-md-3">
            <div class="card stat-card">
                <div class="card-body text-center">
                    <i class="fa fa-file-text fa-2x text-success"></i>
                    <h3>{$stats.invoices_unpaid}</h3>
                    <p>{$LANG.openinvoices}</p>
                </div>
            </div>
        </div>

        <div class="col-md-3">
            <div class="card stat-card">
                <div class="card-body text-center">
                    <i class="fa fa-ticket fa-2x text-warning"></i>
                    <h3>{$stats.tickets_open}</h3>
                    <p>{$LANG.supporttickets}</p>
                </div>
            </div>
        </div>

        <div class="col-md-3">
            <div class="card stat-card">
                <div class="card-body text-center">
                    <i class="fa fa-shopping-cart fa-2x text-info"></i>
                    <h3>{$stats.orders_pending}</h3>
                    <p>{$LANG.pendingorders}</p>
                </div>
            </div>
        </div>
    </div>

    <div class="row mt-4">
        {* Active services *}
        <div class="col-lg-6">
            <div class="card">
                <div class="card-header">
                    <h5 class="mb-0">{$LANG.yourservices}</h5>
                </div>
                <div class="card-body p-0">
                    <div class="table-responsive">
                        <table class="table mb-0">
                            <thead>
                                <tr>
                                    <th>{$LANG.domain}</th>
                                    <th>{$LANG.product}</th>
                                    <th>{$LANG.billingcycle}</th>
                                    <th>{$LANG.nextduedate}</th>
                                    <th>{$LANG.status}</th>
                                </tr>
                            </thead>
                            <tbody>
                                {foreach $services as $service}
                                <tr>
                                    <td>
                                        <a href="{$service.detailsUrl}">{$service.domain}</a>
                                    </td>
                                    <td>{$service.product}</td>
                                    <td>{$service.billingcycle}</td>
                                    <td>{$service.nextduedate|date_format}</td>
                                    <td>
                                        <span class="badge badge-{$service.statusClass}">
                                            {$service.status}
                                        </span>
                                    </td>
                                </tr>
                                {foreachelse}
                                <tr>
                                    <td colspan="5" class="text-center text-muted">
                                        {$LANG.norecords}
                                    </td>
                                </tr>
                                {/foreach}
                            </tbody>
                        </table>
                    </div>
                </div>
            </div>
        </div>

        {* Recent invoices *}
        <div class="col-lg-6">
            <div class="card">
                <div class="card-header">
                    <h5 class="mb-0">{$LANG.recentinvoices}</h5>
                </div>
                <div class="card-body p-0">
                    <div class="table-responsive">
                        <table class="table mb-0">
                            <thead>
                                <tr>
                                    <th>{$LANG.invoicenumber}</th>
                                    <th>{$LANG.date}</th>
                                    <th class="text-right">{$LANG.amount}</th>
                                    <th>{$LANG.status}</th>
                                </tr>
                            </thead>
                            <tbody>
                                {foreach $invoices as $invoice}
                                <tr>
                                    <td>
                                        <a href="{$invoice.viewUrl}">{$invoice.invoicenum}</a>
                                    </td>
                                    <td>{$invoice.date|date_format}</td>
                                    <td class="text-right">{$invoice.total}</td>
                                    <td>
                                        <span class="badge badge-{$invoice.statusClass}">
                                            {$invoice.status}
                                        </span>
                                    </td>
                                </tr>
                                {foreachelse}
                                <tr>
                                    <td colspan="4" class="text-center text-muted">
                                        {$LANG.norecords}
                                    </td>
                                </tr>
                                {/foreach}
                            </tbody>
                        </table>
                    </div>
                </div>
            </div>
        </div>
    </div>

    {* Open tickets *}
    {if $tickets}
    <div class="row mt-4">
        <div class="col-12">
            <div class="card">
                <div class="card-header">
                    <h5 class="mb-0">{$LANG.yourtickets}</h5>
                </div>
                <div class="card-body p-0">
                    <div class="table-responsive">
                        <table class="table mb-0">
                            <thead>
                                <tr>
                                    <th>ID</th>
                                    <th>{$LANG.subject}</th>
                                    <th>{$LANG.department}</th>
                                    <th>{$LANG.priority}</th>
                                    <th>{$LANG.lastreply}</th>
                                    <th>{$LANG.status}</th>
                                </tr>
                            </thead>
                            <tbody>
                                {foreach $tickets as $ticket}
                                <tr>
                                    <td>#{$ticket.id}</td>
                                    <td>
                                        <a href="{$ticket.viewUrl}">{$ticket.subject}</a>
                                    </td>
                                    <td>{$ticket.department}</td>
                                    <td>
                                        <span class="badge badge-{$ticket.priorityClass}">
                                            {$ticket.priority}
                                        </span>
                                    </td>
                                    <td>{$ticket.lastreply|date_format}</td>
                                    <td>
                                        <span class="badge badge-{$ticket.statusClass}">
                                            {$ticket.status}
                                        </span>
                                    </td>
                                </tr>
                                {/foreach}
                            </tbody>
                        </table>
                    </div>
                </div>
            </div>
        </div>
    </div>
    {/if}
</div>
{/block}
```

## Main JavaScript

```javascript
// /templates/your_template/assets/js/main.js

(function() {
    'use strict';

    // Initialize when DOM is ready
    document.addEventListener('DOMContentLoaded', function() {
        initBackToTop();
        initSmoothScroll();
        initFormValidation();
        initAjaxForms();
        initTooltips();
        initConfirmDialogs();
    });

    // Back to top button
    function initBackToTop() {
        const btn = document.getElementById('backToTop');
        if (!btn) return;

        window.addEventListener('scroll', function() {
            if (window.pageYOffset > 300) {
                btn.classList.add('show');
            } else {
                btn.classList.remove('show');
            }
        });

        btn.addEventListener('click', function(e) {
            e.preventDefault();
            window.scrollTo({ top: 0, behavior: 'smooth' });
        });
    }

    // Smooth scroll for anchor links
    function initSmoothScroll() {
        document.querySelectorAll('a[href^="#"]').forEach(function(anchor) {
            anchor.addEventListener('click', function(e) {
                const target = document.querySelector(this.getAttribute('href'));
                if (target) {
                    e.preventDefault();
                    target.scrollIntoView({ behavior: 'smooth' });
                }
            });
        });
    }

    // Form validation
    function initFormValidation() {
        const forms = document.querySelectorAll('form[data-validate]');

        forms.forEach(function(form) {
            form.addEventListener('submit', function(e) {
                if (!validateForm(form)) {
                    e.preventDefault();
                    return false;
                }
            });
        });
    }

    function validateForm(form) {
        let isValid = true;
        const requiredFields = form.querySelectorAll('[required]');

        requiredFields.forEach(function(field) {
            if (!field.value.trim()) {
                isValid = false;
                field.classList.add('is-invalid');
            } else {
                field.classList.remove('is-invalid');
            }
        });

        return isValid;
    }

    // AJAX form submission
    function initAjaxForms() {
        document.querySelectorAll('form[data-ajax]').forEach(function(form) {
            form.addEventListener('submit', function(e) {
                e.preventDefault();

                const formData = new FormData(form);
                const url = form.action || window.location.href;

                fetch(url, {
                    method: 'POST',
                    body: formData,
                    headers: {
                        'X-Requested-With': 'XMLHttpRequest',
                    },
                })
                .then(function(response) {
                    return response.json();
                })
                .then(function(data) {
                    if (data.success) {
                        showNotification('success', data.message || 'Success!');
                        if (data.redirect) {
                            window.location.href = data.redirect;
                        }
                    } else {
                        showNotification('error', data.error || 'An error occurred.');
                    }
                })
                .catch(function(error) {
                    showNotification('error', 'An error occurred. Please try again.');
                });
            });
        });
    }

    // Bootstrap tooltips
    function initTooltips() {
        if (typeof bootstrap !== 'undefined') {
            const tooltips = document.querySelectorAll('[data-toggle="tooltip"]');
            tooltips.forEach(function(el) {
                new bootstrap.Tooltip(el);
            });
        }
    }

    // Confirmation dialogs
    function initConfirmDialogs() {
        document.querySelectorAll('[data-confirm]').forEach(function(el) {
            el.addEventListener('click', function(e) {
                const message = el.getAttribute('data-confirm') || 'Are you sure?';
                if (!confirm(message)) {
                    e.preventDefault();
                    return false;
                }
            });
        });
    }

    // Show notification
    function showNotification(type, message) {
        const alertClass = type === 'success' ? 'alert-success' : 'alert-danger';
        const html = '<div class="alert ' + alertClass + ' alert-dismissible fade show" role="alert">' +
            message +
            '<button type="button" class="close" data-dismiss="alert"><span>&times;</span></button>' +
            '</div>';

        const container = document.querySelector('.notifications') || document.body;
        const notification = document.createElement('div');
        notification.innerHTML = html;
        container.prepend(notification);

        setTimeout(function() {
            notification.querySelector('.alert').remove();
        }, 5000);
    }

    // Global WHMCS namespace
    window.WhmcsApi = {
        ajax: function(url, options) {
            return fetch(url, Object.assign({
                method: 'POST',
                headers: {
                    'X-Requested-With': 'XMLHttpRequest',
                    'Content-Type': 'application/x-www-form-urlencoded',
                },
            }, options));
        },
        notify: showNotification,
    };

})();
```

## Best Practices

1. **Responsive Design**: Always use responsive layouts
2. **Accessibility**: Include skip links and ARIA attributes
3. **Performance**: Minify CSS/JS and use lazy loading
4. **SEO**: Use semantic HTML and proper meta tags
5. **Template Config**: Use template settings for customization
6. **Smarty Best Practices**: Use blocks and template inheritance
7. **Hooks**: Integrate with WHMCS hooks for extensibility
8. **RTL Support**: Design for RTL languages when needed
9. **Dark Mode**: Consider dark mode support
10. **Browser Support**: Test across modern browsers
