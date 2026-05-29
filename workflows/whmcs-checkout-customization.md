# WHMCS Checkout Customization Workflow

## Overview
Comprehensive workflow for customizing the WHMCS checkout process.

## Prerequisites
- WHMCS v8.0+
- Template access

## Step-by-Step Guide

### Step 1: Custom Checkout Fields
```php
<?php
add_hook('ShoppingCartCheckoutFields', 1, function($vars) {
    return [
        [
            'name' => 'custom_field_1',
            'label' => 'How did you hear about us?',
            'type' => 'dropdown',
            'options' => ['Google', 'Referral', 'Social Media', 'Other'],
            'required' => false,
        ],
    ];
});
```

### Step 2: Custom Validation
```php
<?php
add_hook('ShoppingCartValidateCheckout', 1, function($vars) {
    $source = $_POST['custom_field_1'] ?? '';
    
    if (empty($source)) {
        return ['error' => 'Please tell us how you found us'];
    }
    
    return ['success' => true];
});
```

### Step 3: Post-Checkout Actions
```php
<?php
add_hook('AfterShoppingCartCheckout', 1, function($vars) {
    $source = $_SESSION['checkout_source'] ?? '';
    
    \WHMCS\Database\Capsule::table('mod_analytics')->insert([
        'client_id' => $vars['clientId'],
        'source' => $source,
        'order_id' => $vars['orderId'],
    ]);
});
```

## Checklist
- Custom fields added
- Validation rules set
- Post-checkout hooks configured
- Analytics integrated
