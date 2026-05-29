# WHMCS Font Awesome Integration

## Overview

Font Awesome icons are integrated throughout WHMCS for consistent iconography. This reference covers usage in templates and customization.

## Version

WHMCS includes Font Awesome 6 (latest). Check your version by examining the included file.

## Basic Usage

### In HTML

```html
<i class="fa-solid fa-server"></i>
<i class="fa-regular fa-user"></i>
<i class="fa-brands fa-wordpress"></i>
```

### In Smarty Templates

```smarty
<i class="fa-solid fa-server"></i>
<i class="fa-regular fa-user"></i>
<i class="fa-brands fa-wordpress"></i>
```

### With Text

```smarty
<a href="#">
    <i class="fa-solid fa-home"></i> Home
</a>

<button type="button">
    <i class="fa-solid fa-plus"></i> Add New
</button>
```

## Icon Styles

### Solid (Default)

```smarty
<i class="fa-solid fa-server"></i>
<i class="fa-solid fa-user"></i>
<i class="fa-solid fa-gear"></i>
<i class="fa-solid fa-envelope"></i>
<i class="fa-solid fa-cart-shopping"></i>
```

### Regular

```smarty
<i class="fa-regular fa-user"></i>
<i class="fa-regular fa-envelope"></i>
<i class="fa-regular fa-bookmark"></i>
<i class="fa-regular fa-bell"></i>
<i class="fa-regular fa-calendar"></i>
```

### Brands

```smarty
<i class="fa-brands fa-wordpress"></i>
<i class="fa-brands fa-whmcs"></i>
<i class="fa-brands fa-stripe"></i>
<i class="fa-brands fa-paypal"></i>
<i class="fa-brands fa-twitter"></i>
```

## Common WHMCS Icons

### Navigation

```smarty
<i class="fa-solid fa-home"></i>              {* Home *}
<i class="fa-solid fa-gear"></i>              {* Settings *}
<i class="fa-solid fa-user"></i>              {* User/Account *}
<i class="fa-solid fa-right-to-bracket"></i> {* Login *}
<i class="fa-solid fa-right-from-bracket"></i>{* Logout *}
<i class="fa-solid fa-bars"></i>              {* Menu *}
```

### Products & Services

```smarty
<i class="fa-solid fa-server"></i>           {* Hosting *}
<i class="fa-solid fa-desktop"></i>          {* VPS/Dedicated *}
<i class="fa-solid fa-cloud"></i>             {* Cloud *}
<i class="fa-solid fa-box"></i>               {* Products *}
<i class="fa-solid fa-cart-shopping"></i>     {* Cart *}
<i class="fa-solid fa-credit-card"></i>       {* Payment *}
```

### Support

```smarty
<i class="fa-solid fa-ticket"></i>           {* Tickets *}
<i class="fa-solid fa-headset"></i>           {* Support *}
<i class="fa-solid fa-circle-question"></i>  {* Help *}
<i class="fa-solid fa-message"></i>           {* Messages *}
<i class="fa-solid fa-phone"></i>             {* Phone *}
```

### Status

```smarty
<i class="fa-solid fa-check"></i>            {* Success/Active *}
<i class="fa-solid fa-xmark"></i>             {* Error/Close *}
<i class="fa-solid fa-exclamation"></i>       {* Warning *}
<i class="fa-solid fa-circle-info"></i>      {* Info *}
<i class="fa-solid fa-clock"></i>            {* Pending *}
<i class="fa-solid fa-spinner"></i>          {* Loading *}
```

### File & Data

```smarty
<i class="fa-solid fa-file"></i>             {* File *}
<i class="fa-solid fa-file-lines"></i>       {* Document *}
<i class="fa-solid fa-invoice-dollar"></i>    {* Invoice *}
<i class="fa-solid fa-chart-line"></i>        {* Statistics *}
<i class="fa-solid fa-download"></i>          {* Download *}
<i class="fa-solid fa-upload"></i>            {* Upload *}
```

### Domain

```smarty
<i class="fa-solid fa-globe"></i>            {* Domain *}
<i class="fa-solid fa-earth-americas"></i>   {* World *}
<i class="fa-solid fa-link"></i>              {* URL *}
<i class="fa-solid fa-at"></i>                {* Email *}
```

## Sizing

```smarty
<i class="fa-solid fa-server fa-xs"></i>
<i class="fa-solid fa-server fa-sm"></i>
<i class="fa-solid fa-server"></i>            {* Default *}
<i class="fa-solid fa-server fa-lg"></i>
<i class="fa-solid fa-server fa-xl"></i>
<i class="fa-solid fa-server fa-2xl"></i>
<i class="fa-solid fa-server fa-3xl"></i>
<i class="fa-solid fa-server fa-5xl"></i>
<i class="fa-solid fa-server fa-10xl"></i>
```

## Fixed Width

```smarty
<div class="fa-ul">
    <li><i class="fa-solid fa-home fa-fw"></i> Home</li>
    <li><i class="fa-solid fa-user fa-fw"></i> Profile</li>
    <li><i class="fa-solid fa-gear fa-fw"></i> Settings</li>
</div>
```

## Animations

### Spin

```smarty
<i class="fa-solid fa-spinner fa-spin"></i>
<i class="fa-solid fa-circle-notch fa-spin"></i>
<i class="fa-solid fa-sync fa-spin"></i>
<i class="fa-solid fa-server fa-spin"></i>
```

### Pulse

```smarty
<i class="fa-solid fa-circle-notch fa-pulse"></i>
<i class="fa-solid fa-heart fa-pulse"></i>
```

### Rotate/Flip

```smarty
<i class="fa-solid fa-home fa-rotate-90"></i>
<i class="fa-solid fa-home fa-rotate-180"></i>
<i class="fa-solid fa-home fa-rotate-270"></i>
<i class="fa-solid fa-home fa-flip-horizontal"></i>
<i class="fa-solid fa-home fa-flip-vertical"></i>
```

## Borders & Pull

```smarty
<i class="fa-solid fa-quote-left fa-border"></i>

<p><i class="fa-solid fa-quote-left fa-pull-left"></i> 
    Text wrapping around the icon.
</p>
```

## Layered Icons

### Badge on Icon

```smarty
<span class="fa-layers fa-fw">
    <i class="fa-solid fa-envelope"></i>
    <span class="fa-layers-counter">3</span>
</span>
```

### Custom Colors

```smarty
<span class="fa-layers fa-fw">
    <i class="fa-solid fa-square text-primary"></i>
    <i class="fa-solid fa-check text-white" 
       data-fa-transform="shrink-6"></i>
</span>
```

## Icon Display

### Using {icon} Tag (WHMCS)

```smarty
{icon type="server"}
{icon type="user"}
{icon type="ticket"}
{icon type="invoice"}
```

### With Style Attribute

```smarty
<i class="fa-solid fa-server" style="font-size: 24px; color: #007bff;"></i>
```

## WHMCS-Specific Usage

### Sidebar Navigation

```smarty
<ul class="nav nav-pills">
    <li>
        <a href="clientarea.php">
            <i class="fa-solid fa-dashboard"></i> Dashboard
        </a>
    </li>
    <li>
        <a href="clientarea.php?action=services">
            <i class="fa-solid fa-server"></i> Services
        </a>
    </li>
    <li>
        <a href="clientarea.php?action=domains">
            <i class="fa-solid fa-globe"></i> Domains
        </a>
    </li>
    <li>
        <a href="clientarea.php?action=invoices">
            <i class="fa-solid fa-file-invoice-dollar"></i> Invoices
        </a>
    </li>
</ul>
```

### Status Indicators

```smarty
<span class="service-status">
    {if $service->status == 'Active'}
        <i class="fa-solid fa-check-circle text-success"></i>
    {elseif $service->status == 'Suspended'}
        <i class="fa-solid fa-pause-circle text-warning"></i>
    {elseif $service->status == 'Terminated'}
        <i class="fa-solid fa-xmark-circle text-danger"></i>
    {else}
        <i class="fa-solid fa-clock text-muted"></i>
    {/if}
    {$service->status}
</span>
```

### Action Buttons

```smarty
<div class="btn-group">
    <button type="button" class="btn btn-default">
        <i class="fa-solid fa-pencil"></i> Edit
    </button>
    <button type="button" class="btn btn-info">
        <i class="fa-solid fa-eye"></i> View
    </button>
    <button type="button" class="btn btn-danger">
        <i class="fa-solid fa-trash"></i> Delete
    </button>
</div>
```

## Customization

### Using Custom Icons

```php
<?php
// Add custom icon set
add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    return '<link rel="stylesheet" href="templates/your-template/custom-icons.css">';
});
```

### Icon Color Classes

```smarty
<i class="fa-solid fa-server text-primary"></i>   {* Blue *}
<i class="fa-solid fa-server text-success"></i>  {* Green *}
<i class="fa-solid fa-server text-info"></i>     {* Light blue *}
<i class="fa-solid fa-server text-warning"></i> {* Yellow *}
<i class="fa-solid fa-server text-danger"></i>  {* Red *}
```

## See Also

- [Bootstrap Components](../whmcs-bootstrap-components.md)
- [Custom CSS](../whmcs-custom-css.md)
- [Client Area Templates](../whmcs-clientarea-templates.md)