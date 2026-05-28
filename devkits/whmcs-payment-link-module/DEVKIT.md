# WHMCS Payment Link Module DevKit

## Header

**Purpose:** Payment link generator module that creates shareable payment links, supports multiple payment methods, tracks payment link analytics, and enables custom payment page branding.

**Module Type:** Payment/Gateway Module

**Use Case:** Hosting companies needing custom payment links for invoices, supporting multiple payment options via single link, custom branding for payment pages, and tracking payment link performance.

---

## Complete Code Template

### File Structure
```
whmcs-payment-link-module/
├── README.md
├── DEVKIT.md
├── payment_link.php         # Main payment link logic
├── hooks.php                # WHMCS hook integrations
└── templates/
    └── admin_payment_link.tpl
```

### Main Module File: payment_link.php

```php
<?php
/**
 * WHMCS Payment Link Module
 * 
 * Provides custom payment link generation and tracking.
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Module Configuration
define('PAYMENT_LINK_MODULE_VERSION', '1.0.0');

// Payment Link Types
define('PLINK_INVOICE', 'invoice');
define('PLINK_AMOUNT', 'amount');
define('PLINK_SUBSCRIPTION', 'subscription');
define('PLINK_DONATION', 'donation');

// Payment Link Status
define('PLINK_STATUS_ACTIVE', 'active');
define('PLINK_STATUS_USED', 'used');
define('PLINK_STATUS_EXPIRED', 'expired');
define('PLINK_STATUS_DISABLED', 'disabled');

/**
 * Create payment link
 */
function payment_link_create($data) {
    $linkId = payment_link_generate_id();
    $token = payment_link_generate_token();
    
    $fields = [
        'link_id', 'token', 'type', 'client_id', 'invoice_id',
        'amount', 'min_amount', 'max_amount', 'currency',
        'title', 'description', 'allowed_methods', 'expiry_date',
        'max_uses', 'use_count', 'branding_id', 'status',
        'created_by', 'created_at', 'updated_at'
    ];
    
    $values = [
        $linkId, $token, $data['type'], $data['client_id'] ?? null,
        $data['invoice_id'] ?? null, $data['amount'] ?? null,
        $data['min_amount'] ?? 0, $data['max_amount'] ?? null,
        $data['currency'] ?? 'USD', $data['title'] ?? 'Payment',
        $data['description'] ?? '', json_encode($data['allowed_methods'] ?? []),
        $data['expiry_date'] ?? null, $data['max_uses'] ?? 1, 0,
        $data['branding_id'] ?? null, PLINK_STATUS_ACTIVE,
        $data['created_by'] ?? $_SESSION['adminid'] ?? 0,
        date('Y-m-d H:i:s'), date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_payment_links', array_combine($fields, $values));
    
    $dbId = mysql_insert_id();
    
    payment_link_log($dbId, 'created', ['type' => $data['type']]);
    
    return [
        'success' => true,
        'link_id' => $linkId,
        'token' => $token,
        'url' => payment_link_get_url($token)
    ];
}

/**
 * Generate unique link ID
 */
function payment_link_generate_id() {
    return 'PL-' . strtoupper(substr(md5(uniqid()), 0, 6)) . '-' . date('ymd');
}

/**
 * Generate secure token
 */
function payment_link_generate_token() {
    return bin2hex(random_bytes(16));
}

/**
 * Get payment link URL
 */
function payment_link_get_url($token) {
    $baseUrl = $GLOBALS['CONFIG']['SystemURL'];
    return $baseUrl . '/pay.php?token=' . $token;
}

/**
 * Get payment link by token
 */
function payment_link_get_by_token($token) {
    $query = "SELECT pl.*, c.firstname, c.lastname, c.email,
              i.invoicenum, i.total as invoice_total
              FROM mod_payment_links pl
              LEFT JOIN tblclients c ON pl.client_id = c.id
              LEFT JOIN tblinvoices i ON pl.invoice_id = i.id
              WHERE pl.token = ?";
    $result = full_query($query, [$token]);
    
    $link = mysql_fetch_assoc($result);
    if ($link) {
        $link['allowed_methods'] = json_decode($link['allowed_methods'], true) ?: [];
    }
    
    return $link;
}

/**
 * Get payment link
 */
function payment_link_get($linkId) {
    $query = "SELECT * FROM mod_payment_links WHERE id = ? OR link_id = ?";
    $result = full_query($query, [$linkId, $linkId]);
    return mysql_fetch_assoc($result);
}

/**
 * Validate payment link
 */
function payment_link_validate($token) {
    $link = payment_link_get_by_token($token);
    
    if (!$link) {
        return ['valid' => false, 'error' => 'Link not found'];
    }
    
    // Check status
    if ($link['status'] != PLINK_STATUS_ACTIVE) {
        $statusMessages = [
            PLINK_STATUS_USED => 'This link has already been used',
            PLINK_STATUS_EXPIRED => 'This link has expired',
            PLINK_STATUS_DISABLED => 'This link has been disabled'
        ];
        return ['valid' => false, 'error' => $statusMessages[$link['status']] ?? 'Link not available'];
    }
    
    // Check expiry
    if ($link['expiry_date'] && $link['expiry_date'] < date('Y-m-d')) {
        payment_link_expire($link['id']);
        return ['valid' => false, 'error' => 'This link has expired'];
    }
    
    // Check uses
    if ($link['max_uses'] > 0 && $link['use_count'] >= $link['max_uses']) {
        return ['valid' => false, 'error' => 'This link has reached its maximum uses'];
    }
    
    return ['valid' => true, 'link' => $link];
}

/**
 * Expire payment link
 */
function payment_link_expire($linkId) {
    update_query('mod_payment_links', [
        'status' => PLINK_STATUS_EXPIRED,
        'updated_at' => date('Y-m-d H:i:s')
    ], ['id' => $linkId]);
}

/**
 * Process payment via link
 */
function payment_link_process($token, $paymentData) {
    $validation = payment_link_validate($token);
    
    if (!$validation['valid']) {
        return $validation;
    }
    
    $link = $validation['link'];
    
    // Validate amount
    $amount = $paymentData['amount'];
    
    if ($link['min_amount'] > 0 && $amount < $link['min_amount']) {
        return ['valid' => false, 'error' => 'Minimum amount is ' . formatCurrency($link['min_amount'])];
    }
    
    if ($link['max_amount'] > 0 && $amount > $link['max_amount']) {
        return ['valid' => false, 'error' => 'Maximum amount is ' . formatCurrency($link['max_amount'])];
    }
    
    // Process payment
    $result = payment_link_execute_payment($link, $paymentData);
    
    if ($result['success']) {
        // Update use count
        payment_link_record_use($link['id'], $result);
        
        // Update status if max uses reached
        $link = payment_link_get($link['id']);
        if ($link['use_count'] >= $link['max_uses']) {
            update_query('mod_payment_links', [
                'status' => PLINK_STATUS_USED
            ], ['id' => $link['id']]);
        }
    }
    
    return $result;
}

/**
 * Execute payment
 */
function payment_link_execute_payment($link, $paymentData) {
    $method = $paymentData['method'];
    $amount = $paymentData['amount'];
    
    // Create transaction record
    $txnId = payment_link_generate_txn_id();
    
    $invoiceId = null;
    if ($link['invoice_id']) {
        $invoiceId = $link['invoice_id'];
    } else {
        // Create invoice for custom amount
        $invoiceId = payment_link_create_invoice($link, $amount);
    }
    
    // Process based on payment method
    switch ($method) {
        case 'card':
            return payment_link_process_card($link, $invoiceId, $amount, $paymentData, $txnId);
        case 'bank':
            return payment_link_process_bank($link, $invoiceId, $amount, $paymentData, $txnId);
        case 'paypal':
            return payment_link_process_paypal($link, $invoiceId, $amount, $txnId);
        case 'stripe':
            return payment_link_process_stripe($link, $invoiceId, $amount, $paymentData, $txnId);
        default:
            return ['success' => false, 'error' => 'Invalid payment method'];
    }
}

/**
 * Generate transaction ID
 */
function payment_link_generate_txn_id() {
    return 'PLTXN-' . date('ymd') . '-' . strtoupper(substr(md2hex(random_bytes(6)), 0, 8));
}

/**
 * Create invoice for custom amount payment
 */
function payment_link_create_invoice($link, $amount) {
    $clientId = $link['client_id'] ?? 0;
    
    $invoiceData = [
        'userid' => $clientId,
        'invoicenum' => payment_link_generate_invoice_num(),
        'date' => date('Y-m-d'),
        'duedate' => date('Y-m-d'),
        'status' => 'Unpaid',
        'paymentmethod' => 'payment_link',
        'notes' => 'Payment via link: ' . $link['link_id']
    ];
    
    $invoiceId = insert_query('tblinvoices', $invoiceData);
    
    insert_query('tblinvoiceitems', [
        'invoiceid' => $invoiceId,
        'userid' => $clientId,
        'description' => $link['title'],
        'amount' => $amount,
        'taxed' => 0
    ]);
    
    return $invoiceId;
}

/**
 * Generate invoice number for payment link
 */
function payment_link_generate_invoice_num() {
    return 'PL-' . date('ymd') . '-' . str_pad(mt_rand(1, 9999), 4, '0', STR_PAD_LEFT);
}

/**
 * Process card payment
 */
function payment_link_process_card($link, $invoiceId, $amount, $paymentData, $txnId) {
    // Card payment processing
    // Integration with payment gateway
    
    // Record transaction
    $transData = [
        'link_id' => $link['id'],
        'invoice_id' => $invoiceId,
        'transaction_id' => $txnId,
        'amount' => $amount,
        'currency' => $link['currency'],
        'method' => 'card',
        'status' => 'completed',
        'card_last_four' => $paymentData['card_last_four'] ?? null,
        'created_at' => date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_payment_link_transactions', $transData);
    
    // Record payment
    insert_query('tblaccounts', [
        'userid' => $link['client_id'],
        'description' => 'Payment via link ' . $link['link_id'],
        'amount' => $amount,
        'date' => date('Y-m-d'),
        'invoiceid' => $invoiceId,
        'transid' => $txnId
    ]);
    
    // Update invoice
    payment_link_mark_invoice_paid($invoiceId, $txnId);
    
    return [
        'success' => true,
        'transaction_id' => $txnId,
        'amount' => $amount
    ];
}

/**
 * Process bank payment
 */
function payment_link_process_bank($link, $invoiceId, $amount, $paymentData, $txnId) {
    // Bank transfer processing
    
    $transData = [
        'link_id' => $link['id'],
        'invoice_id' => $invoiceId,
        'transaction_id' => $txnId,
        'amount' => $amount,
        'currency' => $link['currency'],
        'method' => 'bank',
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_payment_link_transactions', $transData);
    
    return [
        'success' => true,
        'transaction_id' => $txnId,
        'status' => 'pending',
        'instructions' => payment_link_get_bank_instructions($txnId)
    ];
}

/**
 * Get bank payment instructions
 */
function payment_link_get_bank_instructions($txnId) {
    return [
        'reference' => $txnId,
        'bank_name' => 'Your Bank',
        'account_number' => 'XXXX',
        'routing_number' => 'XXXX'
    ];
}

/**
 * Process PayPal payment
 */
function payment_link_process_paypal($link, $invoiceId, $amount, $txnId) {
    // PayPal processing
    
    $transData = [
        'link_id' => $link['id'],
        'invoice_id' => $invoiceId,
        'transaction_id' => $txnId,
        'amount' => $amount,
        'currency' => $link['currency'],
        'method' => 'paypal',
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_payment_link_transactions', $transData);
    
    return [
        'success' => true,
        'transaction_id' => $txnId,
        'redirect_url' => payment_link_get_paypal_url($txnId, $amount)
    ];
}

/**
 * Get PayPal redirect URL
 */
function payment_link_get_paypal_url($txnId, $amount) {
    // Generate PayPal payment URL
    return 'https://www.paypal.com/cgi-bin/webscr?cmd=_xclick&tx=' . $txnId . '&amount=' . $amount;
}

/**
 * Process Stripe payment
 */
function payment_link_process_stripe($link, $invoiceId, $amount, $paymentData, $txnId) {
    // Stripe processing
    
    $transData = [
        'link_id' => $link['id'],
        'invoice_id' => $invoiceId,
        'transaction_id' => $txnId,
        'amount' => $amount,
        'currency' => $link['currency'],
        'method' => 'stripe',
        'status' => 'completed',
        'stripe_payment_id' => $paymentData['stripe_payment_id'] ?? null,
        'created_at' => date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_payment_link_transactions', $transData);
    
    // Record payment
    insert_query('tblaccounts', [
        'userid' => $link['client_id'],
        'description' => 'Payment via Stripe link',
        'amount' => $amount,
        'date' => date('Y-m-d'),
        'invoiceid' => $invoiceId,
        'transid' => $txnId
    ]);
    
    payment_link_mark_invoice_paid($invoiceId, $txnId);
    
    return [
        'success' => true,
        'transaction_id' => $txnId,
        'amount' => $amount
    ];
}

/**
 * Mark invoice as paid
 */
function payment_link_mark_invoice_paid($invoiceId, $txnId) {
    $query = "SELECT SUM(amount) as total_paid FROM tblaccounts WHERE invoiceid = ?";
    $result = full_query($query, [$invoiceId]);
    $data = mysql_fetch_assoc($result);
    
    $invoice = get_query_vals('tblinvoices', '*', ['id' => $invoiceId]);
    
    if ($data['total_paid'] >= $invoice['total']) {
        update_query('tblinvoices', [
            'status' => 'Paid',
            'datepaid' => date('Y-m-d H:i:s')
        ], ['id' => $invoiceId]);
    }
}

/**
 * Record payment link use
 */
function payment_link_record_use($linkId, $result) {
    update_query('mod_payment_links', [
        'use_count' => db_query("SELECT use_count + 1 FROM mod_payment_links WHERE id = ?"),
        'last_used_at' => date('Y-m-d H:i:s'),
        'updated_at' => date('Y-m-d H:i:s')
    ], ['id' => $linkId]);
    
    payment_link_log($linkId, 'used', [
        'transaction_id' => $result['transaction_id'],
        'amount' => $result['amount']
    ]);
}

/**
 * Create link for invoice
 */
function payment_link_create_for_invoice($invoiceId, $options = []) {
    $invoice = get_query_vals('tblinvoices', '*', ['id' => $invoiceId]);
    
    if (!$invoice) {
        return ['success' => false, 'error' => 'Invoice not found'];
    }
    
    $data = [
        'type' => PLINK_INVOICE,
        'client_id' => $invoice['userid'],
        'invoice_id' => $invoiceId,
        'amount' => $invoice['total'] - ($invoice['amount_paid'] ?? 0),
        'title' => 'Invoice #' . ($invoice['invoicenum'] ?: $invoiceId),
        'description' => 'Payment for invoice',
        'expiry_date' => $options['expiry_date'] ?? null,
        'max_uses' => $options['max_uses'] ?? 5,
        'allowed_methods' => $options['allowed_methods'] ?? [],
        'branding_id' => $options['branding_id'] ?? null
    ];
    
    return payment_link_create($data);
}

/**
 * Create link for amount (custom payment)
 */
function payment_link_create_for_amount($clientId, $amount, $title, $options = []) {
    return payment_link_create([
        'type' => PLINK_AMOUNT,
        'client_id' => $clientId,
        'amount' => $amount,
        'title' => $title,
        'description' => $options['description'] ?? '',
        'min_amount' => $options['min_amount'] ?? 0,
        'max_amount' => $options['max_amount'] ?? $amount,
        'expiry_date' => $options['expiry_date'] ?? null,
        'allowed_methods' => $options['allowed_methods'] ?? []
    ]);
}

/**
 * Disable payment link
 */
function payment_link_disable($linkId) {
    update_query('mod_payment_links', [
        'status' => PLINK_STATUS_DISABLED,
        'updated_at' => date('Y-m-d H:i:s')
    ], ['id' => $linkId]);
    
    payment_link_log($linkId, 'disabled');
    
    return ['success' => true];
}

/**
 * Regenerate payment link URL
 */
function payment_link_regenerate($linkId) {
    $link = payment_link_get($linkId);
    
    if (!$link) {
        return ['success' => false, 'error' => 'Link not found'];
    }
    
    $newToken = payment_link_generate_token();
    
    update_query('mod_payment_links', [
        'token' => $newToken,
        'use_count' => 0,
        'status' => PLINK_STATUS_ACTIVE,
        'updated_at' => date('Y-m-d H:i:s')
    ], ['id' => $linkId]);
    
    return [
        'success' => true,
        'token' => $newToken,
        'url' => payment_link_get_url($newToken)
    ];
}

/**
 * Log payment link action
 */
function payment_link_log($linkId, $action, $data = []) {
    if (is_numeric($linkId)) {
        $link = payment_link_get($linkId);
        $linkId = $link ? $link['link_id'] : 0;
    }
    
    insert_query('mod_payment_link_logs', [
        'link_id' => $linkId,
        'action' => $action,
        'data' => json_encode($data),
        'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
        'created_at' => date('Y-m-d H:i:s')
    ]);
}

/**
 * Get payment link analytics
 */
function payment_link_get_analytics($startDate = null, $endDate = null) {
    $where = "1=1";
    $params = [];
    
    if ($startDate && $endDate) {
        $where .= " AND pl.created_at BETWEEN ? AND ?";
        $params = [$startDate, $endDate];
    }
    
    $query = "SELECT 
                COUNT(*) as total_links,
                SUM(pl.use_count) as total_uses,
                SUM(plt.amount) as total_collected,
                COUNT(DISTINCT plt.transaction_id) as total_transactions,
                pl.type,
                plt.method
              FROM mod_payment_links pl
              LEFT JOIN mod_payment_link_transactions plt ON pl.id = plt.link_id
              WHERE {$where}
              GROUP BY pl.type, plt.method";
    
    $result = full_query($query, $params);
    
    $analytics = [];
    while ($row = mysql_fetch_assoc($result)) {
        $analytics[] = $row;
    }
    
    return $analytics;
}

/**
 * Get link statistics
 */
function payment_link_get_stats($linkId) {
    $link = payment_link_get($linkId);
    
    $query = "SELECT 
                COUNT(*) as transactions,
                SUM(amount) as total_collected,
                AVG(amount) as avg_amount,
                MIN(amount) as min_amount,
                MAX(amount) as max_amount,
                method
              FROM mod_payment_link_transactions
              WHERE link_id = ?
              GROUP BY method";
    
    $result = full_query($query, [$linkId]);
    
    $stats = [
        'link' => $link,
        'transactions' => []
    ];
    
    while ($row = mysql_fetch_assoc($result)) {
        $stats['transactions'][] = $row;
    }
    
    return $stats;
}

/**
 * Get client payment links
 */
function payment_link_get_client($clientId) {
    $query = "SELECT pl.*, COUNT(plt.id) as transaction_count
              FROM mod_payment_links pl
              LEFT JOIN mod_payment_link_transactions plt ON pl.id = plt.link_id
              WHERE pl.client_id = ?
              GROUP BY pl.id
              ORDER BY pl.created_at DESC";
    $result = full_query($query, [$clientId]);
    
    $links = [];
    while ($row = mysql_fetch_assoc($result)) {
        $links[] = $row;
    }
    
    return $links;
}

/**
 * Create branded payment page
 */
function payment_link_create_branding($data) {
    $brandId = 'BRAND-' . substr(md5(uniqid()), 0, 8);
    
    $fields = ['brand_id', 'name', 'logo_url', 'primary_color', 'secondary_color',
               'header_text', 'footer_text', 'custom_css', 'status', 'created_at'];
    
    $values = [
        $brandId, $data['name'], $data['logo_url'] ?? '', $data['primary_color'] ?? '#007bff',
        $data['secondary_color'] ?? '#6c757d', $data['header_text'] ?? '',
        $data['footer_text'] ?? '', $data['custom_css'] ?? '',
        'active', date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_payment_link_branding', array_combine($fields, $values));
    
    return ['success' => true, 'brand_id' => $brandId];
}

/**
 * Get branding
 */
function payment_link_get_branding($brandId) {
    $query = "SELECT * FROM mod_payment_link_branding WHERE brand_id = ? OR id = ?";
    $result = full_query($query, [$brandId, $brandId]);
    return mysql_fetch_assoc($result);
}

/**
 * Validate payment link configuration
 */
function payment_link_validate_config($data, &$errors = []) {
    if (empty($data['title'])) {
        $errors[] = 'Title is required';
    }
    
    if (isset($data['amount']) && $data['amount'] <= 0) {
        $errors[] = 'Amount must be positive';
    }
    
    if (isset($data['min_amount']) && isset($data['max_amount'])) {
        if ($data['max_amount'] > 0 && $data['min_amount'] > $data['max_amount']) {
            $errors[] = 'Minimum amount cannot exceed maximum amount';
        }
    }
    
    if (!empty($data['expiry_date']) && $data['expiry_date'] < date('Y-m-d')) {
        $errors[] = 'Expiry date must be in the future';
    }
    
    return count($errors) === 0;
}
```

### Hooks File: hooks.php

```php
<?php
/**
 * WHMCS Payment Link Module Hooks
 */

// Hook: Generate payment link on invoice creation
add_hook('InvoiceCreationPreCheck', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    // Auto-generate payment link for invoices over certain amount
    $invoice = get_query_vals('tblinvoices', '*', ['id' => $invoiceId]);
    
    if ($invoice['total'] >= get_payment_link_min_amount()) {
        $result = payment_link_create_for_invoice($invoiceId, [
            'max_uses' => 3,
            'expiry_date' => date('Y-m-d', strtotime('+30 days'))
        ]);
        
        return [
            'payment_link' => $result['url'],
            'link_id' => $result['link_id']
        ];
    }
});

// Hook: Add payment link to invoice email
add_hook('InvoiceEmailPreSend', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    // Check for payment link
    $query = "SELECT token FROM mod_payment_links 
              WHERE invoice_id = ? AND status = 'active' ORDER BY created_at DESC LIMIT 1";
    $result = full_query($query, [$invoiceId]);
    
    if ($link = mysql_fetch_assoc($result)) {
        $url = payment_link_get_url($link['token']);
        
        return [
            'payment_link_url' => $url,
            'payment_link_text' => 'Or pay quickly with this link: ' . $url
        ];
    }
});

// Hook: Track payment link views
add_hook('PaymentLinkViewed', 1, function($params) {
    $token = $params['token'];
    $link = payment_link_get_by_token($token);
    
    if ($link) {
        // Increment view count
        full_query("UPDATE mod_payment_links SET view_count = view_count + 1 WHERE id = ?", [$link['id']]);
        
        payment_link_log($link['id'], 'viewed', [
            'ip' => $_SERVER['REMOTE_ADDR'],
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? ''
        ]);
    }
});

// Hook: Process PayPal IPN for payment links
add_hook('PayPalIPNReceived', 1, function($params) {
    $txnId = $params['txn_id'];
    
    // Check if this is a payment link transaction
    $query = "SELECT * FROM mod_payment_link_transactions WHERE transaction_id = ?";
    $result = full_query($query, [$txnId]);
    
    if ($trans = mysql_fetch_assoc($result)) {
        // Update transaction status
        update_query('mod_payment_link_transactions', [
            'status' => 'completed'
        ], ['id' => $trans['id']]);
        
        // Mark invoice as paid
        payment_link_mark_invoice_paid($trans['invoice_id'], $txnId);
    }
});

// Hook: Add payment link to client area
add_hook('ClientAreaPageInvoice', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    // Check for existing payment link
    $query = "SELECT * FROM mod_payment_links 
              WHERE invoice_id = ? AND status = 'active'
              ORDER BY created_at DESC LIMIT 1";
    $result = full_query($query, [$invoiceId]);
    
    if ($link = mysql_fetch_assoc($result)) {
        return [
            'payment_link_url' => payment_link_get_url($link['token']),
            'payment_link_expires' => $link['expiry_date']
        ];
    }
});

// Hook: Generate custom payment links on demand
add_hook('AdminAreaPageHook', 1, function($params) {
    if ($params['filename'] == 'invoices' && $_POST['action'] == 'generate_payment_link') {
        $invoiceId = $_POST['invoice_id'];
        
        $result = payment_link_create_for_invoice($invoiceId, [
            'max_uses' => $_POST['max_uses'] ?? 5,
            'expiry_date' => $_POST['expiry_date'] ?? null
        ]);
        
        return $result;
    }
});

// Hook: Track payment link conversions
add_hook('PaymentLinkCompleted', 1, function($params) {
    $token = $params['token'];
    $txnId = $params['transaction_id'];
    $amount = $params['amount'];
    
    $link = payment_link_get_by_token($token);
    
    if ($link) {
        // Log conversion
        payment_link_log($link['id'], 'converted', [
            'transaction_id' => $txnId,
            'amount' => $amount,
            'conversion_value' => $amount
        ]);
        
        // Track for analytics
        payment_link_track_conversion($link['id'], $amount);
    }
});

// Hook: Add payment link options to admin invoice view
add_hook('AdminAreaViewInvoice', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    $query = "SELECT pl.*, COUNT(plt.id) as uses 
              FROM mod_payment_links pl
              LEFT JOIN mod_payment_link_transactions plt ON pl.id = plt.link_id
              WHERE pl.invoice_id = ?
              GROUP BY pl.id";
    $result = full_query($query, [$invoiceId]);
    
    $links = [];
    while ($link = mysql_fetch_assoc($result)) {
        $link['url'] = payment_link_get_url($link['token']);
        $links[] = $link;
    }
    
    return ['payment_links' => $links];
});

// Hook: Process expired links daily
add_hook('DailyCronJob', 1, function($params) {
    // Expire old links
    $query = "UPDATE mod_payment_links 
              SET status = 'expired' 
              WHERE status = 'active' 
              AND expiry_date IS NOT NULL 
              AND expiry_date < CURDATE()";
    full_query($query);
});
```

---

## Database Schema

```sql
-- Payment links
CREATE TABLE `mod_payment_links` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `link_id` VARCHAR(50) NOT NULL UNIQUE,
    `token` VARCHAR(64) NOT NULL UNIQUE,
    `type` ENUM('invoice', 'amount', 'subscription', 'donation') DEFAULT 'invoice',
    `client_id` INT DEFAULT NULL,
    `invoice_id` INT DEFAULT NULL,
    `amount` DECIMAL(10,2) DEFAULT NULL,
    `min_amount` DECIMAL(10,2) DEFAULT 0,
    `max_amount` DECIMAL(10,2) DEFAULT NULL,
    `currency` VARCHAR(3) DEFAULT 'USD',
    `title` VARCHAR(255) NOT NULL,
    `description` TEXT DEFAULT NULL,
    `allowed_methods` JSON DEFAULT NULL,
    `expiry_date` DATE DEFAULT NULL,
    `max_uses` INT DEFAULT 1,
    `use_count` INT DEFAULT 0,
    `view_count` INT DEFAULT 0,
    `branding_id` VARCHAR(50) DEFAULT NULL,
    `status` ENUM('active', 'used', 'expired', 'disabled') DEFAULT 'active',
    `last_used_at` DATETIME DEFAULT NULL,
    `created_by` INT DEFAULT 0,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
    INDEX `idx_token` (`token`),
    INDEX `idx_client` (`client_id`),
    INDEX `idx_invoice` (`invoice_id`),
    INDEX `idx_status` (`status`),
    INDEX `idx_expiry` (`expiry_date`)
);

-- Payment link transactions
CREATE TABLE `mod_payment_link_transactions` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `link_id` INT NOT NULL,
    `invoice_id` INT DEFAULT NULL,
    `transaction_id` VARCHAR(50) NOT NULL UNIQUE,
    `amount` DECIMAL(10,2) NOT NULL,
    `currency` VARCHAR(3) DEFAULT 'USD',
    `method` VARCHAR(20) NOT NULL,
    `status` ENUM('pending', 'completed', 'failed', 'refunded') DEFAULT 'pending',
    `card_last_four` VARCHAR(4) DEFAULT NULL,
    `stripe_payment_id` VARCHAR(255) DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_link` (`link_id`),
    INDEX `idx_invoice` (`invoice_id`),
    INDEX `idx_transaction` (`transaction_id`),
    FOREIGN KEY (`link_id`) REFERENCES `mod_payment_links`(`id`) ON DELETE CASCADE
);

-- Payment link branding
CREATE TABLE `mod_payment_link_branding` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `brand_id` VARCHAR(50) NOT NULL UNIQUE,
    `name` VARCHAR(255) NOT NULL,
    `logo_url` VARCHAR(500) DEFAULT NULL,
    `primary_color` VARCHAR(7) DEFAULT '#007bff',
    `secondary_color` VARCHAR(7) DEFAULT '#6c757d',
    `header_text` TEXT DEFAULT NULL,
    `footer_text` TEXT DEFAULT NULL,
    `custom_css` TEXT DEFAULT NULL,
    `status` ENUM('active', 'inactive') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_brand` (`brand_id`)
);

-- Payment link logs
CREATE TABLE `mod_payment_link_logs` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `link_id` VARCHAR(50) NOT NULL,
    `action` VARCHAR(50) NOT NULL,
    `data` JSON DEFAULT NULL,
    `ip_address` VARCHAR(45) DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_link` (`link_id`),
    INDEX `idx_action` (`action`),
    INDEX `idx_created` (`created_at`)
);
```

---

## Hook Integrations

| Hook Name | Priority | Purpose |
|-----------|----------|---------|
| `InvoiceCreationPreCheck` | 1 | Auto-generate payment links |
| `InvoiceEmailPreSend` | 1 | Add link to invoice email |
| `PaymentLinkViewed` | 1 | Track link views |
| `PayPalIPNReceived` | 1 | Process PayPal payments |
| `ClientAreaPageInvoice` | 1 | Show link in client area |
| `AdminAreaPageHook` | 1 | Generate links from admin |
| `PaymentLinkCompleted` | 1 | Track conversions |
| `AdminAreaViewInvoice` | 1 | Show links in admin |
| `DailyCronJob` | 1 | Expire old links |

---

## Development Checklist

- [ ] Create module directory structure
- [ ] Set up database tables
- [ ] Implement link generation
- [ ] Create secure token system
- [ ] Implement validation
- [ ] Add payment processing
- [ ] Support multiple methods
- [ ] Create invoice linking
- [ ] Build custom amount links
- [ ] Add branding system
- [ ] Implement expiry handling
- [ ] Create analytics
- [ ] Build admin interface
- [ ] Add client area integration
- [ ] Implement view tracking
- [ ] Add conversion tracking
- [ ] Create URL generation
- [ ] Implement use limits
- [ ] Add email integration
- [ ] Test payment flows