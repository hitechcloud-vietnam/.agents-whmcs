# WHMCS Client Area

## Overview

The WHMCS client area provides the interface for customers to manage their accounts, services, invoices, and support.

## Client Area Structure

```
templates/
├── six/                     # Default template
│   ├── clienthome.tpl
│   ├── clientareahome.tpl
│   ├── viewinvoice.tpl
│   └── ...
└── your-template/
    ├── clientarea.tpl      # Main wrapper
    ├── header.tpl
    ├── footer.tpl
    └── ...
```

## Client Area Template

```smarty
{include file="$templatefile/header.tpl"}

<div class="client-area">
    <div class="container">
        <div class="row">
            <div class="col-md-3">
                {include file="$templatefile/sidebar.tpl"}
            </div>
            <div class="col-md-9">
                <div class="page-header">
                    <h1>{$pagetitle}</h1>
                </div>
                
                <div class="content">
                    {$content}
                </div>
            </div>
        </div>
    </div>
</div>

{include file="$templatefile/footer.tpl"}
```

## Client Area Variables

```smarty
{* Client information *}
{$client->id}
{$client->email}
{$client->firstname}
{$client->lastname}
{$client->fullName}
{$client->companyname}
{$client->phonenumber}
{$client->address1}
{$client->address2}
{$client->city}
{$client->state}
{$client->postcode}
{$client->country}
{$client->status}

{* Session information *}
{$loggedin}
{$client->uid}

{* Breadcrumbs *}
{$breadcrumb}
```

## Navigation Menu

```smarty
{if $loggedin}
    <nav class="client-nav">
        <ul class="nav navbar-nav">
            <li>
                <a href="clientarea.php">
                    <i class="fa fa-home"></i> Home
                </a>
            </li>
            <li class="dropdown">
                <a href="#" class="dropdown-toggle" data-toggle="dropdown">
                    Services <span class="caret"></span>
                </a>
                <ul class="dropdown-menu">
                    <li><a href="clientarea.php?action=services">My Services</a></li>
                    <li><a href="clientarea.php?action=domains">Domains</a></li>
                    <li><a href="clientarea.php?action=addons">Addons</a></li>
                </ul>
            </li>
            <li>
                <a href="clientarea.php?action=billing">
                    <i class="fa fa-credit-card"></i> Billing
                </a>
            </li>
            <li>
                <a href="supporttickets.php">
                    <i class="fa fa-life-ring"></i> Support
                </a>
            </li>
        </ul>
    </nav>
{/if}
```

## Service List

```smarty
<div class="services-list">
    {foreach from=$services item=service}
        <div class="service-card panel panel-default">
            <div class="panel-heading">
                <h4>{$service->domain}</h4>
                <span class="label label-{$service->status|lower}">
                    {$service->status}
                </span>
            </div>
            <div class="panel-body">
                <div class="row">
                    <div class="col-md-4">
                        <strong>Product:</strong><br>
                        {$service->product->name}
                    </div>
                    <div class="col-md-4">
                        <strong>Next Due:</strong><br>
                        {$service->nextduedate|date_format}
                    </div>
                    <div class="col-md-4">
                        <strong>Amount:</strong><br>
                        {$service->amount|currency}
                    </div>
                </div>
            </div>
            <div class="panel-footer">
                <a href="clientarea.php?action=productdetails&id={$service->id}" 
                   class="btn btn-sm btn-default">
                    View Details
                </a>
                <a href="clientarea.php?action=manage&id={$service->id}" 
                   class="btn btn-sm btn-primary">
                    Manage
                </a>
            </div>
        </div>
    {/foreach}
</div>
```

## Client Area Pages

### Home Page (clientareahome.tpl)

```smarty
<div class="welcome-banner">
    <h2>Welcome back, {$client->firstname}!</h2>
</div>

<div class="dashboard-stats row">
    <div class="col-md-3">
        <div class="stat-card">
            <div class="stat-value">{$serviceCount}</div>
            <div class="stat-label">Active Services</div>
        </div>
    </div>
    <div class="col-md-3">
        <div class="stat-card">
            <div class="stat-value">{$domainCount}</div>
            <div class="stat-label">Domains</div>
        </div>
    </div>
    <div class="col-md-3">
        <div class="stat-card">
            <div class="stat-value">{$openTickets}</div>
            <div class="stat-label">Open Tickets</div>
        </div>
    </div>
    <div class="col-md-3">
        <div class="stat-card">
            <div class="stat-value">{$creditBalance|currency}</div>
            <div class="stat-label">Account Credit</div>
        </div>
    </div>
</div>

<div class="row">
    <div class="col-md-6">
        <div class="panel panel-default">
            <div class="panel-heading">Recent Invoices</div>
            <div class="panel-body">
                {include file="$templatefile/invoices-list.tpl"}
            </div>
        </div>
    </div>
    <div class="col-md-6">
        <div class="panel panel-default">
            <div class="panel-heading">Recent Tickets</div>
            <div class="panel-body">
                {include file="$templatefile/tickets-list.tpl"}
            </div>
        </div>
    </div>
</div>
```

### Product Details

```smarty
<div class="product-details">
    <div class="header-section">
        <h2>{$product->name}</h2>
        <span class="status-badge status-{$product->status|lower}">
            {$product->status}
        </span>
    </div>
    
    <div class="product-info panel panel-default">
        <div class="panel-body">
            <div class="info-row">
                <label>Domain:</label>
                <span>{$product->domain}</span>
            </div>
            <div class="info-row">
                <label>Username:</label>
                <span>{$product->username}</span>
            </div>
            <div class="info-row">
                <label>Next Due Date:</label>
                <span>{$product->nextduedate|date_format}</span>
            </div>
            <div class="info-row">
                <label>Recurring Amount:</label>
                <span>{$product->amount|currency}</span>
            </div>
        </div>
    </div>
    
    <div class="actions">
        <a href="clientarea.php?action=manage&id={$product->id}" 
           class="btn btn-primary">
            Manage Service
        </a>
        <a href="clientarea.php?action=upgrade&id={$product->id}" 
           class="btn btn-default">
            Upgrade/Downgrade
        </a>
    </div>
</div>
```

## Custom Client Area Page

```php
<?php
// In addon module
function yourmodule_clientarea(array $vars): array
{
    global $smarty;
    
    $userId = (int) $_SESSION['uid'];
    
    // Check if logged in
    if (!$userId) {
        return [
            'templatefile' => 'login-required',
            'vars' => [
                'redirect' => $_SERVER['REQUEST_URI'],
            ],
        ];
    }
    
    // Get user data
    $data = getModuleDataForClient($userId);
    
    return [
        'pagetitle' => 'My Module Data',
        'templatefile' => 'clientarea',
        'vars' => [
            'moduleData' => $data,
            'user' => getClientsDetails($userId),
        ],
    ];
}
```

```smarty
{include file="$templatefile/header.tpl"}

<div class="client-module-page">
    <h1>My Module Data</h1>
    
    <div class="data-list">
        {foreach from=$moduleData item=item}
            <div class="data-item">
                <h3>{$item.title}</h3>
                <p>{$item.description}</p>
                <a href="?action=view&id={$item.id}">View Details</a>
            </div>
        {foreachelse}
            <p>No data found.</p>
        {/foreach}
    </div>
</div>

{include file="$templatefile/footer.tpl"}
```

## Best Practices

1. **Check authentication** - Always verify user is logged in
2. **Use existing styles** - Match WHMCS design
3. **Support mobile** - Responsive design
4. **Include navigation** - Easy access to all sections
5. **Show breadcrumbs** - Help with navigation

## Related Documentation

- [WHMCS Smarty Templates](/docs/whmcs-smarty-templates.md)
- [WHMCS CSS Customization](/docs/whmcs-css-customization.md)