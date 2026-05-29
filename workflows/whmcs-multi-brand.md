# WHMCS Multi-Brand Configuration Workflow

## Overview
Configure multiple brands within a single WHMCS installation.

## Prerequisites
- WHMCS v8.0+
- Multi-brand enabled

## Step-by-Step Guide

### Step 1: Brand Configuration
```php
<?php
// Database: tblproductgroups
// Add brand_id column

// Database: tblclients
// Add brand_id column

// Set default brand
function get_client_brand(int $clientId): int
{
    return \WHMCS\Database\Capsule::table('tblclients')
        ->where('id', $clientId)
        ->value('brand_id') ?? 1;
}
```

### Step 2: Brand-Aware Hooks
```php
<?php
add_hook('ClientAreaPage', 1, function($vars) {
    $clientId = \Auth::id();
    $brandId = get_client_brand($clientId);
    $brand = get_brand_config($brandId);
    
    return [
        'brand_name' => $brand['name'],
        'brand_url' => $brand['url'],
        'brand_logo' => $brand['logo'],
        'brand_colors' => $brand['colors'],
    ];
});
```

### Step 3: Product Filtering by Brand
```php
<?php
add_hook('ProductListOutput', 1, function($vars) {
    $brandId = $_SESSION['brand_id'] ?? 1;
    
    $products = \WHMCS\Products\Product::where('brand_id', $brandId)->get();
    
    return ['products' => $products];
});
```

## Checklist
- Brands created in database
- Products assigned to brands
- Client-brand association set
- Templates configured per brand
