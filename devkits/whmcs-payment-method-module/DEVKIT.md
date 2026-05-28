# WHMCS Payment Method Module DevKit

## Header

**Purpose:** Payment method management module that handles custom payment methods, payment method configuration, and secure token storage for recurring payments.

**Module Type:** Payment/Gateway Module

**Use Case:** Hosting companies needing custom payment method management beyond WHMCS defaults, secure storage of payment tokens, and multi-method payment handling.

---

## Complete Code Template

### File Structure
```
whmcs-payment-method-module/
├── README.md
├── DEVKIT.md
├── payment_method.php      # Main payment method logic
├── hooks.php              # WHMCS hook integrations
└── templates/
    └── admin_payment_method.tpl
```

### Main Module File: payment_method.php

```php
<?php
/**
 * WHMCS Payment Method Module
 * 
 * Provides payment method management and token storage.
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Module Configuration
define('PAYMENT_METHOD_MODULE_VERSION', '1.0.0');

// Payment Method Types
define('PM_TYPE_CARD', 'card');
define('PM_TYPE_BANK', 'bank');
define('PM_TYPE_WALLET', 'wallet');
define('PM_TYPE_CRYPTO', 'crypto');
define('PM_TYPE_CUSTOM', 'custom');

// Payment Method Status
define('PM_STATUS_ACTIVE', 'active');
define('PM_STATUS_INACTIVE', 'inactive');
define('PM_STATUS_EXPIRED', 'expired');
define('PM_STATUS_PENDING', 'pending');

/**
 * Add payment method for client
 */
function payment_method_add($clientId, $data) {
    // Validate payment method data
    $validation = payment_method_validate($data);
    if (!$validation['valid']) {
        return ['success' => false, 'errors' => $validation['errors']];
    }
    
    // Encrypt sensitive data
    $encryptedData = payment_method_encrypt($data);
    
    $methodId = payment_method_generate_id();
    
    $fields = [
        'method_id', 'client_id', 'type', 'name', 'details',
        'token', 'is_default', 'is_primary', 'status',
        'expires_at', 'created_at', 'updated_at'
    ];
    
    $values = [
        $methodId, $clientId, $data['type'], $data['name'],
        json_encode($encryptedData['details']), $encryptedData['token'],
        $data['is_default'] ?? 0, $data['is_primary'] ?? 0,
        PM_STATUS_ACTIVE, $data['expires_at'] ?? null,
        date('Y-m-d H:i:s'), date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_payment_methods', array_combine($fields, $values));
    
    // If this is default, update other methods
    if (!empty($data['is_default'])) {
        payment_method_clear_defaults($clientId);
        update_query('mod_payment_methods', ['is_default' => 1], ['id' => mysql_insert_id()]);
    }
    
    // If this is primary, update other methods
    if (!empty($data['is_primary'])) {
        payment_method_clear_primary($clientId);
        update_query('mod_payment_methods', ['is_primary' => 1], ['id' => mysql_insert_id()]);
    }
    
    // Log the action
    payment_method_log($clientId, 'added', $methodId, ['type' => $data['type']]);
    
    return [
        'success' => true,
        'method_id' => $methodId
    ];
}

/**
 * Generate unique method ID
 */
function payment_method_generate_id() {
    return 'PM-' . strtoupper(substr(md5(uniqid()), 0, 8)) . '-' . date('ymd');
}

/**
 * Validate payment method data
 */
function payment_method_validate($data, &$errors = []) {
    if (empty($data['type'])) {
        $errors[] = 'Payment method type is required';
    }
    
    if (empty($data['name'])) {
        $errors[] = 'Payment method name is required';
    }
    
    if ($data['type'] == PM_TYPE_CARD) {
        if (empty($data['last_four']) || strlen($data['last_four']) != 4) {
            $errors[] = 'Last 4 digits of card are required';
        }
        
        if (empty($data['expiry_month']) || empty($data['expiry_year'])) {
            $errors[] = 'Card expiry date is required';
        }
        
        // Check if card is expired
        if (payment_method_is_card_expired($data['expiry_month'], $data['expiry_year'])) {
            $errors[] = 'Card is expired';
        }
    }
    
    if ($data['type'] == PM_TYPE_BANK) {
        if (empty($data['account_last_four'])) {
            $errors[] = 'Last 4 digits of account are required';
        }
    }
    
    return ['valid' => count($errors) === 0, 'errors' => $errors];
}

/**
 * Check if card is expired
 */
function payment_method_is_card_expired($month, $year) {
    $now = new DateTime();
    $expiry = new DateTime($year . '-' . str_pad($month, 2, '0', STR_PAD_LEFT) . '-01');
    $expiry->modify('last day of this month');
    
    return $expiry < $now;
}

/**
 * Encrypt payment method data
 */
function payment_method_encrypt($data) {
    $key = payment_method_get_encryption_key();
    
    // Prepare details for encryption
    $details = [];
    if ($data['type'] == PM_TYPE_CARD) {
        $details = [
            'last_four' => $data['last_four'],
            'expiry_month' => $data['expiry_month'],
            'expiry_year' => $data['expiry_year'],
            'card_type' => $data['card_type'] ?? 'unknown',
            'cardholder_name' => $data['cardholder_name'] ?? ''
        ];
    } elseif ($data['type'] == PM_TYPE_BANK) {
        $details = [
            'account_last_four' => $data['account_last_four'],
            'account_type' => $data['account_type'] ?? 'checking',
            'bank_name' => $data['bank_name'] ?? '',
            'routing_number' => '****' // Only show last 4
        ];
    }
    
    // Encrypt token if provided
    $token = '';
    if (!empty($data['token'])) {
        $token = payment_method_aes_encrypt($data['token'], $key);
    }
    
    return [
        'details' => $details,
        'token' => $token
    ];
}

/**
 * Get encryption key
 */
function payment_method_get_encryption_key() {
    // In production, this should come from secure config
    $query = "SELECT value FROM mod_payment_method_config WHERE `key` = 'encryption_key'";
    $result = full_query($query);
    $data = mysql_fetch_assoc($result);
    
    if (!$data) {
        // Generate and store new key
        $key = bin2hex(random_bytes(32));
        insert_query('mod_payment_method_config', ['key' => 'encryption_key', 'value' => $key]);
        return $key;
    }
    
    return $data['value'];
}

/**
 * AES encrypt
 */
function payment_method_aes_encrypt($data, $key) {
    // Using OpenSSL for encryption
    $iv = random_bytes(16);
    $encrypted = openssl_encrypt($data, 'AES-256-CBC', $key, 0, $iv);
    
    return base64_encode($iv . $encrypted);
}

/**
 * AES decrypt
 */
function payment_method_aes_decrypt($data, $key) {
    $decoded = base64_decode($data);
    $iv = substr($decoded, 0, 16);
    $encrypted = substr($decoded, 16);
    
    return openssl_decrypt($encrypted, 'AES-256-CBC', $key, 0, $iv);
}

/**
 * Update payment method
 */
function payment_method_update($methodId, $data) {
    $method = payment_method_get($methodId);
    
    if (!$method) {
        return ['success' => false, 'error' => 'Payment method not found'];
    }
    
    $updateData = [];
    
    if (!empty($data['name'])) {
        $updateData['name'] = $data['name'];
    }
    
    if (isset($data['is_default'])) {
        payment_method_clear_defaults($method['client_id']);
        $updateData['is_default'] = 1;
    }
    
    if (isset($data['is_primary'])) {
        payment_method_clear_primary($method['client_id']);
        $updateData['is_primary'] = 1;
    }
    
    if (isset($data['status'])) {
        $updateData['status'] = $data['status'];
    }
    
    $updateData['updated_at'] = date('Y-m-d H:i:s');
    
    update_query('mod_payment_methods', $updateData, ['method_id' => $methodId]);
    
    payment_method_log($method['client_id'], 'updated', $methodId, $data);
    
    return ['success' => true];
}

/**
 * Clear default payment methods
 */
function payment_method_clear_defaults($clientId) {
    update_query('mod_payment_methods', ['is_default' => 0], ['client_id' => $clientId]);
}

/**
 * Clear primary payment methods
 */
function payment_method_clear_primary($clientId) {
    update_query('mod_payment_methods', ['is_primary' => 0], ['client_id' => $clientId]);
}

/**
 * Delete payment method
 */
function payment_method_delete($methodId) {
    $method = payment_method_get($methodId);
    
    if (!$method) {
        return ['success' => false, 'error' => 'Payment method not found'];
    }
    
    // Soft delete by marking inactive
    update_query('mod_payment_methods', [
        'status' => PM_STATUS_INACTIVE,
        'deleted_at' => date('Y-m-d H:i:s')
    ], ['method_id' => $methodId]);
    
    payment_method_log($method['client_id'], 'deleted', $methodId);
    
    return ['success' => true];
}

/**
 * Get payment method
 */
function payment_method_get($methodId) {
    $query = "SELECT * FROM mod_payment_methods WHERE method_id = ? OR id = ?";
    $result = full_query($query, [$methodId, $methodId]);
    
    $method = mysql_fetch_assoc($result);
    if ($method) {
        $method['details'] = json_decode($method['details'], true);
    }
    
    return $method;
}

/**
 * Get client's payment methods
 */
function payment_method_get_client($clientId, $activeOnly = true) {
    $where = "client_id = ?";
    $params = [$clientId];
    
    if ($activeOnly) {
        $where .= " AND status = 'active'";
    }
    
    $query = "SELECT * FROM mod_payment_methods WHERE {$where} ORDER BY is_default DESC, is_primary DESC, created_at DESC";
    $result = full_query($query, $params);
    
    $methods = [];
    while ($row = mysql_fetch_assoc($result)) {
        $row['details'] = json_decode($row['details'], true);
        
        // Decrypt token
        if (!empty($row['token'])) {
            $key = payment_method_get_encryption_key();
            $row['token_decrypted'] = payment_method_aes_decrypt($row['token'], $key);
        }
        
        $methods[] = $row;
    }
    
    return $methods;
}

/**
 * Get default payment method for client
 */
function payment_method_get_default($clientId) {
    $query = "SELECT * FROM mod_payment_methods 
              WHERE client_id = ? AND is_default = 1 AND status = 'active'
              LIMIT 1";
    $result = full_query($query, [$clientId]);
    
    $method = mysql_fetch_assoc($result);
    if ($method) {
        $method['details'] = json_decode($method['details'], true);
    }
    
    return $method;
}

/**
 * Get primary payment method for client
 */
function payment_method_get_primary($clientId) {
    $query = "SELECT * FROM mod_payment_methods 
              WHERE client_id = ? AND is_primary = 1 AND status = 'active'
              LIMIT 1";
    $result = full_query($query, [$clientId]);
    
    $method = mysql_fetch_assoc($result);
    if ($method) {
        $method['details'] = json_decode($method['details'], true);
    }
    
    return $method;
}

/**
 * Set payment method as default
 */
function payment_method_set_default($methodId) {
    $method = payment_method_get($methodId);
    
    if (!$method) {
        return ['success' => false, 'error' => 'Payment method not found'];
    }
    
    payment_method_clear_defaults($method['client_id']);
    
    update_query('mod_payment_methods', [
        'is_default' => 1,
        'updated_at' => date('Y-m-d H:i:s')
    ], ['method_id' => $methodId]);
    
    return ['success' => true];
}

/**
 * Set payment method as primary
 */
function payment_method_set_primary($methodId) {
    $method = payment_method_get($methodId);
    
    if (!$method) {
        return ['success' => false, 'error' => 'Payment method not found'];
    }
    
    payment_method_clear_primary($method['client_id']);
    
    update_query('mod_payment_methods', [
        'is_primary' => 1,
        'updated_at' => date('Y-m-d H:i:s')
    ], ['method_id' => $methodId]);
    
    return ['success' => true];
}

/**
 * Process payment with stored method
 */
function payment_method_process($methodId, $amount, $invoiceId) {
    $method = payment_method_get($methodId);
    
    if (!$method || $method['status'] != PM_STATUS_ACTIVE) {
        return ['success' => false, 'error' => 'Payment method not available'];
    }
    
    // Decrypt token
    $key = payment_method_get_encryption_key();
    $token = payment_method_aes_decrypt($method['token'], $key);
    
    // Process based on type
    switch ($method['type']) {
        case PM_TYPE_CARD:
            return payment_method_process_card($method, $token, $amount, $invoiceId);
        case PM_TYPE_BANK:
            return payment_method_process_bank($method, $token, $amount, $invoiceId);
        case PM_TYPE_WALLET:
            return payment_method_process_wallet($method, $token, $amount, $invoiceId);
        default:
            return ['success' => false, 'error' => 'Unsupported payment method type'];
    }
}

/**
 * Process card payment
 */
function payment_method_process_card($method, $token, $amount, $invoiceId) {
    // Integration with payment gateway
    // This would call the actual payment processor
    
    $result = [
        'success' => true,
        'transaction_id' => 'txn_' . time(),
        'amount' => $amount
    ];
    
    // Record payment
    if ($result['success']) {
        insert_query('tblaccounts', [
            'userid' => $method['client_id'],
            'description' => 'Payment via stored card',
            'amount' => $amount,
            'date' => date('Y-m-d'),
            'invoiceid' => $invoiceId,
            'transid' => $result['transaction_id'],
            'paymentmethod' => 'stored_card'
        ]);
        
        payment_method_log($method['client_id'], 'payment_processed', $method['method_id'], [
            'amount' => $amount,
            'invoice_id' => $invoiceId,
            'transaction_id' => $result['transaction_id']
        ]);
    }
    
    return $result;
}

/**
 * Process bank payment
 */
function payment_method_process_bank($method, $token, $amount, $invoiceId) {
    // ACH/bank transfer processing
    
    return [
        'success' => true,
        'transaction_id' => 'ach_' . time(),
        'amount' => $amount
    ];
}

/**
 * Process wallet payment
 */
function payment_method_process_wallet($method, $token, $amount, $invoiceId) {
    // e.g., PayPal, Stripe, etc.
    
    return [
        'success' => true,
        'transaction_id' => 'wallet_' . time(),
        'amount' => $amount
    ];
}

/**
 * Check and update expired payment methods
 */
function payment_method_check_expirations() {
    $query = "SELECT * FROM mod_payment_methods 
              WHERE status = 'active' AND expires_at IS NOT NULL
              AND expires_at <= CURDATE() + INTERVAL 30 DAY";
    $result = full_query($query);
    
    $expiring = [];
    $expired = [];
    
    while ($method = mysql_fetch_assoc($result)) {
        if ($method['expires_at'] <= date('Y-m-d')) {
            // Mark as expired
            update_query('mod_payment_methods', [
                'status' => PM_STATUS_EXPIRED
            ], ['id' => $method['id']]);
            
            $expired[] = $method['method_id'];
            
            // Notify client
            payment_method_notify_expiration($method['client_id'], $method, true);
        } else {
            $expiring[] = $method['method_id'];
            
            // Notify client of upcoming expiration
            if (date('d') == '01') { // Only on 1st of month
                payment_method_notify_expiration($method['client_id'], $method, false);
            }
        }
    }
    
    return [
        'expiring' => count($expiring),
        'expired' => count($expired)
    ];
}

/**
 * Notify client of expiration
 */
function payment_method_notify_expiration($clientId, $method, $isExpired) {
    $query = "SELECT email, firstname, lastname FROM tblclients WHERE id = ?";
    $result = full_query($query, [$clientId]);
    $client = mysql_fetch_assoc($result);
    
    if ($client) {
        $type = $method['type'] == PM_TYPE_CARD ? 'card' : 'payment method';
        
        if ($isExpired) {
            $subject = 'Your ' . $type . ' has expired';
            $message = 'Your stored ' . $type . ' (' . $method['name'] . ') has expired. Please update your payment information.';
        } else {
            $subject = 'Your ' . $type . ' will expire soon';
            $message = 'Your stored ' . $type . ' (' . $method['name'] . ') will expire on ' . $method['expires_at'] . '. Please update your payment information.';
        }
        
        send_email($client['email'], $subject, $message);
    }
}

/**
 * Log payment method action
 */
function payment_method_log($clientId, $action, $methodId, $data = []) {
    insert_query('mod_payment_method_logs', [
        'client_id' => $clientId,
        'method_id' => $methodId,
        'action' => $action,
        'data' => json_encode($data),
        'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
        'created_at' => date('Y-m-d H:i:s')
    ]);
}

/**
 * Get payment method logs
 */
function payment_method_get_logs($methodId, $limit = 50) {
    $query = "SELECT * FROM mod_payment_method_logs 
              WHERE method_id = ? ORDER BY created_at DESC LIMIT ?";
    $result = full_query($query, [$methodId, $limit]);
    
    $logs = [];
    while ($row = mysql_fetch_assoc($result)) {
        $row['data'] = json_decode($row['data'], true);
        $logs[] = $row;
    }
    
    return $logs;
}

/**
 * Get payment method statistics
 */
function payment_method_get_stats($clientId = null) {
    $where = "1=1";
    $params = [];
    
    if ($clientId) {
        $where .= " AND client_id = ?";
        $params[] = $clientId;
    }
    
    $query = "SELECT 
                type,
                COUNT(*) as total,
                SUM(is_default) as defaults,
                SUM(CASE WHEN status = 'active' THEN 1 ELSE 0 END) as active,
                SUM(CASE WHEN status = 'expired' THEN 1 ELSE 0 END) as expired
              FROM mod_payment_methods
              WHERE {$where}
              GROUP BY type";
    
    $result = full_query($query, $params);
    
    $stats = [];
    while ($row = mysql_fetch_assoc($result)) {
        $stats[] = $row;
    }
    
    return $stats;
}

/**
 * Import payment method from gateway token
 */
function payment_method_import($clientId, $gateway, $token, $data) {
    $data['token'] = $token;
    $data['type'] = payment_method_get_type_from_gateway($gateway);
    
    return payment_method_add($clientId, $data);
}

/**
 * Get payment method type from gateway name
 */
function payment_method_get_type_from_gateway($gateway) {
    $mapping = [
        'stripe' => PM_TYPE_CARD,
        'paypal' => PM_TYPE_WALLET,
        'authorizenet' => PM_TYPE_CARD,
        'braintree' => PM_TYPE_CARD
    ];
    
    return $mapping[$gateway] ?? PM_TYPE_CUSTOM;
}

/**
 * Create custom payment method
 */
function payment_method_create_custom($clientId, $name, $type, $details = []) {
    return payment_method_add($clientId, [
        'name' => $name,
        'type' => $type,
        'details' => $details,
        'is_default' => false
    ]);
}

/**
 * Validate payment method for use
 */
function payment_method_can_use($methodId) {
    $method = payment_method_get($methodId);
    
    if (!$method) {
        return ['can_use' => false, 'reason' => 'Method not found'];
    }
    
    if ($method['status'] != PM_STATUS_ACTIVE) {
        return ['can_use' => false, 'reason' => 'Method is ' . $method['status']];
    }
    
    if ($method['expires_at'] && $method['expires_at'] <= date('Y-m-d')) {
        return ['can_use' => false, 'reason' => 'Method has expired'];
    }
    
    return ['can_use' => true];
}
```

### Hooks File: hooks.php

```php
<?php
/**
 * WHMCS Payment Method Module Hooks
 */

// Hook: Show stored payment methods at checkout
add_hook('ShoppingCartCheckoutOverride', 1, function($params) {
    if ($_SESSION['uid']) {
        $methods = payment_method_get_client($_SESSION['uid']);
        
        return [
            'stored_payment_methods' => $methods,
            'has_stored_methods' => !empty($methods)
        ];
    }
});

// Hook: Use stored payment method
add_hook('CheckoutComplete', 1, function($params) {
    $selectedMethod = $_POST['payment_method_id'] ?? null;
    
    if ($selectedMethod && $_SESSION['uid']) {
        $canUse = payment_method_can_use($selectedMethod);
        
        if ($canUse['can_use']) {
            // Process with stored method
            $invoiceId = $params['invoiceId'] ?? 0;
            $amount = $params['total'] ?? 0;
            
            return payment_method_process($selectedMethod, $amount, $invoiceId);
        }
    }
});

// Hook: Check for expiring payment methods
add_hook('DailyCronJob', 1, function($params) {
    $result = payment_method_check_expirations();
    
    return [
        'expiring_count' => $result['expiring'],
        'expired_count' => $result['expired']
    ];
});

// Hook: Add payment method options to client profile
add_hook('ClientProfileTabFields', 1, function($params) {
    $clientId = $params['clientId'];
    $methods = payment_method_get_client($clientId);
    
    return [
        'payment_methods' => $methods,
        'method_count' => count($methods)
    ];
});

// Hook: Handle automatic retry with stored method
add_hook('InvoicePaymentFailed', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    $invoice = get_query_vals('tblinvoices', 'userid', ['id' => $invoiceId]);
    
    // Try to charge default payment method
    $default = payment_method_get_default($invoice['userid']);
    
    if ($default) {
        $canUse = payment_method_can_use($default['method_id']);
        
        if ($canUse['can_use']) {
            return payment_method_process($default['method_id'], 
                                            $params['amount'], $invoiceId);
        }
    }
});

// Hook: Update payment method on successful transaction
add_hook('InvoicePaid', 1, function($params) {
    $paymentMethod = $_POST['payment_method_id'] ?? null;
    
    if ($paymentMethod && $params['clientId']) {
        // Update last used timestamp
        update_query('mod_payment_methods', [
            'last_used_at' => date('Y-m-d H:i:s'),
            'updated_at' => date('Y-m-d H:i:s')
        ], ['method_id' => $paymentMethod]);
    }
});

// Hook: Prevent service changes if no valid payment method
add_hook('BeforeServiceRenewal', 1, function($params) {
    $clientId = $params['clientId'];
    
    $methods = payment_method_get_client($clientId, true);
    
    if (empty($methods)) {
        return [
            'warning' => true,
            'message' => 'Please add a payment method to ensure continuous service'
        ];
    }
});

// Hook: Store payment method after transaction
add_hook('AfterPaymentProcessed', 1, function($params) {
    if (!empty($_POST['save_payment_method']) && $_SESSION['uid']) {
        $data = [
            'name' => $_POST['payment_method_name'] ?? 'Saved Payment Method',
            'type' => $_POST['payment_type'] ?? PM_TYPE_CARD,
            'token' => $_POST['payment_token'] ?? '',
            'last_four' => $_POST['last_four'] ?? '',
            'expiry_month' => $_POST['expiry_month'] ?? '',
            'expiry_year' => $_POST['expiry_year'] ?? '',
            'is_default' => !empty($_POST['set_as_default'])
        ];
        
        payment_method_add($_SESSION['uid'], $data);
    }
});

// Hook: Delete payment method on client deletion
add_hook('ClientDeleted', 1, function($params) {
    $clientId = $params['clientId'];
    
    // Delete all payment methods (soft delete)
    update_query('mod_payment_methods', [
        'status' => PM_STATUS_INACTIVE,
        'deleted_at' => date('Y-m-d H:i:s')
    ], ['client_id' => $clientId]);
});
```

---

## Database Schema

```sql
-- Payment methods
CREATE TABLE `mod_payment_methods` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `method_id` VARCHAR(50) NOT NULL UNIQUE,
    `client_id` INT NOT NULL,
    `type` ENUM('card', 'bank', 'wallet', 'crypto', 'custom') NOT NULL,
    `name` VARCHAR(255) NOT NULL,
    `details` TEXT DEFAULT NULL,
    `token` TEXT DEFAULT NULL,
    `is_default` TINYINT(1) DEFAULT 0,
    `is_primary` TINYINT(1) DEFAULT 0,
    `status` ENUM('active', 'inactive', 'expired', 'pending') DEFAULT 'active',
    `expires_at` DATE DEFAULT NULL,
    `last_used_at` DATETIME DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
    `deleted_at` DATETIME DEFAULT NULL,
    INDEX `idx_client` (`client_id`),
    INDEX `idx_type` (`type`),
    INDEX `idx_status` (`status`),
    INDEX `idx_default` (`is_default`),
    FOREIGN KEY (`client_id`) REFERENCES `tblclients`(`id`) ON DELETE CASCADE
);

-- Payment method logs
CREATE TABLE `mod_payment_method_logs` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `client_id` INT NOT NULL,
    `method_id` VARCHAR(50) NOT NULL,
    `action` VARCHAR(50) NOT NULL,
    `data` JSON DEFAULT NULL,
    `ip_address` VARCHAR(45) DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_client` (`client_id`),
    INDEX `idx_method` (`method_id`),
    INDEX `idx_created` (`created_at`)
);

-- Configuration
CREATE TABLE `mod_payment_method_config` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `key` VARCHAR(100) NOT NULL UNIQUE,
    `value` TEXT DEFAULT NULL,
    `updated_at` DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP
);
```

---

## Hook Integrations

| Hook Name | Priority | Purpose |
|-----------|----------|---------|
| `ShoppingCartCheckoutOverride` | 1 | Show stored methods |
| `CheckoutComplete` | 1 | Process with stored method |
| `DailyCronJob` | 1 | Check expirations |
| `ClientProfileTabFields` | 1 | Add to profile |
| `InvoicePaymentFailed` | 1 | Auto-retry payment |
| `InvoicePaid` | 1 | Update last used |
| `BeforeServiceRenewal` | 1 | Warning if no methods |
| `AfterPaymentProcessed` | 1 | Save new method |
| `ClientDeleted` | 1 | Clean up methods |

---

## Development Checklist

- [ ] Create module directory structure
- [ ] Set up database tables
- [ ] Implement encryption
- [ ] Create method storage
- [ ] Add CRUD operations
- [ ] Implement default/primary logic
- [ ] Build token management
- [ ] Create expiration handling
- [ ] Add payment processing
- [ ] Build admin interface
- [ ] Add client profile integration
- [ ] Implement checkout integration
- [ ] Create notifications
- [ ] Add logging
- [ ] Test encryption
- [ ] Verify token storage
- [ ] Test payment processing
- [ ] Add import/export
- [ ] Create reports
- [ ] Add validation