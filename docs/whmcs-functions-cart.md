# WHMCS Cart Functions

Complete reference for shopping cart and order fulfillment functions in WHMCS.

## Overview

WHMCS provides comprehensive cart management for the ordering process including product selection, configuration, and checkout.

## Cart Session Functions

### getCart()

Gets the current cart from session.

```php
/**
 * Get current cart from session
 * 
 * @return array Cart data
 */
function getCart(): array
{
    return $_SESSION['cart'] ?? [
        'products' => [],
        'domains' => [],
        'addons' => [],
        'configoptions' => []
    ];
}
```

### saveCart()

Saves cart to session.

```php
/**
 * Save cart to session
 * 
 * @param array $cart Cart data
 * @return void
 */
function saveCart(array $cart): void
{
    $_SESSION['cart'] = $cart;
}
```

### clearCart()

Clears the cart.

```php
/**
 * Clear cart contents
 * 
 * @return void
 */
function clearCart(): void
{
    $_SESSION['cart'] = [
        'products' => [],
        'domains' => [],
        'addons' => [],
        'configoptions' => []
    ];
}
```

## Cart Products

### addProductToCart()

Adds a product to cart.

```php
/**
 * Add product to cart
 * 
 * @param array $data Product data
 * @return int Cart index
 */
function addProductToCart(array $data): int
{
    $cart = getCart();
    
    $index = count($cart['products']);
    
    $cart['products'][$index] = [
        'product_id' => $data['productid'],
        'qty' => $data['qty'] ?? 1,
        'billingcycle' => $data['billingcycle'] ?? 'Monthly',
        'configoptions' => $data['configoptions'] ?? [],
        'customfields' => $data['customfields'] ?? [],
        'domain' => $data['domain'] ?? '',
        'serverid' => $data['serverid'] ?? 0,
    ];
    
    saveCart($cart);
    
    return $index;
}
```

**Example:**
```php
// Add hosting product
addProductToCart([
    'productid' => 1,
    'billingcycle' => 'Annually',
    'domain' => 'example.com',
    'configoptions' => [
        5 => 10, // Storage option
        6 => 'SSD' // Disk type
    ]
]);

// Add addon
addProductToCart([
    'productid' => 1,
    'qty' => 2,
    'billingcycle' => 'Monthly'
]);
```

### removeProductFromCart()

Removes a product from cart.

```php
/**
 * Remove product from cart
 * 
 * @param int $index Cart index
 * @return bool Success status
 */
function removeProductFromCart(int $index): bool
{
    $cart = getCart();
    
    if (isset($cart['products'][$index])) {
        unset($cart['products'][$index]);
        $cart['products'] = array_values($cart['products']);
        saveCart($cart);
        return true;
    }
    
    return false;
}
```

### updateCartProduct()

Updates a product in cart.

```php
/**
 * Update cart product
 * 
 * @param int $index Cart index
 * @param array $data Updated data
 * @return bool Success status
 */
function updateCartProduct(int $index, array $data): bool
{
    $cart = getCart();
    
    if (!isset($cart['products'][$index])) {
        return false;
    }
    
    $cart['products'][$index] = array_merge($cart['products'][$index], $data);
    saveCart($cart);
    
    return true;
}
```

**Example:**
```php
updateCartProduct(0, [
    'billingcycle' => 'Monthly',
    'qty' => 3,
    'configoptions' => [5 => 20]
]);
```

### getCartProducts()

Gets all products in cart.

```php
/**
 * Get cart products
 * 
 * @return array Products
 */
function getCartProducts(): array
{
    $cart = getCart();
    
    $products = [];
    foreach ($cart['products'] as $index => $item) {
        $product = getProduct($item['product_id']);
        
        if (!$product) continue;
        
        $pricing = getProductPricing($item['product_id']);
        
        $products[$index] = [
            'index' => $index,
            'product_id' => $item['product_id'],
            'name' => $product['name'],
            'description' => $product['description'],
            'qty' => $item['qty'],
            'billingcycle' => $item['billingcycle'],
            'configoptions' => getCartProductConfigOptions($index),
            'price' => calculateCartProductPrice($index),
            'recurring' => isRecurring($pricing, $item['billingcycle'])
        ];
    }
    
    return $products;
}
```

## Cart Domains

### addDomainToCart()

Adds domain registration/transfer to cart.

```php
/**
 * Add domain to cart
 * 
 * @param array $data Domain data
 * @return int Cart index
 */
function addDomainToCart(array $data): int
{
    $cart = getCart();
    
    $index = count($cart['domains']);
    
    $cart['domains'][$index] = [
        'domain' => $data['domain'],
        'type' => $data['type'] ?? 'register', // register, transfer, own
        'period' => $data['period'] ?? 1,
        'dnsmanagement' => $data['dnsmanagement'] ?? 0,
        'emailforwarding' => $data['emailforwarding'] ?? 0,
        'idprotection' => $data['idprotection'] ?? 0,
        'registrar' => $data['registrar'] ?? '',
    ];
    
    saveCart($cart);
    
    return $index;
}
```

**Example:**
```php
// Register new domain
addDomainToCart([
    'domain' => 'example.com',
    'type' => 'register',
    'period' => 2,
    'dnsmanagement' => 1,
    'idprotection' => 1
]);

// Transfer domain
addDomainToCart([
    'domain' => 'example.org',
    'type' => 'transfer',
    'period' => 1,
    'registrar' => 'enom'
]);
```

### removeDomainFromCart()

Removes a domain from cart.

```php
/**
 * Remove domain from cart
 * 
 * @param int $index Cart index
 * @return bool Success status
 */
function removeDomainFromCart(int $index): bool
{
    $cart = getCart();
    
    if (isset($cart['domains'][$index])) {
        unset($cart['domains'][$index]);
        $cart['domains'] = array_values($cart['domains']);
        saveCart($cart);
        return true;
    }
    
    return false;
}
```

## Cart Addons

### addAddonToCart()

Adds addon to cart.

```php
/**
 * Add addon to cart
 * 
 * @param int $productIndex Associated product index
 * @param int $addonId Addon ID
 * @return int Cart index
 */
function addAddonToCart(int $productIndex, int $addonId): int
{
    $cart = getCart();
    
    $index = count($cart['addons']);
    
    $cart['addons'][$index] = [
        'product_index' => $productIndex,
        'addon_id' => $addonId,
        'billingcycle' => 'Monthly',
    ];
    
    saveCart($cart);
    
    return $index;
}
```

**Example:**
```php
// Add SSL addon to first product
addAddonToCart(0, 5);
```

### removeAddonFromCart()

Removes addon from cart.

```php
/**
 * Remove addon from cart
 * 
 * @param int $index Cart index
 * @return bool Success status
 */
function removeAddonFromCart(int $index): bool
{
    $cart = getCart();
    
    if (isset($cart['addons'][$index])) {
        unset($cart['addons'][$index]);
        $cart['addons'] = array_values($cart['addons']);
        saveCart($cart);
        return true;
    }
    
    return false;
}
```

## Cart Configuration Options

### setCartConfigOption()

Sets configuration option for cart item.

```php
/**
 * Set configuration option
 * 
 * @param string $type Item type (product, addon)
 * @param int $index Cart index
 * @param int $optionId Config option ID
 * @param mixed $value Option value
 * @return bool Success status
 */
function setCartConfigOption(string $type, int $index, int $optionId, $value): bool
{
    $cart = getCart();
    
    $key = $type . 's'; // products or addons
    $subkey = $type === 'product' ? 'configoptions' : 'options';
    
    $cart[$key][$index][$subkey][$optionId] = $value;
    
    saveCart($cart);
    
    return true;
}
```

**Example:**
```php
setCartConfigOption('product', 0, 5, 20); // Set 20GB storage
setCartConfigOption('product', 0, 6, 'SSD'); // Set SSD disk type
```

### getCartProductConfigOptions()

Gets config options for a cart product.

```php
/**
 * Get cart product config options
 * 
 * @param int $index Cart product index
 * @return array Config options with pricing
 */
function getCartProductConfigOptions(int $index): array
{
    $cart = getCart();
    
    if (!isset($cart['products'][$index])) {
        return [];
    }
    
    $productId = $cart['products'][$index]['product_id'];
    $selectedOptions = $cart['products'][$index]['configoptions'] ?? [];
    
    $configOptions = getProductConfigOptions($productId);
    $result = [];
    
    foreach ($configOptions as $option) {
        $optionValues = getConfigOptionValues($option['id']);
        $selectedValue = $selectedOptions[$option['id']] ?? null;
        
        $result[] = [
            'id' => $option['id'],
            'name' => $option['optionname'],
            'type' => $option['optiontype'],
            'values' => $optionValues,
            'selected' => $selectedValue
        ];
    }
    
    return $result;
}
```

## Cart Pricing

### calculateCartTotal()

Calculates cart total.

```php
/**
 * Calculate cart total
 * 
 * @param string $currency Currency code
 * @return array Totals
 */
function calculateCartTotal(string $currency = 'USD'): array
{
    $cart = getCart();
    
    $subtotal = 0;
    $discount = 0;
    $tax = 0;
    $total = 0;
    
    // Calculate product totals
    foreach ($cart['products'] as $index => $item) {
        $price = calculateCartProductPrice($index);
        $subtotal += $price;
    }
    
    // Calculate domain totals
    foreach ($cart['domains'] as $index => $domain) {
        $price = calculateCartDomainPrice($index);
        $subtotal += $price;
    }
    
    // Calculate addon totals
    foreach ($cart['addons'] as $index => $addon) {
        $price = calculateCartAddonPrice($index);
        $subtotal += $price;
    }
    
    // Apply promo code
    if (!empty($_SESSION['promo_code'])) {
        $discount = applyPromoCode($_SESSION['promo_code'], $subtotal);
    }
    
    $taxable = $subtotal - $discount;
    $tax = calculateCartTax($taxable);
    $total = $taxable + $tax;
    
    return [
        'subtotal' => $subtotal,
        'discount' => $discount,
        'tax' => $tax,
        'total' => $total,
        'currency' => $currency
    ];
}
```

### calculateCartProductPrice()

Calculates price for a cart product.

```php
/**
 * Calculate cart product price
 * 
 * @param int $index Cart product index
 * @return float Price
 */
function calculateCartProductPrice(int $index): float
{
    $cart = getCart();
    
    if (!isset($cart['products'][$index])) {
        return 0;
    }
    
    $item = $cart['products'][$index];
    $product = getProduct($item['product_id']);
    $currencyId = getClientCurrency($_SESSION['uid'] ?? 0) ?: 1;
    $pricing = getProductPricing($item['product_id'], $currencyId);
    
    $basePrice = $pricing[$item['billingcycle']] ?? 0;
    
    // Add config option prices
    $configPrice = 0;
    foreach ($item['configoptions'] ?? [] as $optionId => $value) {
        $configPrice += getConfigOptionPrice($optionId, $value, $item['billingcycle'], $currencyId);
    }
    
    return ($basePrice + $configPrice) * $item['qty'];
}
```

### calculateCartDomainPrice()

Calculates price for a domain.

```php
/**
 * Calculate cart domain price
 * 
 * @param int $index Cart domain index
 * @return float Price
 */
function calculateCartDomainPrice(int $index): float
{
    $cart = getCart();
    
    if (!isset($cart['domains'][$index])) {
        return 0;
    }
    
    $domain = $cart['domains'][$index];
    $currencyId = getClientCurrency($_SESSION['uid'] ?? 0) ?: 1;
    
    $pricing = getTldPricing($domain['domain'], $currencyId);
    
    $basePrice = $pricing[$domain['type']][$domain['period']] ?? 0;
    
    // Add addon prices
    $addonPrice = 0;
    if ($domain['dnsmanagement']) {
        $addonPrice += getDomainAddOnPrice('dnsmanagement', $currencyId);
    }
    if ($domain['emailforwarding']) {
        $addonPrice += getDomainAddOnPrice('emailforwarding', $currencyId);
    }
    if ($domain['idprotection']) {
        $addonPrice += getDomainAddOnPrice('idprotection', $currencyId);
    }
    
    return $basePrice + ($addonPrice * $domain['period']);
}
```

## Cart Promo Codes

### applyPromoCode()

Applies promo code to cart.

```php
/**
 * Apply promo code
 * 
 * @param string $code Promo code
 * @return array Result
 */
function applyPromoCode(string $code): array
{
    $promo = Capsule::table('tblpromotions')
        ->where('code', $code)
        ->where('active', 1)
        ->first();
    
    if (!$promo) {
        return ['success' => false, 'error' => 'Invalid promo code'];
    }
    
    // Check expiration
    if ($promo->expirydate && strtotime($promo->expirydate) < time()) {
        return ['success' => false, 'error' => 'Promo code expired'];
    }
    
    // Check uses
    if ($promo->maxuses && $promo->uses >= $promo->maxuses) {
        return ['success' => false, 'error' => 'Promo code limit reached'];
    }
    
    $_SESSION['promo_code'] = $code;
    $_SESSION['promo_data'] = (array) $promo;
    
    return [
        'success' => true,
        'code' => $code,
        'type' => $promo->type,
        'value' => $promo->value
    ];
}
```

### removePromoCode()

Removes promo code from cart.

```php
/**
 * Remove promo code
 * 
 * @return void
 */
function removePromoCode(): void
{
    unset($_SESSION['promo_code']);
    unset($_SESSION['promo_data']);
}
```

### applyPromoDiscount()

Applies discount based on promo type.

```php
/**
 * Apply promo discount
 * 
 * @param float $subtotal Cart subtotal
 * @return float Discount amount
 */
function applyPromoDiscount(float $subtotal): float
{
    if (empty($_SESSION['promo_data'])) {
        return 0;
    }
    
    $promo = $_SESSION['promo_data'];
    
    if ($promo['type'] === 'percentage') {
        return $subtotal * ($promo['value'] / 100);
    }
    
    return min($promo['value'], $subtotal);
}
```

## Cart Validation

### validateCart()

Validates cart contents.

```php
/**
 * Validate cart
 * 
 * @return array Validation result
 */
function validateCart(): array
{
    $cart = getCart();
    $errors = [];
    
    // Validate products
    foreach ($cart['products'] as $index => $item) {
        $product = getProduct($item['product_id']);
        
        if (!$product) {
            $errors[] = "Product at index {$index} is invalid";
            continue;
        }
        
        // Check domain availability if required
        if ($product['showdomainoptions'] && !empty($item['domain'])) {
            if (!isValidDomain($item['domain'])) {
                $errors[] = "Invalid domain: {$item['domain']}";
            }
        }
        
        // Check stock
        if ($product['stockcontrol'] && $product['stocklevel'] < $item['qty']) {
            $errors[] = "Insufficient stock for {$product['name']}";
        }
    }
    
    // Validate domains
    foreach ($cart['domains'] as $index => $domain) {
        if (!isValidDomain($domain['domain'])) {
            $errors[] = "Invalid domain: {$domain['domain']}";
        }
    }
    
    // Validate addons
    foreach ($cart['addons'] as $index => $addon) {
        if (!isset($cart['products'][$addon['product_index']])) {
            $errors[] = "Addon at index {$index} references invalid product";
        }
    }
    
    return [
        'valid' => empty($errors),
        'errors' => $errors
    ];
}
```

## Cart Checkout

### processCart()

Processes cart into order.

```php
/**
 * Process cart and create order
 * 
 * @param array $data Checkout data
 * @return array Order result
 */
function processCart(array $data): array
{
    $validation = validateCart();
    
    if (!$validation['valid']) {
        return [
            'success' => false,
            'errors' => $validation['errors']
        ];
    }
    
    $clientId = $data['clientid'] ?? $_SESSION['uid'];
    
    // Create order
    $orderId = createOrder([
        'clientid' => $clientId,
        'paymentmethod' => $data['paymentmethod'],
        'ipaddress' => getClientIp()
    ], false, true, true);
    
    // Add products to order
    $cart = getCart();
    
    foreach ($cart['products'] as $item) {
        addOrderItem($orderId, [
            'clientid' => $clientId,
            'type' => 'Hosting',
            'productid' => $item['product_id'],
            'domain' => $item['domain'] ?? '',
            'qty' => $item['qty'],
            'billingcycle' => $item['billingcycle'],
            'amount' => calculateCartProductPrice(array_search($item, $cart['products']))
        ]);
    }
    
    // Add domains to order
    foreach ($cart['domains'] as $domain) {
        $type = $domain['type'] === 'transfer' ? 'DomainTransfer' : 'DomainRegister';
        
        addOrderItem($orderId, [
            'clientid' => $clientId,
            'type' => $type,
            'domain' => $domain['domain'],
            'billingcycle' => 'DomainRegister',
            'amount' => calculateCartDomainPrice(array_search($domain, $cart['domains']))
        ]);
    }
    
    // Generate invoice
    $invoiceId = generateInvoice($orderId, false);
    
    // Apply promo discount
    if (!empty($_SESSION['promo_code'])) {
        $totals = calculateCartTotal();
        addInvoiceCredit($invoiceId, $totals['discount'], 'Promo code: ' . $_SESSION['promo_code']);
    }
    
    // Clear cart
    clearCart();
    
    return [
        'success' => true,
        'order_id' => $orderId,
        'invoice_id' => $invoiceId
    ];
}
```

## Best Practices

1. **Validate domain availability** - Check before adding to cart
2. **Handle stock levels** - Show availability status
3. **Calculate in real-time** - Update totals as items change
4. **Persist cart** - Save cart for returning users
5. **Apply discounts correctly** - Handle percentage vs fixed discounts
6. **Create orders atomically** - Use transactions for order creation

## Related Functions

- [whmcs-functions-orders.md](whmcs-functions-orders.md) - Order processing
- [whmcs-functions-products.md](whmcs-functions-products.md) - Product management
- [whmcs-functions-domains.md](whmcs-functions-domains.md) - Domain management