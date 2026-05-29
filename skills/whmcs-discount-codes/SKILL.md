# WHMCS Discount Codes

## Concept Explanation

Discount codes (promo codes) allow temporary or permanent price reductions on products and services. WHMCS supports percentage and fixed amount discounts, minimum order requirements, single/multiple use limits, and automatic or manual application.

### Code Types

- **Percentage**: X% off total
- **Fixed Amount**: $X off total
- **Free Setup**: Waive setup fees
- **Free Period**: Free billing cycle
- **Product-Specific**: Discount on specific items

## Code Patterns

```php
<?php
// hooks/discount_enhancements.php

add_hook('CartValidateCoupon', 1, function($vars) {
    $code = strtoupper($vars['code']);
    
    // Custom validation rules
    $client = getClientsDetails($_SESSION['uid']);
    
    // VIP customer exclusive
    if (substr($code, 0, 4) === 'VIP-') {
        if ($client['groupid'] != VIP_GROUP_ID) {
            return ['valid' => false, 'error' => 'This code is for VIP members only'];
        }
    }
    
    // First order only
    if (substr($code, 0, 5) === 'NEW10') {
        $orderCount = getClientOrderCount($client['id']);
        if ($orderCount > 0) {
            return ['valid' => false, 'error' => 'This code is for new customers only'];
        }
    }
    
    return $vars;
});

add_hook('CartApplyCouponDiscount', 1, function($vars) {
    $code = $vars['code'];
    
    // Tiered discount based on order value
    if ($code === 'TIERED20') {
        $subtotal = $vars['subtotal'];
        if ($subtotal >= 500) return ['discount' => $subtotal * 0.20];
        if ($subtotal >= 200) return ['discount' => $subtotal * 0.15];
        if ($subtotal >= 100) return ['discount' => $subtotal * 0.10];
    }
    
    return $vars;
});
```

## Implementation Checklist

- [ ] Create promotional code strategy
- [ ] Set up code tracking
- [ ] Configure usage limits
- [ ] Test discount application
