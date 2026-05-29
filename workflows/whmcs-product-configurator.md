# WHMCS Product Configurator Setup Workflow

## Overview
Comprehensive workflow for setting up product configurators with options, addons, and pricing.

## Prerequisites
- WHMCS v8.0+
- Product groups configured

## Step-by-Step Guide

### Step 1: Configure Product Groups
```php
<?php
function create_product_group(string $name, string $description): int
{
    return \WHMCS\Database\Capsule::table('tblproductgroups')->insertGetId([
        'name' => $name,
        'headline' => '',
        'tagline' => $description,
        'orderfrm' => 1,
        'disabled' => 0,
    ]);
}
```

### Step 2: Create Configurable Options
```php
<?php
function create_config_option(array $config): int
{
    $optionId = \WHMCS\Database\Capsule::table('tblproductconfigoptions')->insertGetId([
        'gid' => $config['group_id'],
        'optionname' => $config['name'],
        'optiontype' => $config['type'],
        'qtyminimum' => $config['qty_min'] ?? 0,
        'qtymaximum' => $config['qty_max'] ?? 0,
        'order' => $config['order'] ?? 0,
    ]);

    foreach ($config['options'] as $option) {
        \WHMCS\Database\Capsule::table('tblproductconfigoptionssub')->insert([
            'configid' => $optionId,
            'optionname' => $option['name'],
            'sortorder' => $option['order'] ?? 0,
            'hidden' => 0,
        ]);
    }

    return $optionId;
}
```

### Step 3: Pricing Tiers
```php
<?php
function set_config_pricing(int $optionId, array $pricing): void
{
    $currencies = \WHMCS\Database\Capsule::table('tblcurrencies')->get();

    foreach ($currencies as $currency) {
        foreach ($pricing as $cycle => $price) {
            \WHMCS\Database\Capsule::table('tblpricing')->insert([
                'type' => 'configoptions',
                'relid' => $optionId,
                'currency' => $currency->id,
                $cycle => $price,
            ]);
        }
    }
}
```

## Checklist
- Product groups created
- Configurable options added
- Pricing tiers set
- Dependencies configured
