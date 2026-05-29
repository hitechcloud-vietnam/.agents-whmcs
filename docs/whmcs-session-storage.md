# WHMCS Session Storage

## Overview

Session management handles user state across requests in WHMCS.

## Basic Session Operations

```php
<?php
use WHMCS\Session\Session;

// Start session (already done by WHMCS)
$session = new Session();

// Set session value
$_SESSION['key'] = 'value';
$session->set('key', 'value');

// Get session value
$value = $_SESSION['key'] ?? null;
$value = $session->get('key');

// Get with default
$value = $session->get('key', 'default');

// Check if exists
if (isset($_SESSION['key'])) {
    // Exists
}

if ($session->has('key')) {
    // Exists
}

// Delete session value
unset($_SESSION['key']);
$session->forget('key');

// Destroy session
session_destroy();
$session->destroy();
```

## Session Flash Data

```php
<?php
// Set flash data (available only for next request)
$_SESSION['_flash'] = ['message' => 'Success!'];
$_SESSION['_flash_old'] = $_SESSION['_flash'] ?? [];

// Get flash data
$message = $_SESSION['_flash']['message'] ?? null;
$session->flush(); // Clear old flash data
```

## Session in Hooks

```php
<?php
add_hook('ClientLogin', 1, function($vars) {
    // Store login timestamp
    $_SESSION['last_login'] = time();
    
    // Store login IP
    $_SESSION['login_ip'] = $_SERVER['REMOTE_ADDR'];
    
    // Store user agent
    $_SESSION['user_agent'] = $_SERVER['HTTP_USER_AGENT'];
    
    // Initialize cart
    $_SESSION['cart'] = [];
    
    return ['success' => true];
});

add_hook('ClientLogout', 1, function($vars) {
    // Log session duration
    if (isset($_SESSION['last_login'])) {
        $duration = time() - $_SESSION['last_login'];
        logSessionDuration($vars['user_id'], $duration);
    }
    
    // Clear session
    $_SESSION = [];
    
    return ['success' => true];
});
```

## Session Cart Data

```php
<?php
class CartSessionManager
{
    public static function addProduct(int $productId, array $config = []): void
    {
        $_SESSION['cart']['products'][] = [
            'pid' => $productId,
            'config' => $config,
            'added_at' => time(),
        ];
    }
    
    public static function removeProduct(int $index): void
    {
        unset($_SESSION['cart']['products'][$index]);
        $_SESSION['cart']['products'] = array_values($_SESSION['cart']['products']);
    }
    
    public static function getProducts(): array
    {
        return $_SESSION['cart']['products'] ?? [];
    }
    
    public static function clearCart(): void
    {
        $_SESSION['cart'] = [
            'products' => [],
            'addons' => [],
            'domains' => [],
        ];
    }
    
    public static function calculateTotal(): float
    {
        $total = 0;
        
        foreach (self::getProducts() as $product) {
            $price = getProductPrice($product['pid']);
            $total += $price;
        }
        
        return $total;
    }
}
```

## Session Security

```php
<?php
function secureSessionStart(): void
{
    // Use secure cookies
    ini_set('session.cookie_httponly', 1);
    ini_set('session.cookie_secure', 1);
    ini_set('session.use_strict_mode', 1);
    
    // Regenerate session ID periodically
    if (!isset($_SESSION['last_regeneration'])) {
        $_SESSION['last_regeneration'] = time();
    }
    
    if (time() - $_SESSION['last_regeneration'] > 300) {
        session_regenerate_id(true);
        $_SESSION['last_regeneration'] = time();
    }
}

function validateSession(): bool
{
    // Check IP address
    if (!isset($_SESSION['ip_address'])) {
        $_SESSION['ip_address'] = $_SERVER['REMOTE_ADDR'];
    }
    
    if ($_SESSION['ip_address'] !== $_SERVER['REMOTE_ADDR']) {
        session_destroy();
        return false;
    }
    
    // Check user agent
    if (!isset($_SESSION['user_agent'])) {
        $_SESSION['user_agent'] = $_SERVER['HTTP_USER_AGENT'] ?? '';
    }
    
    if (strlen($_SESSION['user_agent']) > 0 && 
        $_SESSION['user_agent'] !== ($_SERVER['HTTP_USER_AGENT'] ?? '')) {
        session_destroy();
        return false;
    }
    
    // Check timeout
    if (isset($_SESSION['last_activity']) && 
        (time() - $_SESSION['last_activity']) > 1800) {
        session_destroy();
        return false;
    }
    
    $_SESSION['last_activity'] = time();
    
    return true;
}
```

## Session in AJAX Requests

```php
<?php
// AJAX endpoint with session
if ($_SERVER['REQUEST_METHOD'] === 'AJAX') {
    // Session is already started in WHMCS
    
    // Validate session
    if (!isset($_SESSION['uid'])) {
        http_response_code(401);
        echo json_encode(['error' => 'Unauthorized']);
        exit;
    }
    
    // Process request
    $result = processAjaxRequest($_POST);
    
    echo json_encode($result);
    exit;
}
```

## Best Practices

1. **Never store sensitive data** - Don't store passwords in session
2. **Regenerate IDs** - On login/logout
3. **Set timeouts** - Prevent session hijacking
4. **Use HTTPS** - Always use secure cookies
5. **Clear sensitive data** - On logout

## Related Documentation

- [WHMCS Cache Layer](/docs/whmcs-cache-layer.md)
- [WHMCS Module Security](/docs/whmcs-module-security.md)