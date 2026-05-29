# WHMCS Client Area Templates

## Overview

Client area templates control the appearance of the customer-facing pages in WHMCS. These include account pages, service management, invoices, and support tickets.

## Template Structure

### Directory Layout

```
templates/
    six/
        clientarea.tpl
        clientareaheader.tpl
        clientareafooter.tpl
        clientareadomaindetails.tpl
        clientareahome.tpl
        clientarealogin.tpl
        clientareapasswordvalidate.tpl
        clientareaquotas.tpl
        clientareasingup.tpl
        clientareastatus.tpl
        ...
```

### Main Client Area Files

| File | Purpose |
|------|---------|
| `clientarea.tpl` | Main wrapper for all client pages |
| `clientareaheader.tpl` | Header section |
| `clientareafooter.tpl` | Footer section |
| `clientareahome.tpl` | Dashboard/home page |

## Client Area Variables

### Always Available

```smarty
{$loggedin}              {* Boolean: user logged in *}
{$client}                {* Client object *}
{$currency}              {* Current currency *}
{$currencies}            {* Available currencies *}
{$client->id}            {* Client ID *}
{$client->firstname}     {* First name *}
{$client->lastname}      {* Last name *}
{$client->email}         {* Email *}
{$client->credit}        {* Account credit *}
```

### Page-Specific Variables

```smarty
{$services}              {* Array of services *}
{$domains}               {* Array of domains *}
{$invoices}              {* Array of invoices *}
{$tickets}               {* Array of tickets *}
{$quotes}                {* Array of quotes *}
{$affiliate}             {* Affiliate info *}
```

## Common Templates

### Client Home (Dashboard)

```smarty
{* clientareahome.tpl *}
<div class="dashboard">
    <div class="row">
        <div class="col-md-4">
            <div class="stat-box">
                <h3>Services</h3>
                <p class="stat-number">{$services|count}</p>
                <a href="clientarea.php?action=services">View All</a>
            </div>
        </div>
        <div class="col-md-4">
            <div class="stat-box">
                <h3>Domains</h3>
                <p class="stat-number">{$domains|count}</p>
                <a href="clientarea.php?action=domains">View All</a>
            </div>
        </div>
        <div class="col-md-4">
            <div class="stat-box">
                <h3>Account Credit</h3>
                <p class="stat-number">{$client->credit}</p>
                <a href="clientarea.php?action=addfunds">Add Funds</a>
            </div>
        </div>
    </div>
    
    <div class="recent-items">
        <h3>Recent Invoices</h3>
        {include file="$template/includes/table.tpl" 
            items=$invoices 
            fields=['id', 'date', 'total', 'status']
        }
    </div>
</div>
```

### Services List

```smarty
{* clientareaservices.tpl *}
<div class="services-page">
    <h2>Your Products & Services</h2>
    
    <div class="service-list">
        {foreach $services as $service}
            <div class="service-item">
                <div class="service-info">
                    <h4>{$service->product->name}</h4>
                    <p class="domain">{$service->domain}</p>
                    <span class="status status-{$service->status|lower}">
                        {$service->status}
                    </span>
                </div>
                <div class="service-actions">
                    <a href="clientarea.php?action=productdetails&id={$service->id}">
                        Manage
                    </a>
                </div>
                <div class="service-renewal">
                    <span>Next Due: {$service->nextduedate}</span>
                </div>
            </div>
        {/foreach}
    </div>
</div>
```

### Invoice List

```smarty
{* clientareainvoices.tpl *}
<table class="invoice-list">
    <thead>
        <tr>
            <th>Invoice #</th>
            <th>Date</th>
            <th>Due Date</th>
            <th>Amount</th>
            <th>Status</th>
        </tr>
    </thead>
    <tbody>
        {foreach $invoices as $invoice}
            <tr>
                <td>
                    <a href="clientarea.php?action=viewinvoice&id={$invoice->id}">
                        #{$invoice->invoicenum}
                    </a>
                </td>
                <td>{$invoice->date}</td>
                <td>{$invoice->duedate}</td>
                <td>{$invoice->total}</td>
                <td>
                    <span class="label label-{$invoice->status|lower}">
                        {$invoice->status}
                    </span>
                </td>
            </tr>
        {/foreach}
    </tbody>
</table>
```

## Sidebar Navigation

### Default Sidebar Structure

```smarty
<div class="sidebar-nav">
    <ul>
        <li>
            <a href="clientarea.php">
                <i class="fa fa-home"></i> Dashboard
            </a>
        </li>
        <li>
            <a href="clientarea.php?action=services">
                <i class="fa fa-server"></i> Services
            </a>
        </li>
        <li>
            <a href="clientarea.php?action=domains">
                <i class="fa fa-globe"></i> Domains
            </a>
        </li>
        <li>
            <a href="clientarea.php?action=invoices">
                <i class="fa fa-file-text"></i> Invoices
            </a>
        </li>
        <li>
            <a href="supporttickets.php">
                <i class="fa fa-ticket"></i> Support
            </a>
        </li>
    </ul>
</div>
```

## Header and Footer

### Header Template

```smarty
{*
    clientareaheader.tpl
*}
<!DOCTYPE html>
<html lang="{$language}">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{$page_title} | {$company_name}</title>
    
    <link rel="stylesheet" href="{$base_path}/css/bootstrap.min.css">
    <link rel="stylesheet" href="{$template_path}/style.css">
    
    {include file="$template/includes/head.tpl"}
</head>
<body>
    <nav class="navbar navbar-default">
        <div class="container">
            <a class="navbar-brand" href="{$WEB_ROOT}/">
                {$company_name}
            </a>
            
            <div class="nav-collapse">
                {if $loggedin}
                    <span>Welcome, {$client->firstname}</span>
                    <a href="logout.php">Logout</a>
                {else}
                    <a href="login.php">Login</a>
                    <a href="register.php">Register</a>
                {/if}
            </div>
        </div>
    </nav>
    
    <main class="container">
```

### Footer Template

```smarty
{*
    clientareafooter.tpl
*}
    </main>
    
    <footer class="footer">
        <div class="container">
            <p>&copy; {date("Y")} {$company_name}. All rights reserved.</p>
        </div>
    </footer>
    
    <script src="{$base_path}/js/jquery.min.js"></script>
    <script src="{$base_path}/js/bootstrap.min.js"></script>
    
    {include file="$template/includes/scripts.tpl"}
    {hook point="ClientAreaFooterOutput"}
</body>
</html>
```

## Customizing Client Area

### Custom Dashboard

```php
<?php
// Add custom data to client area
add_hook('ClientAreaPageHome', 1, function($vars) {
    return [
        'custom_stats' => [
            'active_services' => countActiveServices($vars['client']->id),
            'total_spend' => getTotalSpend($vars['client']->id)
        ]
    ];
});
```

### Adding to Sidebar

```php
<?php
add_hook('ClientAreaPrimarySidebar', 1, function($vars) {
    $sidebar = $vars['sidebar'];
    
    $customSection = $sidebar->addChild('custom-section', [
        'label' => 'Custom Section',
        'icon' => 'fa-star'
    ]);
    
    $customSection->addChild('custom-link', [
        'label' => 'Custom Link',
        'uri' => 'custom-page.php'
    ]);
});
```

## Best Practices

1. **Use Bootstrap classes** for consistency
2. **Follow naming conventions** for CSS classes
3. **Include hooks** for extensibility
4. **Test responsive design** on all devices
5. **Keep templates modular** with includes

## See Also

- [Admin Templates](../whmcs-admin-templates.md)
- [Order Form Templates](../whmcs-orderform-templates.md)
- [Bootstrap Components](../whmcs-bootstrap-components.md)