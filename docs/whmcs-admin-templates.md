# WHMCS Admin Templates

## Overview

Admin templates control the appearance of the WHMCS admin area. They follow a similar structure to client area templates but with additional administrative functionality.

## Template Structure

### Directory Layout

```
admin/templates/
    blank/
    default/
        header.tpl
        footer.tpl
        sidebar.tpl
        ...
```

### Main Admin Files

| File | Purpose |
|------|---------|
| `header.tpl` | Admin area header |
| `footer.tpl` | Admin area footer |
| `sidebar.tpl` | Navigation sidebar |
| `login.tpl` | Admin login page |

## Admin Variables

### Global Admin Variables

```smarty
{$admin_logged_in}       {* Boolean *}
{$admin_username}        {* Current admin username *}
{$admin_id}              {* Admin ID *}
{$admin_permissions}     {* Array of permissions *}
{$admin_two_factor}       {* 2FA enabled *}
```

### Page-Specific Variables

```smarty
{$displayString}         {* Page heading *}
{$breadcrumb}             {* Navigation breadcrumb *}
{$datepicker}             {* Date picker assets *}
{$clientssel}             {* Client selector *}
{$infobox}                {* Info box content *}
```

## Admin Header

### Standard Header

```smarty
{*
    header.tpl
*}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{if $displayTitle}{$displayTitle}{else}{$displayString}{/if} - {$companyname}</title>
    
    <link rel="stylesheet" href="{$BASE_PATH}/css/bootstrap.min.css">
    <link rel="stylesheet" href="{$BASE_PATH}/css/font-awesome.min.css">
    <link rel="stylesheet" href="templates/default/style.css">
    
    {include file="$template/includes.tpl"}
</head>
<body>
    <nav class="navbar navbar-default">
        <div class="navbar-header">
            <a class="navbar-brand" href="index.php">
                WHMCS Admin
            </a>
        </div>
        
        <ul class="nav navbar-nav">
            <li><a href="index.php">Home</a></li>
            {* Dynamic menu items *}
        </ul>
        
        <ul class="nav navbar-nav navbar-right">
            <li class="dropdown">
                <a href="#" class="dropdown-toggle" data-toggle="dropdown">
                    {$admin_username} <span class="caret"></span>
                </a>
                <ul class="dropdown-menu">
                    <li><a href="myaccount.php">My Account</a></li>
                    <li><a href="logout.php">Logout</a></li>
                </ul>
            </li>
        </ul>
    </nav>
    
    <div class="container-fluid">
        <div class="row">
```

## Admin Sidebar

### Navigation Structure

```smarty
{*
    sidebar.tpl
*}
<div class="sidebar">
    <ul class="nav nav-pills nav-stacked">
        <li class="nav-header">Main</li>
        <li>
            <a href="index.php">
                <i class="fa fa-dashboard"></i> Dashboard
            </a>
        </li>
        
        <li class="nav-header">Clients</li>
        <li>
            <a href="clients.php">
                <i class="fa fa-users"></i> Clients
            </a>
        </li>
        <li>
            <a href="clients.php?action=add">
                <i class="fa fa-user-plus"></i> Add New
            </a>
        </li>
        
        <li class="nav-header">Billing</li>
        <li>
            <a href="invoices.php">
                <i class="fa fa-file-text"></i> Invoices
            </a>
        </li>
        <li>
            <a href="transactions.php">
                <i class="fa fa-exchange"></i> Transactions
            </a>
        </li>
        <li>
            <a href="billableitems.php">
                <i class="fa fa-usd"></i> Billable Items
            </a>
        </li>
        
        <li class="nav-header">Support</li>
        <li>
            <a href="supporttickets.php">
                <i class="fa fa-ticket"></i> Tickets
            </a>
        </li>
        
        <li class="nav-header">Configuration</li>
        <li>
            <a href="configapps.php">
                <i class="fa fa-cogs"></i> Apps & Integrations
            </a>
        </li>
    </ul>
</div>
```

## Admin Footer

```smarty
{*
    footer.tpl
*}
        </div> {* Close row *}
    </div> {* Close container *}
    
    <footer class="admin-footer">
        <p>
            WHMCS Version {$whmcs->getVersion()}
            | <a href="https://docs.whmcs.com" target="_blank">Documentation</a>
        </p>
    </footer>
    
    <script src="{$BASE_PATH}/js/jquery.min.js"></script>
    <script src="{$BASE_PATH}/js/bootstrap.min.js"></script>
    <script src="templates/default/scripts.js"></script>
    
    {include file="$template/includes.tpl"}
</body>
</html>
```

## Admin Login Template

```smarty
{*
    login.tpl
*}
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Admin Login - {$companyname}</title>
    <link rel="stylesheet" href="{$BASE_PATH}/css/login.css">
</head>
<body class="login-page">
    <div class="login-container">
        <div class="login-header">
            <h1>{$companyname}</h1>
            <p>Admin Login</p>
        </div>
        
        <form method="post" action="dologin.php" class="login-form">
            {if $incorrect}
                <div class="alert alert-danger">
                    Invalid username or password
                </div>
            {/if}
            
            <div class="form-group">
                <label for="username">Username</label>
                <input type="text" id="username" name="username" 
                       class="form-control" required autofocus>
            </div>
            
            <div class="form-group">
                <label for="password">Password</label>
                <input type="password" id="password" name="password" 
                       class="form-control" required>
            </div>
            
            <div class="form-group">
                <label>
                    <input type="checkbox" name="rememberme"> 
                    Remember me
                </label>
            </div>
            
            <button type="submit" class="btn btn-primary btn-block">
                Login
            </button>
        </form>
        
        <div class="login-footer">
            <a href="../index.php">Return to Website</a>
        </div>
    </div>
</body>
</html>
```

## Admin Forms

### Standard Form Layout

```smarty
<div class="row">
    <div class="col-md-8 col-md-offset-2">
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">{$displayString}</h3>
            </div>
            <div class="panel-body">
                <form method="post" action="{$smarty.server.PHP_SELF}" 
                      class="form-horizontal">
                    
                    <div class="form-group">
                        <label class="col-sm-3 control-label">
                            Field Label
                        </label>
                        <div class="col-sm-9">
                            <input type="text" name="field" 
                                   class="form-control">
                        </div>
                    </div>
                    
                    <div class="form-group">
                        <div class="col-sm-offset-3 col-sm-9">
                            <button type="submit" class="btn btn-primary">
                                Save Changes
                            </button>
                            <a href="page.php" class="btn btn-default">
                                Cancel
                            </a>
                        </div>
                    </div>
                    
                </form>
            </div>
        </div>
    </div>
</div>
```

## Admin Tables

### Data Table Structure

```smarty
<div class="table-responsive">
    <table class="datatable table table-striped">
        <thead>
            <tr>
                <th>Column 1</th>
                <th>Column 2</th>
                <th>Column 3</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            {foreach $items as $item}
                <tr>
                    <td>{$item->field1}</td>
                    <td>{$item->field2}</td>
                    <td>{$item->field3}</td>
                    <td>
                        <a href="edit.php?id={$item->id}" 
                           class="btn btn-xs btn-default">
                            Edit
                        </a>
                        <a href="delete.php?id={$item->id}" 
                           class="btn btn-xs btn-danger"
                           onclick="return confirm('Are you sure?')">
                            Delete
                        </a>
                    </td>
                </tr>
            {/foreach}
        </tbody>
    </table>
</div>
```

## Hook Integration

### Admin Header Hook

```php
<?php
add_hook('AdminAreaHeaderOutput', 1, function($vars) {
    // Add custom CSS
    return '<link rel="stylesheet" href="custom-admin.css">';
});
```

### Admin Menu Hook

```php
<?php
add_hook('AdminAreaNavTabs', 1, function($vars) {
    return [
        'label' => 'Custom Link',
        'uri' => 'custom.php'
    ];
});
```

## Best Practices

1. **Use Bootstrap** for consistent styling
2. **Include CSRF tokens** in all forms
3. **Use proper permissions** for menu items
4. **Keep admin area clean** and functional
5. **Add proper breadcrumbs** for navigation

## See Also

- [Client Area Templates](../whmcs-clientarea-templates.md)
- [Order Form Templates](../whmcs-orderform-templates.md)
- [Bootstrap Components](../whmcs-bootstrap-components.md)