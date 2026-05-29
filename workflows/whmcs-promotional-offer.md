# WHMCS Promotional Offer Workflow

## Overview
Comprehensive workflow for managing promotional offers with time-limited deals, flash sales, and special pricing.

## Prerequisites
- WHMCS v8.0+
- Promotional pricing configured

## Step-by-Step Guide

### Step 1: Configure Promotions
```php
<?php
class PromotionManager
{
    public function createPromotion(array $data): int
    {
        return \WHMCS\Database\Capsule::table('mod_promotions')->insertGetId([
            'name' => $data['name'],
            'code' => $data['code'],
            'type' => $data['type'],
            'value' => $data['value'],
            'start_date' => $data['start_date'],
            'end_date' => $data['end_date'],
            'max_uses' => $data['max_uses'] ?? null,
            'product_ids' => json_encode($data['product_ids'] ?? []),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function isActive(int $promotionId): bool
    {
        $promo = \WHMCS\Database\Capsule::table('mod_promotions')
            ->where('id', $promotionId)
            ->first();

        if (!$promo) return false;

        $now = date('Y-m-d H:i:s');
        return $promo->start_date <= $now && $promo->end_date >= $now;
    }
}
```

### Step 2: Apply to Cart
```php
add_hook('ShoppingCartValidateCheckout', 1, function($vars) {
    $code = $_POST['promo_code'] ?? '';
    
    $promo = \WHMCS\Database\Capsule::table('mod_promotions')
        ->where('code', $code)
        ->first();

    if ($promo && (new PromotionManager())->isActive($promo->id)) {
        $discount = calculate_promo_discount($promo);
        return ['promo_discount' => $discount];
    }
});
```

## Checklist
- Promotions configured
- Time limits enforced
- Discounts applied correctly
- Usage tracked
