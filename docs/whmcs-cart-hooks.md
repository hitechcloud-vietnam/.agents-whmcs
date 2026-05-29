# WHMCS Cart Hooks

## Overview

Cart hooks allow customization of the shopping cart and checkout process, including product additions, removals, and order completion.

## Available Cart Hooks

### Cart Item Add Hook

```php
<?php
// Triggered when item is added to cart
add_hook('CartItemAdd', 1, function(array $vars) {
    $productId = $vars['pid'];
    $userId = $_SESSION['uid'] ?? 0;
    $configOptions = $vars['configoptions'] ?? [];
    
    // Check for bundle compatibility
    if (isPartOfBundle($productId)) {
        return [
            'abort' => true,
            'error_msg' => 'This product is part of a bundle. Please purchase the bundle.',
        ];
    }
    
    // Check for conflicting products
    $conflicts = getConflictingProducts($productId);
    foreach ($conflicts as $conflictId) {
        if (inCart($conflictId)) {
            return [
                'abort' => true,
                'error_msg' => 'This product conflicts with an item in your cart.',
            ];
        }
    }
    
    // Apply volume discounts
    $discount = calculateVolumeDiscount($userId, $productId);
    if ($discount > 0) {
        applyCartDiscount($productId, $discount);
    }
    
    // Add related products
    suggestRelatedProducts($productId);
    
    return ['abort' => false, 'product_id' => $productId];
});
```

### Cart Item Remove Hook

```php
<?php
// Triggered when item is removed from cart
add_hook('CartItemRemove', 1, function(array $vars) {
    $productId = $vars['pid'];
    
    // Remove associated addons
    removeAssociatedAddons($productId);
    
    // Remove bundle if incomplete
    if (isBundleProduct($productId)) {
        removeBundleProducts($productId);
    }
    
    return ['success' => true];
});
```

### Cart Calculate Total Hook

```php
<?php
// Modify cart total calculation
add_hook('CartCalculateTotal', 1, function(array $vars) {
    $subtotal = $vars['subtotal'];
    $userId = $_SESSION['uid'] ?? 0;
    
    $adjustments = [];
    
    // Apply loyalty discount
    $loyaltyDiscount = getLoyaltyDiscount($userId);
    if ($loyaltyDiscount > 0) {
        $adjustments[] = [
            'type' => 'discount',
            'description' => 'Loyalty Discount',
            'amount' => -$loyaltyDiscount,
        ];
    }
    
    // Apply promo code
    if (isset($_SESSION['promo_code'])) {
        $promoResult = applyPromoCode($_SESSION['promo_code'], $subtotal);
        if ($promoResult['valid']) {
            $adjustments[] = $promoResult;
        }
    }
    
    // Add handling fee
    if ($vars['payment_method'] === 'banktransfer') {
        $adjustments[] = [
            'type' => 'fee',
            'description' => 'Bank Transfer Processing Fee',
            'amount' => 2.50,
        ];
    }
    
    return [
        'adjustments' => $adjustments,
        'continue' => true,
    ];
});
```

### Pre-Order Checkout Hook

```php
<?php
// Runs before order is finalized
add_hook('PreOrderCheckout', 1, function(array $vars) {
    $userId = $_SESSION['uid'];
    $errors = [];
    $warnings = [];
    
    // Validate shipping address
    if (!$vars['has_valid_address']) {
        $errors[] = 'Please provide a valid shipping address';
    }
    
    // Check credit limit
    if (exceedsCreditLimit($userId, $vars['total'])) {
        $errors[] = 'Order exceeds your credit limit';
    }
    
    // Verify product availability
    foreach ($vars['products'] as $product) {
        if (!isProductAvailable($product['pid'])) {
            $errors[] = "Product {$product['name']} is not available";
        }
    }
    
    // Check age verification
    foreach ($vars['products'] as $product) {
        if (requiresAgeVerification($product['pid'])) {
            if (!$vars['age_verified']) {
                $errors[] = 'Age verification required for some products';
            }
        }
    }
    
    if (!empty($errors)) {
        return [
            'abort' => true,
            'error_msg' => implode('. ', $errors),
        ];
    }
    
    if (!empty($warnings)) {
        return [
            'abort' => false,
            'warning_msg' => implode('. ', $warnings),
        ];
    }
    
    return ['abort' => false];
});
```

### Order Complete Hook

```php
<?php
// Triggered when order is completed
add_hook('OrderComplete', 1, function(array $vars) {
    $orderId = $vars['order_id'];
    $userId = $vars['user_id'];
    
    // Clear cart
    clearCart($userId);
    
    // Award points
    awardLoyaltyPoints($userId, $vars['total']);
    
    // Process referral
    processReferral($userId);
    
    // Setup subscriptions
    setupRecurringBilling($orderId);
    
    // Send confirmation
    sendOrderConfirmation($orderId);
    
    return ['success' => true];
});
```

### Promo Code Hook

```php
<?php
// Validate promo code
add_hook('PromoCodeValidation', 1, function(array $vars) {
    $code = $vars['code'];
    $userId = $_SESSION['uid'] ?? 0;
    $cartTotal = $vars['cart_total'];
    
    // Check if already used
    if (hasUsedPromo($userId, $code)) {
        return [
            'valid' => false,
            'error_msg' => 'You have already used this promo code',
        ];
    }
    
    // Check minimum order value
    $promo = getPromoDetails($code);
    if ($promo['min_order'] > $cartTotal) {
        return [
            'valid' => false,
            'error_msg' => "Minimum order of {$promo['min_order']} required",
        ];
    }
    
    // Check expiration
    if (strtotime($promo['expires_at']) < time()) {
        return [
            'valid' => false,
            'error_msg' => 'This promo code has expired',
        ];
    }
    
    return [
        'valid' => true,
        'discount_type' => $promo['type'],
        'discount_value' => $promo['value'],
    ];
});
```

## Comprehensive Cart Handler

```php
<?php
class CartHookHandler {
    
    public function register(): void
    {
        add_hook('CartItemAdd', 1, [$this, 'handleItemAdd']);
        add_hook('CartItemRemove', 1, [$this, 'handleItemRemove']);
        add_hook('CartCalculateTotal', 1, [$this, 'handleCalculate']);
        add_hook('PreOrderCheckout', 1, [$this, 'handlePreCheckout']);
        add_hook('OrderComplete', 1, [$this, 'handleComplete']);
        add_hook('PromoCodeValidation', 1, [$this, 'handlePromo']);
    }
    
    public function handleItemAdd(array $vars): array
    {
        $this->validateProduct($vars);
        $this->applyDiscounts($vars);
        return ['abort' => false];
    }
    
    public function handleItemRemove(array $vars): array
    {
        $this->removeAddons($vars['pid']);
        return ['success' => true];
    }
    
    public function handleCalculate(array $vars): array
    {
        return [
            'adjustments' => $this->getAdjustments($vars),
            'continue' => true,
        ];
    }
    
    public function handlePreCheckout(array $vars): array
    {
        $errors = $this->validateCheckout($vars);
        
        if (!empty($errors)) {
            return ['abort' => true, 'error_msg' => implode('. ', $errors)];
        }
        
        return ['abort' => false];
    }
    
    public function handleComplete(array $vars): array
    {
        $this->clearCart($vars['user_id']);
        $this->awardPoints($vars['user_id']);
        $this->processReferral($vars['user_id']);
        return ['success' => true];
    }
    
    public function handlePromo(array $vars): array
    {
        return $this->validatePromo($vars);
    }
    
    private function validateProduct(array $vars): void
    {
        // Validation logic
    }
    
    private function applyDiscounts(array $vars): void
    {
        // Apply discounts
    }
    
    private function removeAddons(int $productId): void
    {
        // Remove addons
    }
    
    private function getAdjustments(array $vars): array
    {
        return [];
    }
    
    private function validateCheckout(array $vars): array
    {
        return [];
    }
    
    private function clearCart(int $userId): void
    {
        // Clear cart
    }
    
    private function awardPoints(int $userId): void
    {
        // Award points
    }
    
    private function processReferral(int $userId): void
    {
        // Process referral
    }
    
    private function validatePromo(array $vars): array
    {
        return ['valid' => false, 'error_msg' => 'Invalid promo code'];
    }
}

$handler = new CartHookHandler();
$handler->register();
```

## Related Documentation

- [WHMCS Order Hooks](/docs/whmcs-order-hooks.md)
- [WHMCS Client Hooks](/docs/whmcs-client-hooks.md)