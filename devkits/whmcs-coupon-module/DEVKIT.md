# WHMCS Coupon Module DevKit

## Header

**Purpose:** Discount coupon system providing customizable promotional codes, automatic discounts, usage limits, and campaign tracking for marketing promotions.

**Module Type:** Marketing/Discount Module

**Use Case:** Hosting companies running promotional campaigns, offering first-time discounts, or providing loyalty rewards through coupon codes.

---

## Complete Code Template

### File Structure
```
whmcs-coupon-module/
├── README.md
├── DEVKIT.md
├── coupon.php          # Main coupon logic
├── hooks.php           # WHMCS hook integrations
├── campaign.php        # Campaign management
└── templates/
    └── admin_coupon.tpl
```

### Main Module File: coupon.php

```php
<?php
/**
 * WHMCS Coupon Module
 * 
 * Provides discount coupon and promotional code system.
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Module Configuration
define('COUPON_MODULE_VERSION', '1.0.0');

// Coupon Types
define('COUPON_TYPE_PERCENTAGE', 'percentage');
define('COUPON_TYPE_FIXED', 'fixed');
define('COUPON_TYPE_FREE_SETUP', 'free_setup');
define('COUPON_TYPE_FREE_MONTH', 'free_month');

// Coupon Status
define('COUPON_STATUS_ACTIVE', 'active');
define('COUPON_STATUS_EXPIRED', 'expired');
define('COUPON_STATUS_DISABLED', 'disabled');
define('COUPON_STATUS_DEPLETED', 'depleted');

// Discount Applies To
define('COUPON_APPLY_PRODUCT', 'product');
define('COUPON_APPLY_CATEGORY', 'category');
define('COUPON_APPLY_ORDER', 'order');
define('COUPON_APPLY_ALL', 'all');

/**
 * Validate coupon code
 */
function coupon_validate($code, $clientId = 0, $cartData = []) {
    $coupon = coupon_get_by_code($code);
    
    if (!$coupon) {
        return [
            'valid' => false,
            'error' => 'Invalid coupon code'
        ];
    }
    
    // Check if coupon is active
    if ($coupon['status'] != COUPON_STATUS_ACTIVE) {
        return [
            'valid' => false,
            'error' => coupon_get_status_error($coupon['status'])
        ];
    }
    
    // Check expiration
    if ($coupon['expiry_date'] && $coupon['expiry_date'] < date('Y-m-d')) {
        return [
            'valid' => false,
            'error' => 'This coupon has expired'
        ];
    }
    
    // Check start date
    if ($coupon['start_date'] && $coupon['start_date'] > date('Y-m-d')) {
        return [
            'valid' => false,
            'error' => 'This coupon is not yet active'
        ];
    }
    
    // Check usage limit
    if ($coupon['max_uses'] > 0 && $coupon['uses'] >= $coupon['max_uses']) {
        return [
            'valid' => false,
            'error' => 'This coupon has reached its usage limit'
        ];
    }
    
    // Check client usage limit
    if ($coupon['max_uses_client'] > 0 && $clientId > 0) {
        $clientUses = coupon_get_client_uses($code, $clientId);
        if ($clientUses >= $coupon['max_uses_client']) {
            return [
                'valid' => false,
                'error' => 'You have already used this coupon the maximum number of times'
            ];
        }
    }
    
    // Check minimum order value
    if ($coupon['min_order_value'] > 0) {
        $cartTotal = $cartData['total'] ?? 0;
        if ($cartTotal < $coupon['min_order_value']) {
            return [
                'valid' => false,
                'error' => 'Minimum order value of ' . formatCurrency($coupon['min_order_value']) . ' required'
            ];
        }
    }
    
    // Check products applicability
    if (!empty($coupon['applies_to'])) {
        $appliesToProducts = json_decode($coupon['applies_to'], true);
        if (!empty($appliesToProducts['products'])) {
            $cartProducts = $cartData['products'] ?? [];
            $hasValidProduct = false;
            
            foreach ($cartProducts as $productId) {
                if (in_array($productId, $appliesToProducts['products'])) {
                    $hasValidProduct = true;
                    break;
                }
            }
            
            if (!$hasValidProduct) {
                return [
                    'valid' => false,
                    'error' => 'This coupon does not apply to items in your cart'
                ];
            }
        }
    }
    
    // Check client group restriction
    if ($coupon['client_group_id'] && $clientId > 0) {
        $clientGroup = coupon_get_client_group($clientId);
        if ($clientGroup != $coupon['client_group_id']) {
            return [
                'valid' => false,
                'error' => 'This coupon is not available for your client group'
            ];
        }
    }
    
    return [
        'valid' => true,
        'coupon' => $coupon
    ];
}

/**
 * Get coupon by code
 */
function coupon_get_by_code($code) {
    $query = "SELECT * FROM mod_coupons WHERE code = ?";
    $result = full_query($query, [strtoupper($code)]);
    return mysql_fetch_assoc($result);
}

/**
 * Get status error message
 */
function coupon_get_status_error($status) {
    switch ($status) {
        case COUPON_STATUS_EXPIRED:
            return 'This coupon has expired';
        case COUPON_STATUS_DISABLED:
            return 'This coupon has been disabled';
        case COUPON_STATUS_DEPLETED:
            return 'This coupon has reached its usage limit';
        default:
            return 'This coupon is not available';
    }
}

/**
 * Get client uses for a coupon
 */
function coupon_get_client_uses($code, $clientId) {
    $query = "SELECT COUNT(*) as uses FROM mod_coupon_uses 
              WHERE coupon_code = ? AND client_id = ?";
    $result = full_query($query, [$code, $clientId]);
    $data = mysql_fetch_assoc($result);
    return $data['uses'] ?? 0;
}

/**
 * Get client group ID
 */
function coupon_get_client_group($clientId) {
    $query = "SELECT groupid FROM tblclients WHERE id = ?";
    $result = full_query($query, [$clientId]);
    $data = mysql_fetch_assoc($result);
    return $data['groupid'] ?? 0;
}

/**
 * Calculate discount amount
 */
function coupon_calculate_discount($coupon, $amount, $productId = 0) {
    switch ($coupon['type']) {
        case COUPON_TYPE_PERCENTAGE:
            return round($amount * ($coupon['value'] / 100), 2);
            
        case COUPON_TYPE_FIXED:
            // Ensure discount doesn't exceed amount
            return min($coupon['value'], $amount);
            
        case COUPON_TYPE_FREE_SETUP:
            // Return setup fee amount (would need product data)
            return $coupon['value'];
            
        case COUPON_TYPE_FREE_MONTH:
            // Return one month's worth of billing
            return $amount; // Full amount free for one month
            
        default:
            return 0;
    }
}

/**
 * Apply coupon to order
 */
function coupon_apply($code, $clientId, $orderId = null) {
    $validation = coupon_validate($code, $clientId);
    
    if (!$validation['valid']) {
        return $validation;
    }
    
    $coupon = $validation['coupon'];
    
    // Record usage
    coupon_record_usage($code, $clientId, $orderId);
    
    // Update usage count
    coupon_increment_uses($code);
    
    return [
        'valid' => true,
        'discount' => $coupon['type'],
        'value' => $coupon['value'],
        'code' => $code
    ];
}

/**
 * Record coupon usage
 */
function coupon_record_usage($code, $clientId, $orderId = null) {
    $data = [
        'coupon_code' => $code,
        'client_id' => $clientId,
        'order_id' => $orderId,
        'used_at' => date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_coupon_uses', $data);
}

/**
 * Increment coupon usage count
 */
function coupon_increment_uses($code) {
    $query = "UPDATE mod_coupons SET uses = uses + 1 WHERE code = ?";
    full_query($query, [$code]);
    
    // Check if depleted
    $coupon = coupon_get_by_code($code);
    if ($coupon['max_uses'] > 0 && $coupon['uses'] >= $coupon['max_uses']) {
        update_query('mod_coupons', ['status' => COUPON_STATUS_DEPLETED], ['code' => $code]);
    }
}

/**
 * Create new coupon
 */
function coupon_create($data) {
    $fields = [
        'code', 'name', 'type', 'value', 'max_uses', 'uses', 'max_uses_client',
        'min_order_value', 'applies_to', 'client_group_id', 'start_date',
        'expiry_date', 'is_recurring', 'recurring_months', 'priority',
        'description', 'status', 'created_at'
    ];
    
    $values = [
        strtoupper($data['code']), $data['name'], $data['type'],
        $data['value'], $data['max_uses'] ?? 0, 0,
        $data['max_uses_client'] ?? 1, $data['min_order_value'] ?? 0,
        json_encode($data['applies_to'] ?? []), $data['client_group_id'] ?? 0,
        $data['start_date'] ?? date('Y-m-d'), $data['expiry_date'] ?? null,
        $data['is_recurring'] ?? 0, $data['recurring_months'] ?? 0,
        $data['priority'] ?? 50, $data['description'] ?? '',
        COUPON_STATUS_ACTIVE, date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_coupons', array_combine($fields, $values));
    return mysql_insert_id();
}

/**
 * Update coupon
 */
function coupon_update($couponId, $data) {
    $allowedFields = ['name', 'value', 'max_uses', 'min_order_value',
                     'applies_to', 'client_group_id', 'start_date',
                     'expiry_date', 'status', 'description'];
    
    $updateData = [];
    foreach ($allowedFields as $field) {
        if (isset($data[$field])) {
            if ($field === 'applies_to') {
                $updateData[$field] = json_encode($data[$field]);
            } else {
                $updateData[$field] = $data[$field];
            }
        }
    }
    
    $updateData['updated_at'] = date('Y-m-d H:i:s');
    update_query('mod_coupons', $updateData, ['id' => $couponId]);
}

/**
 * Delete coupon
 */
function coupon_delete($couponId) {
    // Soft delete by disabling
    update_query('mod_coupons', ['status' => COUPON_STATUS_DISABLED], ['id' => $couponId]);
}

/**
 * Get all coupons
 */
function coupon_get_all($filters = []) {
    $where = "1=1";
    $params = [];
    
    if (!empty($filters['status'])) {
        $where .= " AND status = ?";
        $params[] = $filters['status'];
    }
    
    if (!empty($filters['type'])) {
        $where .= " AND type = ?";
        $params[] = $filters['type'];
    }
    
    if (!empty($filters['search'])) {
        $where .= " AND (code LIKE ? OR name LIKE ?)";
        $params[] = '%' . $filters['search'] . '%';
        $params[] = '%' . $filters['search'] . '%';
    }
    
    $query = "SELECT * FROM mod_coupons WHERE {$where} ORDER BY created_at DESC";
    $result = full_query($query, $params);
    
    $coupons = [];
    while ($row = mysql_fetch_assoc($result)) {
        $row['applies_to'] = json_decode($row['applies_to'], true);
        $coupons[] = $row;
    }
    
    return $coupons;
}

/**
 * Get coupon usage history
 */
function coupon_get_history($couponId) {
    $query = "SELECT cu.*, c.firstname, c.lastname, o.id as order_id
              FROM mod_coupon_uses cu
              LEFT JOIN tblclients c ON cu.client_id = c.id
              LEFT JOIN tblorders o ON cu.order_id = o.id
              WHERE cu.coupon_id = ?
              ORDER BY cu.used_at DESC";
    $result = full_query($query, [$couponId]);
    
    $history = [];
    while ($row = mysql_fetch_assoc($result)) {
        $history[] = $row;
    }
    
    return $history;
}

/**
 * Generate coupon code
 */
function coupon_generate_code($prefix = '', $length = 8) {
    $chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';
    $code = $prefix;
    
    for ($i = 0; $i < $length; $i++) {
        $code .= $chars[rand(0, strlen($chars) - 1)];
    }
    
    // Ensure unique
    $existing = coupon_get_by_code($code);
    if ($existing) {
        return coupon_generate_code($prefix, $length);
    }
    
    return $code;
}

/**
 * Bulk create coupons
 */
function coupon_bulk_create($codes, $data) {
    $created = 0;
    
    foreach ($codes as $code) {
        $data['code'] = $code;
        coupon_create($data);
        $created++;
    }
    
    return $created;
}

/**
 * Get coupon analytics
 */
function coupon_get_analytics($startDate = null, $endDate = null) {
    $dateCondition = "";
    $params = [];
    
    if ($startDate && $endDate) {
        $dateCondition = "WHERE cu.used_at BETWEEN ? AND ?";
        $params = [$startDate, $endDate];
    }
    
    $query = "SELECT 
                c.id, c.code, c.name, c.type, c.value,
                COUNT(cu.id) as total_uses,
                SUM(o.amount) as total_revenue,
                COUNT(DISTINCT cu.client_id) as unique_clients
              FROM mod_coupons c
              LEFT JOIN mod_coupon_uses cu ON c.code = cu.coupon_code
              LEFT JOIN tblorders o ON cu.order_id = o.id
              {$dateCondition}
              GROUP BY c.id
              ORDER BY total_uses DESC";
    
    $result = full_query($query, $params);
    
    $analytics = [];
    while ($row = mysql_fetch_assoc($result)) {
        $analytics[] = $row;
    }
    
    return $analytics;
}

/**
 * Validate coupon configuration
 */
function coupon_validate_config($data, &$errors) {
    if (empty($data['code'])) {
        $errors[] = "Coupon code is required";
    } else {
        // Check for duplicate
        $existing = coupon_get_by_code($data['code']);
        if ($existing && empty($data['id'])) {
            $errors[] = "This coupon code already exists";
        }
    }
    
    if (empty($data['name'])) {
        $errors[] = "Coupon name is required";
    }
    
    if (empty($data['type'])) {
        $errors[] = "Coupon type is required";
    }
    
    if (!isset($data['value']) && $data['type'] != COUPON_TYPE_FREE_MONTH) {
        $errors[] = "Coupon value is required";
    }
    
    if ($data['type'] == COUPON_TYPE_PERCENTAGE && $data['value'] > 100) {
        $errors[] = "Percentage cannot exceed 100%";
    }
    
    if (!empty($data['start_date']) && !empty($data['expiry_date'])) {
        if (strtotime($data['expiry_date']) < strtotime($data['start_date'])) {
            $errors[] = "Expiry date must be after start date";
        }
    }
    
    return count($errors) === 0;
}

/**
 * Get coupon by ID
 */
function coupon_get($couponId) {
    $query = "SELECT * FROM mod_coupons WHERE id = ?";
    $result = full_query($query, [$couponId]);
    $coupon = mysql_fetch_assoc($result);
    if ($coupon) {
        $coupon['applies_to'] = json_decode($coupon['applies_to'], true);
    }
    return $coupon;
}

/**
 * Clone coupon
 */
function coupon_clone($couponId, $newCode = null) {
    $coupon = coupon_get($couponId);
    
    if (!$coupon) {
        return false;
    }
    
    $newData = $coupon;
    $newData['code'] = $newCode ?? coupon_generate_code();
    $newData['uses'] = 0;
    $newData['status'] = COUPON_STATUS_ACTIVE;
    unset($newData['id']);
    
    return coupon_create($newData);
}
```

### Hooks File: hooks.php

```php
<?php
/**
 * WHMCS Coupon Module Hooks
 */

// Hook: Validate coupon in cart
add_hook('ShoppingCartValidateCoupon', 1, function($params) {
    $code = $params['code'];
    $clientId = $_SESSION['uid'] ?? 0;
    
    // Get cart data
    $cartData = [
        'total' => $_SESSION['cart']['total'] ?? 0,
        'products' => array_column($_SESSION['cart']['products'] ?? [], 'product_id')
    ];
    
    $result = coupon_validate($code, $clientId, $cartData);
    
    if (!$result['valid']) {
        return [
            'success' => false,
            'error' => $result['error']
        ];
    }
    
    return [
        'success' => true,
        'coupon' => $result['coupon']
    ];
});

// Hook: Apply coupon discount to cart
add_hook('ShoppingCartDiscountOverride', 1, function($params) {
    $code = $_SESSION['coupon_code'] ?? null;
    
    if (!$code) {
        return [];
    }
    
    $clientId = $_SESSION['uid'] ?? 0;
    $cartTotal = $params['total'];
    
    $coupon = coupon_get_by_code($code);
    if (!$coupon) {
        return [];
    }
    
    $discount = coupon_calculate_discount($coupon, $cartTotal);
    
    return [
        'discount' => $discount,
        'description' => 'Coupon: ' . $coupon['name'] . ' (' . $coupon['code'] . ')'
    ];
});

// Hook: Apply coupon on order placement
add_hook('OrderCreated', 1, function($params) {
    $code = $_SESSION['coupon_code'] ?? null;
    
    if (!$code) {
        return [];
    }
    
    $clientId = $params['clientId'];
    $orderId = $params['orderId'];
    
    coupon_apply($code, $clientId, $orderId);
    
    // Clear coupon from session
    unset($_SESSION['coupon_code']);
});

// Hook: Apply recurring discount for subscription coupons
add_hook('ServiceRenew', 1, function($params) {
    $serviceId = $params['serviceId'];
    $clientId = $params['clientId'];
    
    // Check if service has recurring coupon
    $query = "SELECT coupon_code FROM mod_coupon_services 
              WHERE service_id = ? AND is_recurring = 1
              AND next_apply <= CURDATE()";
    $result = full_query($query, [$serviceId]);
    $data = mysql_fetch_assoc($result);
    
    if ($data && $data['coupon_code']) {
        $coupon = coupon_get_by_code($data['coupon_code']);
        
        if ($coupon && $coupon['is_recurring']) {
            // Apply discount to renewal
            $recurringAmount = get_recurring_amount($serviceId);
            $discount = coupon_calculate_discount($coupon, $recurringAmount);
            
            return [
                'discount' => $discount,
                'description' => 'Recurring coupon: ' . $coupon['name']
            ];
            
            // Update next application date
            $nextDate = date('Y-m-d', strtotime('+' . $coupon['recurring_months'] . ' months'));
            update_query('mod_coupon_services', ['next_apply' => $nextDate], ['service_id' => $serviceId]);
        }
    }
});

// Hook: Handle coupon on invoice creation
add_hook('InvoiceCreationPreCheck', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    // Check for coupon discount line item
    $query = "SELECT * FROM mod_coupon_invoices WHERE invoice_id = ?";
    $result = full_query($query, [$invoiceId]);
    
    while ($couponInvoice = mysql_fetch_assoc($result)) {
        $coupon = coupon_get_by_code($couponInvoice['coupon_code']);
        
        if ($coupon) {
            // Log the application
            logActivity("Coupon {$coupon['code']} applied to invoice #{$invoiceId}");
        }
    }
});

// Hook: Prevent coupon on specific products
add_hook('ShoppingCartValidateCoupon', 2, function($params) {
    $code = $params['code'];
    $coupon = coupon_get_by_code($code);
    
    if ($coupon && !empty($coupon['excluded_products'])) {
        $excludedProducts = json_decode($coupon['excluded_products'], true);
        $cartProducts = $_SESSION['cart']['products'] ?? [];
        
        foreach ($cartProducts as $cartProduct) {
            if (in_array($cartProduct['product_id'], $excludedProducts)) {
                return [
                    'success' => false,
                    'error' => 'This coupon cannot be used with selected products'
                ];
            }
        }
    }
});

// Hook: Track coupon attribution
add_hook('OrderCreated', 2, function($params) {
    $code = $_SESSION['coupon_code'] ?? null;
    
    if ($code) {
        $clientId = $params['clientId'];
        $orderId = $params['orderId'];
        
        // Record attribution
        $query = "INSERT INTO mod_coupon_attribution (coupon_code, client_id, order_id, channel)
                  VALUES (?, ?, ?, ?)";
        full_query($query, [$code, $clientId, $orderId, $_SESSION['coupon_channel'] ?? 'direct']);
    }
});
```

### Admin Template: templates/admin_coupon.tpl

```html
{extends file="admin/template.tpl"}

{block name="content"}
<div class="coupon-admin">
    <div class="header">
        <h2>Coupon Management</h2>
        <div class="actions">
            <button class="btn btn-primary" onclick="showAddCouponModal()">Create Coupon</button>
            <button class="btn btn-secondary" onclick="showBulkCreateModal()">Bulk Create</button>
        </div>
    </div>

    <div class="coupon-filters">
        <input type="text" id="search-coupon" placeholder="Search coupons...">
        <select id="filter-status">
            <option value="">All Status</option>
            <option value="active">Active</option>
            <option value="expired">Expired</option>
            <option value="disabled">Disabled</option>
            <option value="depleted">Depleted</option>
        </select>
        <select id="filter-type">
            <option value="">All Types</option>
            <option value="percentage">Percentage</option>
            <option value="fixed">Fixed Amount</option>
            <option value="free_setup">Free Setup</option>
            <option value="free_month">Free Month</option>
        </select>
        <button class="btn" onclick="filterCoupons()">Filter</button>
    </div>

    <table class="coupon-table">
        <thead>
            <tr>
                <th>Code</th>
                <th>Name</th>
                <th>Type</th>
                <th>Value</th>
                <th>Uses</th>
                <th>Min Order</th>
                <th>Valid Period</th>
                <th>Status</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            {foreach from=$coupons item=c}
            <tr data-coupon-id="{$c.id}">
                <td><strong>{$c.code}</strong></td>
                <td>{$c.name}</td>
                <td><span class="badge badge-{$c.type}">{$c.type}</span></td>
                <td>
                    {if $c.type == 'percentage'}
                        {$c.value}%
                    {else}
                        {$currency_prefix}{$c.value}{$currency_suffix}
                    {/if}
                </td>
                <td>{$c.uses}{if $c.max_uses > 0}/{$c.max_uses}{/if}</td>
                <td>{if $c.min_order_value > 0}{$currency_prefix}{$c.min_order_value}{else}-{/if}</td>
                <td>{$c.start_date} - {$c.expiry_date}</td>
                <td>
                    <span class="status-badge status-{$c.status}">{$c.status}</span>
                </td>
                <td>
                    <button onclick="editCoupon({$c.id})">Edit</button>
                    <button onclick="viewHistory({$c.id})">History</button>
                    <button onclick="cloneCoupon({$c.id})">Clone</button>
                    <button onclick="deleteCoupon({$c.id})">Delete</button>
                </td>
            </tr>
            {/foreach}
        </tbody>
    </table>

    <div class="analytics-section">
        <h3>Coupon Analytics</h3>
        <div class="analytics-grid">
            <div class="analytics-card">
                <h4>Total Coupons Used</h4>
                <p class="analytics-value">{$analytics.total_uses}</p>
            </div>
            <div class="analytics-card">
                <h4>Total Revenue</h4>
                <p class="analytics-value">{$currency_prefix}{$analytics.total_revenue}{$currency_suffix}</p>
            </div>
            <div class="analytics-card">
                <h4>Unique Customers</h4>
                <p class="analytics-value">{$analytics.unique_clients}</p>
            </div>
        </div>
    </div>
</div>

<!-- Add/Edit Coupon Modal -->
<div id="coupon-modal" class="modal" style="display:none;">
    <div class="modal-content">
        <h3>{if $edit_id}Edit{else}Create{/if} Coupon</h3>
        <form id="coupon-form">
            <input type="hidden" name="id" value="{$edit_id}">
            
            <div class="form-group">
                <label>Coupon Code *</label>
                <input type="text" name="code" required maxlength="20">
                <button type="button" class="btn btn-small" onclick="generateCode()">Generate</button>
            </div>

            <div class="form-group">
                <label>Name *</label>
                <input type="text" name="name" required>
            </div>

            <div class="form-row">
                <div class="form-group">
                    <label>Type *</label>
                    <select name="type" required>
                        <option value="percentage">Percentage</option>
                        <option value="fixed">Fixed Amount</option>
                        <option value="free_setup">Free Setup</option>
                        <option value="free_month">Free Month</option>
                    </select>
                </div>
                <div class="form-group">
                    <label>Value *</label>
                    <input type="number" name="value" step="0.01" required>
                </div>
            </div>

            <div class="form-row">
                <div class="form-group">
                    <label>Max Uses</label>
                    <input type="number" name="max_uses" min="0">
                </div>
                <div class="form-group">
                    <label>Max Uses Per Client</label>
                    <input type="number" name="max_uses_client" min="1" value="1">
                </div>
            </div>

            <div class="form-group">
                <label>Minimum Order Value</label>
                <input type="number" name="min_order_value" step="0.01" min="0">
            </div>

            <div class="form-row">
                <div class="form-group">
                    <label>Start Date</label>
                    <input type="date" name="start_date">
                </div>
                <div class="form-group">
                    <label>Expiry Date</label>
                    <input type="date" name="expiry_date">
                </div>
            </div>

            <div class="form-group">
                <label>Client Group</label>
                <select name="client_group_id">
                    <option value="">All Groups</option>
                    {foreach from=$clientGroups item=cg}
                    <option value="{$cg.id}">{$cg.groupname}</option>
                    {/foreach}
                </select>
            </div>

            <div class="form-group">
                <label>Description</label>
                <textarea name="description" rows="3"></textarea>
            </div>

            <div class="form-actions">
                <button type="submit" class="btn btn-primary">Save Coupon</button>
                <button type="button" class="btn" onclick="closeModal()">Cancel</button>
            </div>
        </form>
    </div>
</div>
{/block}
```

---

## Database Schema

```sql
-- Coupon codes
CREATE TABLE `mod_coupons` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `code` VARCHAR(50) NOT NULL UNIQUE,
    `name` VARCHAR(255) NOT NULL,
    `type` ENUM('percentage', 'fixed', 'free_setup', 'free_month') NOT NULL,
    `value` DECIMAL(10,2) NOT NULL,
    `max_uses` INT DEFAULT 0,
    `uses` INT DEFAULT 0,
    `max_uses_client` INT DEFAULT 1,
    `min_order_value` DECIMAL(10,2) DEFAULT 0,
    `applies_to` JSON DEFAULT NULL,
    `excluded_products` JSON DEFAULT NULL,
    `client_group_id` INT DEFAULT 0,
    `start_date` DATE DEFAULT NULL,
    `expiry_date` DATE DEFAULT NULL,
    `is_recurring` TINYINT(1) DEFAULT 0,
    `recurring_months` INT DEFAULT 0,
    `priority` INT DEFAULT 50,
    `description` TEXT DEFAULT NULL,
    `status` ENUM('active', 'expired', 'disabled', 'depleted') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
    INDEX `idx_code` (`code`),
    INDEX `idx_status` (`status`),
    INDEX `idx_expiry` (`expiry_date`)
);

-- Coupon usage history
CREATE TABLE `mod_coupon_uses` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `coupon_id` INT DEFAULT NULL,
    `coupon_code` VARCHAR(50) NOT NULL,
    `client_id` INT NOT NULL,
    `order_id` INT DEFAULT NULL,
    `discount_amount` DECIMAL(10,2) DEFAULT NULL,
    `used_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_coupon` (`coupon_id`),
    INDEX `idx_client` (`client_id`),
    INDEX `idx_order` (`order_id`),
    INDEX `idx_used_at` (`used_at`)
);

-- Coupon-service linking for recurring
CREATE TABLE `mod_coupon_services` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `coupon_code` VARCHAR(50) NOT NULL,
    `service_id` INT NOT NULL,
    `is_recurring` TINYINT(1) DEFAULT 0,
    `next_apply` DATE DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY `idx_service` (`service_id`),
    FOREIGN KEY (`service_id`) REFERENCES `tblhosting`(`id`) ON DELETE CASCADE
);

-- Coupon attribution tracking
CREATE TABLE `mod_coupon_attribution` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `coupon_code` VARCHAR(50) NOT NULL,
    `client_id` INT DEFAULT NULL,
    `order_id` INT DEFAULT NULL,
    `channel` VARCHAR(50) DEFAULT 'direct',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## Hook Integrations

| Hook Name | Priority | Purpose |
|-----------|----------|---------|
| `ShoppingCartValidateCoupon` | 1 | Validate coupon code |
| `ShoppingCartValidateCoupon` | 2 | Check excluded products |
| `ShoppingCartDiscountOverride` | 1 | Apply discount to cart total |
| `OrderCreated` | 1 | Record coupon usage on order |
| `OrderCreated` | 2 | Track coupon attribution |
| `ServiceRenew` | 1 | Apply recurring coupon discount |
| `InvoiceCreationPreCheck` | 1 | Log coupon on invoice |

---

## Development Checklist

- [ ] Create module directory structure
- [ ] Set up database tables
- [ ] Implement coupon validation
- [ ] Create coupon CRUD operations
- [ ] Add discount calculation logic
- [ ] Implement usage tracking
- [ ] Create admin interface
- [ ] Add bulk coupon creation
- [ ] Implement coupon analytics
- [ ] Add client group restrictions
- [ ] Create minimum order validation
- [ ] Implement product restrictions
- [ ] Add recurring coupon support
- [ ] Create coupon cloning
- [ ] Add coupon attribution tracking
- [ ] Test various coupon types
- [ ] Verify usage limits
- [ ] Test expiry handling
- [ ] Add coupon search/filter
- [ ] Implement coupon reports