# WHMCS Reseller Setup Workflow

## Overview
Configure WHMCS for reseller hosting with client management and resource limits.

## Prerequisites
- WHMCS v8.0+
- Reseller account

## Step-by-Step Guide

### Step 1: Reseller Configuration
```php
<?php
function create_reseller(array $config): int
{
    return \WHMCS\Database\Capsule::table('mod_resellers')->insertGetId([
        'client_id' => $config['client_id'],
        'max_clients' => $config['max_clients'],
        'max_products' => $config['max_products'],
        'commission_rate' => $config['commission_rate'],
        'status' => 'active',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}
```

### Step 2: Resource Limits Hook
```php
<?php
add_hook('ClientAreaPage', 1, function($vars) {
    $clientId = \Auth::id();
    $reseller = get_reseller($clientId);
    
    if ($reseller) {
        return [
            'is_reseller' => true,
            'client_count' => get_reseller_client_count($clientId),
            'max_clients' => $reseller->max_clients,
            'available_slots' => $reseller->max_clients - get_reseller_client_count($clientId),
        ];
    }
});
```

### Step 3: Commission Calculation
```php
<?php
add_hook('OrderPaid', 1, function($vars) {
    $clientId = $vars['userId'];
    $parentReseller = get_parent_reseller($clientId);
    
    if ($parentReseller) {
        $order = \WHMCS\Billing\Order\Order::find($vars['orderId']);
        $commission = $order->total * ($parentReseller->commission_rate / 100);
        
        add_reseller_commission($parentReseller->client_id, $commission);
    }
});
```

## Checklist
- Reseller accounts created
- Resource limits configured
- Commission rates set
- Client management enabled
