# WHMCS Billing Separation Workflow

## Overview
Configure separate billing per tenant/client in multi-tenant deployments.

## Prerequisites
- WHMCS v8.0+
- Multi-tenant configuration

## Step-by-Step Guide

### Step 1: Tenant Billing Configuration
```php
<?php
function configure_tenant_billing(int $tenantId, array $config): void
{
    \WHMCS\Database\Capsule::table('tenant_billing_config')->insert([
        'tenant_id' => $tenantId,
        'currency' => $config['currency'],
        'tax_rate' => $config['tax_rate'],
        'payment_terms' => $config['payment_terms'],
        'invoice_prefix' => $config['invoice_prefix'],
    ]);
}
```

### Step 2: Per-Tenant Pricing
```php
<?php
add_hook('ProductPricing', 1, function($vars) {
    $tenantId = $_SESSION['tenant_id'];
    $basePrice = $vars['base_price'];
    
    $tenantConfig = get_tenant_billing_config($tenantId);
    
    // Apply tenant-specific pricing rules
    $price = $basePrice * $tenantConfig['price_multiplier'];
    
    return ['adjusted_price' => $price];
});
```

## Checklist
- Billing config per tenant
- Currency configured
- Tax rules set
- Invoice prefixes unique
