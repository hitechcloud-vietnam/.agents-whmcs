# WHMCS CalculateCartTotals Hook Reference

## Overview

The `CalculateCartTotals` hook fires when cart totals are calculated in WHMCS. This hook runs during the checkout process and allows modification of prices, taxes, discounts, and fees before finalization.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `products` | array | Cart products/services |
| `domains` | array | Cart domains |
| `addons` | array | Cart addons |
| `renewals` | array | Renewal items |
| `subtotal` | float | Subtotal amount |
| `discount` | float | Discount amount |
| `total` | float | Final total |
| `taxlevel` | array | Tax calculations |
| `userid` | int | Client ID |

## Example Implementation

```php
<?php
add_hook('CalculateCartTotals', 1, function(array $params) {
    // Apply custom discount
    $params['discount'] += 10.00;
    $params['total'] -= 10.00;
    
    return $params;
});
```

## Dynamic Pricing

```php
<?php
add_hook('CalculateCartTotals', 1, function(array $params) {
    $userId = (int)$params['userid'];
    $client = getClientsDetails($userId);
    
    // 1. Volume discounts based on total
    $subtotal = $params['subtotal'];
    if ($subtotal >= 500) {
        $discount = $subtotal * 0.15; // 15% off
    } elseif ($subtotal >= 250) {
        $discount = $subtotal * 0.10; // 10% off
    } elseif ($subtotal >= 100) {
        $discount = $subtotal * 0.05; // 5% off
    }
    
    if (isset($discount)) {
        $params['discount'] += $discount;
        $params['total'] -= $discount;
    }
    
    // 2. Client tier discount
    if ($client['groupid'] == 2) { // Premium tier
        $params['discount'] += $subtotal * 0.05;
    }
    
    return $params;
});
```

## Custom Fees

```php
<?php
add_hook('CalculateCartTotals', 1, function(array $params) {
    // 1. Add payment processing fee
    $paymentMethod = $_SESSION['cart']['paymentmethod'] ?? 'default';
    if ($paymentMethod === 'creditcard') {
        $fee = $params['subtotal'] * 0.029 + 0.30; // Stripe-like fee
        $params['fees'][] = [
            'name' => 'Payment Processing Fee',
            'amount' => round($fee, 2)
        ];
        $params['total'] += round($fee, 2);
    }
    
    // 2. Add rush setup fee if requested
    if ($_SESSION['cart']['rush_setup'] ?? false) {
        $params['fees'][] = [
            'name' => 'Rush Setup Fee',
            'amount' => 25.00
        ];
        $params['total'] += 25.00;
    }
    
    // 3. Add domain privacy protection fee
    foreach ($params['domains'] ?? [] as $domain) {
        if ($domain['protect_enable'] ?? false) {
            $params['fees'][] = [
                'name' => "WHOIS Privacy - {$domain['domain']}",
                'amount' => 9.99
            ];
            $params['total'] += 9.99;
        }
    }
    
    return $params;
});
```

## Tax Calculations

```php
<?php
add_hook('CalculateCartTotals', 1, function(array $params) {
    // 1. Override tax calculation for specific regions
    if ($params['taxlevel']['country'] === 'DE') {
        // German reverse charge for B2B
        $params['taxlevel']['taxrate'] = 0;
        $params['taxlevel']['taxname'] = 'Reverse Charge';
    }
    
    // 2. Apply tax exemption
    if (hasTaxExemption($params['userid'])) {
        $params['taxexempt'] = true;
        $params['taxlevel']['taxrate'] = 0;
    }
    
    // 3. Canadian GST/HST harmonization
    $provinceTaxRates = [
        'AB' => 5, 'BC' => 12, 'MB' => 12, 'NB' => 15,
        'NL' => 15, 'NS' => 15, 'NT' => 5, 'NU' => 5,
        'ON' => 13, 'PE' => 15, 'QC' => 14.975, 'SK' => 11,
        'YT' => 5
    ];
    
    if ($params['taxlevel']['country'] === 'CA') {
        $province = $params['taxlevel']['state'] ?? '';
        if (isset($provinceTaxRates[$province])) {
            $params['taxlevel']['taxrate'] = $provinceTaxRates[$province];
        }
    }
    
    return $params;
});
```

## Promotion Codes

```php
<?php
add_hook('CalculateCartTotals', 1, function(array $params) {
    $promoCode = $_SESSION['cart']['promocode'] ?? null;
    
    if (!$promoCode) {
        return $params;
    }
    
    // Validate and apply promo
    $promo = validatePromoCode($promoCode, $params['userid']);
    
    if ($promo) {
        switch ($promo['type']) {
            case 'percentage':
                $discount = $params['subtotal'] * ($promo['value'] / 100);
                break;
            case 'fixed':
                $discount = min($promo['value'], $params['subtotal']);
                break;
            case 'free_setup':
                // Apply to first product only
                $discount = getFirstProductSetupFee($params['products']);
                break;
        }
        
        if (isset($discount)) {
            $params['promo_discount'] = $discount;
            $params['discount'] += $discount;
            $params['total'] -= $discount;
        }
    }
    
    return $params;
});
```

## Use Cases

- **Dynamic Pricing**: Volume, tier, or loyalty discounts
- **Custom Fees**: Processing, setup, or handling fees
- **Tax Override**: Special tax rules by region
- **Promotions**: Promo code validation and application
- **Conditional Totals**: Based on payment method or options

## Notes

- Runs during cart calculation, may run multiple times
- Can modify any total component
- Tax calculations run after this hook
- Use with `AfterCartCheckout` for post-processing

## Related Hooks

- `AfterCartCheckout` - After checkout
- `InvoiceCreated` - Invoice creation
- `CalculateTax` - Tax calculation

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Cart Configuration](../whmcs-cart-setup.md)