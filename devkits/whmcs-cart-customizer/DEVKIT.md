# WHMCS Cart Customizer - Complete Module Code

## Module Definition (version 780)

```php
<?php
/**
 * WHMCS Cart Customizer Module
 * 
 * @package WHMCS
 * @subpackage Addon Modules
 * 
 * Module Version: 1.0.0
 * WHMCS Version: 7.8.0+
 */

// Prevent direct access
if (!defined("WHMCS")) {
    die("Direct access denied");
}

/**
 * Define module information
 */
function whmcs_cart_customizer_config()
{
    return [
        'name'          => 'Cart Customizer',
        'description'   => 'Enhances the WHMCS shopping cart with custom styling and features',
        'author'        => 'DevKit Generator',
        'version'       => '1.0.0',
        'fields'        => [
            'layout' => [
                'Type'          => 'dropdown',
                'Default'       => 'grid',
                'Options'       => [
                    'grid'          => 'Grid View',
                    'list'          => 'List View',
                    'compact'       => 'Compact View',
                    'detailed'      => 'Detailed View'
                ],
                'Description'   => 'Default product display layout in cart'
            ],
            'show_mini_cart' => [
                'Type'          => 'yesno',
                'Default'       => 'on',
                'Description'   => 'Enable mini-cart floating widget'
            ],
            'animations' => [
                'Type'          => 'yesno',
                'Default'       => 'on',
                'Description'   => 'Enable add-to-cart animations'
            ],
            'cart_theme' => [
                'Type'          => 'dropdown',
                'Default'       => 'default',
                'Options'       => [
                    'default'       => 'Default WHMCS',
                    'minimal'       => 'Minimal',
                    'modern'        => 'Modern',
                    'elegant'       => 'Elegant',
                    'dark'          => 'Dark Mode'
                ],
                'Description'   => 'Cart color scheme and theme'
            ],
            'show_product_images' => [
                'Type'          => 'yesno',
                'Default'       => 'on',
                'Description'   => 'Show product images in cart'
            ],
            'show_product_desc' => [
                'Type'          => 'yesno',
                'Default'       => 'off',
                'Description'   => 'Show product descriptions in cart items'
            ],
            'enable_quick_add' => [
                'Type'          => 'yesno',
                'Default'       => 'on',
                'Description'   => 'Enable quantity quick-adjust buttons'
            ],
            'promo_banner' => [
                'Type'          => 'text',
                'Size'          => '100',
                'Description'   => 'Promotional banner text (HTML allowed)'
            ],
            'custom_css' => [
                'Type'          => 'textarea',
                'Rows'          => '10',
                'Description'   => 'Custom CSS for cart customization'
            ]
        ]
    ];
}

/**
 * Activate module
 */
function whmcs_cart_customizer_activate()
{
    // Create module tables
    $sql = "CREATE TABLE IF NOT EXISTS `mod_cart_customizer_settings` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `setting_name` VARCHAR(100) NOT NULL,
        `setting_value` TEXT,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        UNIQUE KEY `setting_name` (`setting_name`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";

    full_query($sql);

    $sql2 = "CREATE TABLE IF NOT EXISTS `mod_cart_customizer_analytics` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `user_id` INT(11) DEFAULT NULL,
        `session_id` VARCHAR(128) NOT NULL,
        `action` VARCHAR(50) NOT NULL,
        `product_id` INT(11) DEFAULT NULL,
        `quantity` INT(11) DEFAULT 1,
        `timestamp` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `session_id` (`session_id`),
        KEY `action` (`action`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";

    full_query($sql2);

    // Insert default settings
    $defaultSettings = [
        'layout'            => 'grid',
        'show_mini_cart'    => '1',
        'animations'        => '1',
        'cart_theme'        => 'default',
        'custom_css'        => '',
        'promo_banner'      => ''
    ];

    foreach ($defaultSettings as $name => $value) {
        insert_query('mod_cart_customizer_settings', [
            'setting_name'   => $name,
            'setting_value'  => $value
        ]);
    }

    return [
        'status'    => 'success',
        'description' => 'Cart Customizer module activated successfully'
    ];
}

/**
 * Deactivate module
 */
function whmcs_cart_customizer_deactivate()
{
    // Optional: Keep data on deactivation
    return [
        'status'    => 'success',
        'description' => 'Cart Customizer module deactivated'
    ];
}

/**
 * Upgrade module
 */
function whmcs_cart_customizer_upgrade($vars)
{
    $version = $vars['version'];

    if ($version < '1.0.1') {
        // Add any upgrade SQL here
        logActivity('Cart Customizer upgraded to version 1.0.1');
    }
}

/**
 * Output module admin area
 */
function whmcs_cart_customizer_output($vars)
{
    $modulelink = $vars['modulelink'];
    $version = $vars['version'];
    $config = $vars;

    // Handle form submission
    if ($_POST['save_settings']) {
        $settings = [
            'layout'            => $_POST['layout'],
            'show_mini_cart'   => isset($_POST['show_mini_cart']) ? '1' : '0',
            'animations'       => isset($_POST['animations']) ? '1' : '0',
            'cart_theme'       => $_POST['cart_theme'],
            'custom_css'       => $_POST['custom_css'],
            'promo_banner'     => $_POST['promo_banner']
        ];

        foreach ($settings as $name => $value) {
            update_query('mod_cart_customizer_settings', [
                'setting_value' => $value
            ], ['setting_name' => $name]);
        }

        echo '<div class="alert alert-success">Settings saved successfully!</div>';
    }

    // Load current settings
    $currentSettings = [];
    $result = select_query('mod_cart_customizer_settings', '*', '');
    while ($row = mysql_fetch_array($result)) {
        $currentSettings[$row['setting_name']] = $row['setting_value'];
    }

    // Get analytics data
    $analytics = [];
    $result = select_query('mod_cart_customizer_analytics', 'COUNT(*) as total', '');
    $row = mysql_fetch_array($result);
    $analytics['total_actions'] = $row['total'];

    $result = select_query('mod_cart_customizer_analytics', 
        'action, COUNT(*) as count GROUP BY action', '', '', '', '', 'action');
    while ($row = mysql_fetch_array($result)) {
        $analytics['by_action'][$row['action']] = $row['count'];
    }

    echo <<<HTML
    <div class="whmcs-module-admin">
        <h2>Cart Customizer Settings</h2>
        
        <div class="module-tabs">
            <button class="tab-btn active" data-tab="settings">Settings</button>
            <button class="tab-btn" data-tab="analytics">Analytics</button>
            <button class="tab-btn" data-tab="preview">Preview</button>
        </div>

        <div id="settings" class="tab-content active">
            <form method="post" action="">
                <div class="form-group">
                    <label>Default Product Layout</label>
                    <select name="layout" class="form-control">
                        <option value="grid" {$selected(grid)}>Grid View</option>
                        <option value="list" {$selected(list)}>List View</option>
                        <option value="compact" {$selected(compact)}>Compact View</option>
                        <option value="detailed" {$selected(detailed)}>Detailed View</option>
                    </select>
                </div>

                <div class="form-group">
                    <label>
                        <input type="checkbox" name="show_mini_cart" value="1" 
                            {$checked($currentSettings.show_mini_cart)} />
                        Enable Mini-Cart Widget
                    </label>
                </div>

                <div class="form-group">
                    <label>
                        <input type="checkbox" name="animations" value="1"
                            {$checked($currentSettings.animations)} />
                        Enable Add-to-Cart Animations
                    </label>
                </div>

                <div class="form-group">
                    <label>Cart Theme</label>
                    <select name="cart_theme" class="form-control">
                        <option value="default" {$selected(default)}>Default WHMCS</option>
                        <option value="minimal" {$selected(minimal)}>Minimal</option>
                        <option value="modern" {$selected(modern)}>Modern</option>
                        <option value="elegant" {$selected(elegant)}>Elegant</option>
                        <option value="dark" {$selected(dark)}>Dark Mode</option>
                    </select>
                </div>

                <div class="form-group">
                    <label>Custom CSS</label>
                    <textarea name="custom_css" class="form-control" rows="10"
                        placeholder="Enter custom CSS rules...">{$currentSettings.custom_css}</textarea>
                </div>

                <div class="form-group">
                    <label>Promotional Banner HTML</label>
                    <input type="text" name="promo_banner" class="form-control"
                        value="{$currentSettings.promo_banner}" />
                </div>

                <button type="submit" name="save_settings" class="btn btn-primary">
                    Save Settings
                </button>
            </form>
        </div>

        <div id="analytics" class="tab-content" style="display:none;">
            <h3>Cart Analytics</h3>
            <div class="stats-grid">
                <div class="stat-box">
                    <h4>Total Actions</h4>
                    <p class="stat-number">{$analytics.total_actions}</p>
                </div>
            </div>
            <table class="datatable">
                <thead>
                    <tr>
                        <th>Action</th>
                        <th>Count</th>
                    </tr>
                </thead>
                <tbody>
HTML;

    if (isset($analytics['by_action'])) {
        foreach ($analytics['by_action'] as $action => $count) {
            echo "<tr><td>{$action}</td><td>{$count}</td></tr>";
        }
    }

    echo <<<HTML
                </tbody>
            </table>
        </div>
    </div>

    <script>
        document.querySelectorAll('.tab-btn').forEach(function(btn) {
            btn.addEventListener('click', function() {
                document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
                document.querySelectorAll('.tab-content').forEach(c => c.style.display = 'none');
                btn.classList.add('active');
                document.getElementById(btn.dataset.tab).style.display = 'block';
            });
        });
    </script>
HTML;
}
```

## Hooks Implementation (CartCustomizerHooks.php)

```php
<?php
/**
 * Cart Customizer Hooks
 */

use WHMCS\Cart\Cart;
use WHMCS\Product Bundles;

/**
 * Pre-cart display hook
 */
add_hook('ClientAreaPageCart', 1, function($vars) {
    $settings = getCartCustomizerSettings();
    
    return [
        'cartCustomizerLayout' => $settings['layout'],
        'cartCustomizerTheme' => $settings['cart_theme'],
        'showMiniCart' => $settings['show_mini_cart'] == '1',
        'animationsEnabled' => $settings['animations'] == '1',
        'promoBanner' => $settings['promo_banner']
    ];
});

/**
 * Add to cart hook - for analytics
 */
add_hook('AddToCart', 1, function($vars) {
    logCartAction('add_to_cart', $vars['pid'], $vars['qty']);
});

/**
 * Remove from cart hook - for analytics
 */
add_hook('RemoveFromCart', 1, function($vars) {
    logCartAction('remove_from_cart', $vars['pid'], 1);
});

/**
 * Cart checkout initiated
 */
add_hook('CheckoutCompleted', 1, function($vars) {
    logCartAction('checkout', 0, 0);
});

/**
 * Inject custom CSS
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $settings = getCartCustomizerSettings();
    $customCss = $settings['custom_css'];
    
    if (!empty($customCss)) {
        return '<style>' . $customCss . '</style>';
    }
    return '';
});

/**
 * Custom cart template
 */
add_hook('ClientAreaPageCartOutputOverride', 1, function($vars) {
    $settings = getCartCustomizerSettings();
    
    if ($settings['layout'] !== 'default') {
        return [
            'templateFile' => 'cart-customizer/view',
            'vars' => [
                'customizerLayout' => $settings['layout'],
                'showProductImages' => true,
                'showMiniCart' => $settings['show_mini_cart'] == '1'
            ]
        ];
    }
});

/**
 * Helper: Get cart customizer settings
 */
function getCartCustomizerSettings()
{
    $settings = [];
    $result = select_query('mod_cart_customizer_settings', '*', '');
    while ($row = mysql_fetch_array($result)) {
        $settings[$row['setting_name']] = $row['setting_value'];
    }
    
    // Defaults
    $defaults = [
        'layout' => 'grid',
        'show_mini_cart' => '1',
        'animations' => '1',
        'cart_theme' => 'default',
        'custom_css' => '',
        'promo_banner' => ''
    ];
    
    return array_merge($defaults, $settings);
}

/**
 * Helper: Log cart action
 */
function logCartAction($action, $productId, $quantity)
{
    $userId = (int)($_SESSION['uid'] ?? 0);
    $sessionId = session_id();
    
    insert_query('mod_cart_customizer_analytics', [
        'user_id'       => $userId > 0 ? $userId : null,
        'session_id'    => $sessionId,
        'action'        => $action,
        'product_id'    => $productId > 0 ? $productId : null,
        'quantity'      => $quantity
    ]);
}
```

## Cart Template (view.tpl)

```smarty
<div class="cart-customizer" data-layout="{$customizerLayout}">
    {if $promoBanner}
        <div class="promo-banner">
            {$promoBanner}
        </div>
    {/if}

    <div class="cart-products">
        {foreach from=$products item=product}
            <div class="cart-product-item" data-product-id="{$product.id}">
                {if $showProductImages && $product.image}
                    <div class="product-image">
                        <img src="{$product.image}" alt="{$product.name}" />
                    </div>
                {/if}
                
                <div class="product-details">
                    <h3 class="product-name">{$product.name}</h3>
                    {if $product.description && $showProductDesc}
                        <p class="product-description">{$product.description}</p>
                    {/if}
                    
                    <div class="product-options">
                        {foreach from=$product.options item=option}
                            <span class="option-label">{$option.name}: {$option.value}</span>
                        {/foreach}
                    </div>
                    
                    <div class="product-pricing">
                        <span class="product-price">{$product.price}</span>
                        {if $product.recurring}
                            <span class="product-billing-cycle">{$product.billingcycle}</span>
                        {/if}
                    </div>
                </div>
                
                <div class="product-actions">
                    <div class="quantity-selector">
                        <button class="qty-decrease" data-action="decrease">-</button>
                        <input type="number" name="qty[{$product.id}]" value="{$product.qty}" min="1" />
                        <button class="qty-increase" data-action="increase">+</button>
                    </div>
                    
                    <button class="btn-remove" data-product-id="{$product.id}">
                        Remove
                    </button>
                </div>
            </div>
        {/foreach}
    </div>

    <div class="cart-summary">
        <div class="summary-row">
            <span>Subtotal:</span>
            <span class="summary-value">{$subtotal}</span>
        </div>
        
        {foreach from=$addons item=addon}
            <div class="summary-row">
                <span>{$addon.name}:</span>
                <span class="summary-value">{$addon.price}</span>
            </div>
        {/foreach}
        
        {if $discounts}
            <div class="summary-row discount">
                <span>Discount:</span>
                <span class="summary-value">-{$discount}</span>
            </div>
        {/if}
        
        <div class="summary-row total">
            <span>Total:</span>
            <span class="summary-value">{$total}</span>
        </div>
    </div>

    <div class="cart-actions">
        <a href="cart.php?a=confproduct" class="btn btn-primary">Continue Shopping</a>
        <a href="cart.php?a=view" class="btn btn-secondary">Update Cart</a>
        <a href="cart.php?a=checkout" class="btn btn-checkout">Checkout</a>
    </div>
</div>

{if $showMiniCart}
    {include file="$template/mini-cart.tpl"}
{/if}
```

## Mini-Cart Widget Template (mini-cart.tpl)

```smarty
<div class="mini-cart-widget" id="miniCartWidget">
    <div class="mini-cart-toggle" id="miniCartToggle">
        <i class="fa fa-shopping-cart"></i>
        <span class="cart-count" id="miniCartCount">{$cartItemCount}</span>
    </div>
    
    <div class="mini-cart-dropdown" id="miniCartDropdown" style="display:none;">
        <div class="mini-cart-header">
            <h4>Your Cart</h4>
            <button class="close-dropdown">&times;</button>
        </div>
        
        <div class="mini-cart-items">
            {foreach from=$cartItems item=item}
                <div class="mini-cart-item">
                    <span class="item-name">{$item.name}</span>
                    <span class="item-price">{$item.price}</span>
                </div>
            {/foreach}
        </div>
        
        <div class="mini-cart-footer">
            <div class="mini-cart-total">
                <span>Total:</span>
                <span>{$cartTotal}</span>
            </div>
            <a href="cart.php" class="btn btn-sm btn-primary">View Cart</a>
            <a href="cart.php?a=checkout" class="btn btn-sm btn-success">Checkout</a>
        </div>
    </div>
</div>
```

## CSS Styles (customizer.css)

```css
/* Cart Customizer Styles */
.cart-customizer {
    --cc-primary: #007bff;
    --cc-secondary: #6c757d;
    --cc-success: #28a745;
    --cc-border: #dee2e6;
}

.cart-customizer .promo-banner {
    background: var(--cc-primary);
    color: white;
    padding: 15px;
    text-align: center;
    margin-bottom: 20px;
    border-radius: 4px;
}

/* Product Layout */
.cart-customizer[data-layout="grid"] .cart-products {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
    gap: 20px;
}

.cart-customizer[data-layout="list"] .cart-product-item {
    display: flex;
    flex-direction: row;
    align-items: center;
}

.cart-customizer[data-layout="compact"] .cart-product-item {
    padding: 10px;
}

.cart-customizer[data-layout="detailed"] .cart-product-item {
    padding: 20px;
    background: #f8f9fa;
}

/* Product Item */
.cart-product-item {
    border: 1px solid var(--cc-border);
    border-radius: 8px;
    padding: 15px;
    margin-bottom: 15px;
    background: white;
    transition: box-shadow 0.3s ease;
}

.cart-product-item:hover {
    box-shadow: 0 4px 12px rgba(0,0,0,0.1);
}

.cart-product-item .product-image img {
    max-width: 100%;
    height: auto;
    border-radius: 4px;
}

.cart-product-item .product-name {
    font-size: 18px;
    margin: 10px 0;
    color: #333;
}

.cart-product-item .product-description {
    color: #666;
    font-size: 14px;
}

/* Quantity Selector */
.quantity-selector {
    display: flex;
    align-items: center;
    gap: 5px;
}

.quantity-selector input {
    width: 60px;
    text-align: center;
    padding: 8px;
    border: 1px solid var(--cc-border);
    border-radius: 4px;
}

.quantity-selector button {
    width: 32px;
    height: 32px;
    border: 1px solid var(--cc-border);
    background: white;
    cursor: pointer;
    border-radius: 4px;
}

.quantity-selector button:hover {
    background: var(--cc-primary);
    color: white;
    border-color: var(--cc-primary);
}

/* Mini Cart Widget */
.mini-cart-widget {
    position: fixed;
    bottom: 20px;
    right: 20px;
    z-index: 9999;
}

.mini-cart-toggle {
    width: 60px;
    height: 60px;
    background: var(--cc-primary);
    color: white;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    box-shadow: 0 4px 12px rgba(0,0,0,0.2);
    position: relative;
}

.mini-cart-toggle:hover {
    transform: scale(1.1);
}

.mini-cart-toggle .cart-count {
    position: absolute;
    top: -5px;
    right: -5px;
    background: var(--cc-success);
    color: white;
    font-size: 12px;
    width: 20px;
    height: 20px;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
}

.mini-cart-dropdown {
    position: absolute;
    bottom: 70px;
    right: 0;
    width: 320px;
    background: white;
    border-radius: 8px;
    box-shadow: 0 8px 24px rgba(0,0,0,0.15);
    overflow: hidden;
}

.mini-cart-header {
    padding: 15px;
    border-bottom: 1px solid var(--cc-border);
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.mini-cart-items {
    max-height: 300px;
    overflow-y: auto;
    padding: 10px;
}

.mini-cart-item {
    padding: 10px;
    border-bottom: 1px solid var(--cc-border);
    display: flex;
    justify-content: space-between;
}

.mini-cart-item:last-child {
    border-bottom: none;
}

.mini-cart-footer {
    padding: 15px;
    background: #f8f9fa;
}

/* Cart Summary */
.cart-summary {
    background: #f8f9fa;
    padding: 20px;
    border-radius: 8px;
    margin: 20px 0;
}

.summary-row {
    display: flex;
    justify-content: space-between;
    padding: 8px 0;
    border-bottom: 1px solid var(--cc-border);
}

.summary-row.discount {
    color: var(--cc-success);
}

.summary-row.total {
    font-size: 18px;
    font-weight: bold;
    border-bottom: none;
    padding-top: 15px;
}

/* Cart Actions */
.cart-actions {
    display: flex;
    gap: 10px;
    justify-content: center;
    flex-wrap: wrap;
}

.cart-actions .btn {
    padding: 12px 24px;
    border-radius: 4px;
    text-decoration: none;
    font-weight: 500;
}

.cart-actions .btn-primary {
    background: var(--cc-primary);
    color: white;
}

.cart-actions .btn-secondary {
    background: var(--cc-secondary);
    color: white;
}

.cart-actions .btn-checkout {
    background: var(--cc-success);
    color: white;
}

/* Animations */
@keyframes addToCartSuccess {
    0% { transform: scale(1); }
    50% { transform: scale(1.2); }
    100% { transform: scale(1); }
}

.cart-product-item.adding {
    animation: addToCartSuccess 0.5s ease;
}

/* Theme Variations */
.cart-customizer.theme-minimal {
    --cc-primary: #333;
}

.cart-customizer.theme-modern {
    --cc-primary: #2196F3;
    --cc-border-radius: 0;
}

.cart-customizer.theme-elegant {
    --cc-primary: #9c27b0;
    --cc-border-radius: 16px;
}

.cart-customizer.theme-dark {
    --cc-primary: #1a1a2e;
    --cc-secondary: #16213e;
    --cc-success: #e94560;
    --cc-border: #333;
}
```

## JavaScript (customizer.js)

```javascript
/**
 * Cart Customizer JavaScript
 */

(function() {
    'use strict';

    // Initialize when DOM is ready
    document.addEventListener('DOMContentLoaded', function() {
        initMiniCart();
        initQuantitySelectors();
        initAnimations();
        initProductLayout();
    });

    /**
     * Mini Cart Toggle
     */
    function initMiniCart() {
        const toggle = document.getElementById('miniCartToggle');
        const dropdown = document.getElementById('miniCartDropdown');
        const closeBtn = dropdown.querySelector('.close-dropdown');

        if (!toggle || !dropdown) return;

        toggle.addEventListener('click', function() {
            const isVisible = dropdown.style.display !== 'none';
            dropdown.style.display = isVisible ? 'none' : 'block';
        });

        if (closeBtn) {
            closeBtn.addEventListener('click', function() {
                dropdown.style.display = 'none';
            });
        }

        // Close on click outside
        document.addEventListener('click', function(e) {
            if (!dropdown.contains(e.target) && !toggle.contains(e.target)) {
                dropdown.style.display = 'none';
            }
        });
    }

    /**
     * Quantity Selectors
     */
    function initQuantitySelectors() {
        document.querySelectorAll('.quantity-selector').forEach(function(selector) {
            const decreaseBtn = selector.querySelector('.qty-decrease');
            const increaseBtn = selector.querySelector('.qty-increase');
            const input = selector.querySelector('input');

            if (!decreaseBtn || !increaseBtn || !input) return;

            decreaseBtn.addEventListener('click', function() {
                const currentValue = parseInt(input.value) || 1;
                if (currentValue > 1) {
                    input.value = currentValue - 1;
                    updateCartQuantity(input);
                }
            });

            increaseBtn.addEventListener('click', function() {
                const currentValue = parseInt(input.value) || 0;
                input.value = currentValue + 1;
                updateCartQuantity(input);
            });
        });
    }

    /**
     * Update Cart Quantity via AJAX
     */
    function updateCartQuantity(input) {
        const productId = input.name.match(/qty\[(\d+)\]/);
        if (!productId) return;

        const quantity = parseInt(input.value);
        const xhr = new XMLHttpRequest();
        
        xhr.open('POST', 'cart.php?a=update', true);
        xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
        xhr.send('pid=' + productId[1] + '&qty=' + quantity);

        xhr.onload = function() {
            if (xhr.status === 200) {
                // Update UI with response
                location.reload();
            }
        };
    }

    /**
     * Add to Cart Animation
     */
    function initAnimations() {
        const animationsEnabled = document.body.dataset.animations === 'true';
        if (!animationsEnabled) return;

        document.querySelectorAll('.btn-add-to-cart').forEach(function(btn) {
            btn.addEventListener('click', function(e) {
                const productItem = btn.closest('.cart-product-item');
                if (productItem) {
                    productItem.classList.add('adding');
                    setTimeout(function() {
                        productItem.classList.remove('adding');
                    }, 500);
                }
            });
        });
    }

    /**
     * Product Layout Handler
     */
    function initProductLayout() {
        const cartCustomizer = document.querySelector('.cart-customizer');
        if (!cartCustomizer) return;

        const layout = cartCustomizer.dataset.layout;
        
        // Add layout class for CSS styling
        cartCustomizer.classList.add('layout-' + layout);
    }

    /**
     * Update mini cart count
     */
    window.updateMiniCartCount = function(count) {
        const countEl = document.getElementById('miniCartCount');
        if (countEl) {
            countEl.textContent = count;
            countEl.style.transform = 'scale(1.3)';
            setTimeout(function() {
                countEl.style.transform = 'scale(1)';
            }, 200);
        }
    };

})();
```

## Database Schema

```sql
-- Settings table
CREATE TABLE `mod_cart_customizer_settings` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `setting_name` VARCHAR(100) NOT NULL,
    `setting_value` TEXT,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `setting_name` (`setting_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Analytics table
CREATE TABLE `mod_cart_customizer_analytics` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `user_id` INT(11) DEFAULT NULL,
    `session_id` VARCHAR(128) NOT NULL,
    `action` VARCHAR(50) NOT NULL,
    `product_id` INT(11) DEFAULT NULL,
    `quantity` INT(11) DEFAULT 1,
    `timestamp` DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    KEY `session_id` (`session_id`),
    KEY `action` (`action`),
    KEY `user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Analytics queries
INSERT INTO `mod_cart_customizer_settings` (`setting_name`, `setting_value`) VALUES
('layout', 'grid'),
('show_mini_cart', '1'),
('animations', '1'),
('cart_theme', 'default'),
('custom_css', ''),
('promo_banner', '');
```

## Hooks Reference

| Hook Name | Priority | Purpose |
|-----------|----------|---------|
| `ClientAreaPageCart` | 1 | Inject customizer settings to cart page |
| `AddToCart` | 1 | Log add-to-cart events |
| `RemoveFromCart` | 1 | Log remove-from-cart events |
| `CheckoutCompleted` | 1 | Log checkout events |
| `ClientAreaHeadOutput` | 1 | Inject custom CSS |
| `ClientAreaPageCartOutputOverride` | 1 | Override cart template |

## Configuration Options

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `layout` | dropdown | grid | Product display layout |
| `show_mini_cart` | yesno | on | Enable mini-cart widget |
| `animations` | yesno | on | Enable visual animations |
| `cart_theme` | dropdown | default | Color scheme |
| `custom_css` | textarea | - | Custom CSS rules |
| `promo_banner` | text | - | Promotional banner HTML |

## Changelog

### Version 1.0.0
- Initial release
- Grid/List/Compact/Detailed layouts
- Mini-cart widget
- Add-to-cart animations
- Analytics tracking
- Custom CSS support
- Multiple theme options