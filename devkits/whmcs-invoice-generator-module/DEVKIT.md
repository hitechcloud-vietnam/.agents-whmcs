# WHMCS Invoice Generator Module DevKit

## Header

**Purpose:** Custom invoice generator that creates specialized invoices with custom layouts, branding, multiple languages, and various invoice formats (PDF, HTML, email).

**Module Type:** Billing/Document Module

**Use Case:** Hosting companies needing branded invoices, multi-language support, custom invoice numbering schemes, or specialized invoice formats for different client types.

---

## Complete Code Template

### File Structure
```
whmcs-invoice-generator-module/
├── README.md
├── DEVKIT.md
├── invoice_generator.php   # Main generator logic
├── hooks.php              # WHMCS hook integrations
├── templates/
│   ├── invoice_classic.tpl
│   ├── invoice_modern.tpl
│   └── invoice_minimal.tpl
└── pdf/
    └── tcpdf_invoice.php
```

### Main Module File: invoice_generator.php

```php
<?php
/**
 * WHMCS Invoice Generator Module
 * 
 * Provides custom invoice generation with various templates and formats.
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Module Configuration
define('INVOICE_GENERATOR_VERSION', '1.0.0');

// Invoice Formats
define('FORMAT_PDF', 'pdf');
define('FORMAT_HTML', 'html');
define('FORMAT_EMAIL', 'email');

// Invoice Templates
define('TEMPLATE_CLASSIC', 'classic');
define('TEMPLATE_MODERN', 'modern');
define('TEMPLATE_MINIMAL', 'minimal');
define('TEMPLATE_BRANDED', 'branded');

// Invoice Status
define('INV_STATUS_DRAFT', 'draft');
define('INV_STATUS_FINALIZED', 'finalized');
define('INV_STATUS_SENT', 'sent');
define('INV_STATUS_PAID', 'paid');
define('INV_STATUS_VOID', 'void');

/**
 * Generate invoice document
 */
function invoice_generate($invoiceId, $format = FORMAT_PDF, $template = TEMPLATE_CLASSIC, $options = []) {
    $invoiceData = invoice_get_data($invoiceId);
    
    if (!$invoiceData) {
        return ['success' => false, 'error' => 'Invoice not found'];
    }
    
    switch ($format) {
        case FORMAT_PDF:
            return invoice_generate_pdf($invoiceData, $template, $options);
        case FORMAT_HTML:
            return invoice_generate_html($invoiceData, $template, $options);
        case FORMAT_EMAIL:
            return invoice_generate_email($invoiceData, $template, $options);
        default:
            return ['success' => false, 'error' => 'Invalid format'];
    }
}

/**
 * Get complete invoice data
 */
function invoice_get_data($invoiceId) {
    $invoice = get_invoice($invoiceId);
    
    if (!$invoice) {
        return null;
    }
    
    // Get client info
    $client = get_client($invoice['userid']);
    
    // Get invoice items
    $items = invoice_get_items($invoiceId);
    
    // Get payments
    $payments = invoice_get_payments($invoiceId);
    
    // Calculate totals
    $totals = invoice_calculate_totals($items, $payments);
    
    // Get custom fields
    $customFields = invoice_get_custom_fields($invoiceId);
    
    return [
        'invoice' => $invoice,
        'client' => $client,
        'items' => $items,
        'payments' => $payments,
        'totals' => $totals,
        'custom_fields' => $customFields,
        'generated_at' => date('Y-m-d H:i:s')
    ];
}

/**
 * Get invoice object
 */
function get_invoice($invoiceId) {
    $query = "SELECT * FROM tblinvoices WHERE id = ?";
    $result = full_query($query, [$invoiceId]);
    return mysql_fetch_assoc($result);
}

/**
 * Get client info
 */
function get_client($clientId) {
    $query = "SELECT c.*, cg.groupname, cg.colour as group_colour
              FROM tblclients c
              LEFT JOIN tblclientgroups cg ON c.groupid = cg.id
              WHERE c.id = ?";
    $result = full_query($query, [$clientId]);
    return mysql_fetch_assoc($result);
}

/**
 * Get invoice line items
 */
function invoice_get_items($invoiceId) {
    $query = "SELECT * FROM tblinvoiceitems WHERE invoiceid = ? ORDER BY id ASC";
    $result = full_query($query, [$invoiceId]);
    
    $items = [];
    while ($row = mysql_fetch_assoc($result)) {
        // Get product details if applicable
        if ($row['type'] == 'Hosting' || $row['type'] == 'Product') {
            $row['product_details'] = invoice_get_product_details($row['relid']);
        }
        $items[] = $row;
    }
    
    return $items;
}

/**
 * Get product details
 */
function invoice_get_product_details($productId) {
    $query = "SELECT p.name, p.description, h.domain, h.regdate
              FROM tblproducts p
              LEFT JOIN tblhosting h ON h.packageid = p.id AND h.relid = ?
              WHERE p.id = ?";
    $result = full_query($query, [$productId, $productId]);
    return mysql_fetch_assoc($result);
}

/**
 * Get payments on invoice
 */
function invoice_get_payments($invoiceId) {
    $query = "SELECT * FROM tblinvoices WHERE id = ?";
    $invoice = full_query($query, [$invoiceId]);
    
    if ($invoice['datepaid'] && $invoice['datepaid'] != '0000-00-00 00:00:00') {
        return [[
            'id' => 0,
            'date' => $invoice['datepaid'],
            'amount' => $invoice['total'],
            'method' => $invoice['paymentmethod']
        ]];
    }
    
    return [];
}

/**
 * Calculate invoice totals
 */
function invoice_calculate_totals($items, $payments) {
    $subtotal = 0;
    $tax = 0;
    $paid = 0;
    
    foreach ($items as $item) {
        $subtotal += $item['amount'];
        if ($item['taxed']) {
            $tax += $item['tax'] ?? 0;
        }
    }
    
    foreach ($payments as $payment) {
        $paid += $payment['amount'];
    }
    
    $total = $subtotal + $tax;
    $balance = $total - $paid;
    
    return [
        'subtotal' => round($subtotal, 2),
        'tax' => round($tax, 2),
        'total' => round($total, 2),
        'paid' => round($paid, 2),
        'balance' => round($balance, 2)
    ];
}

/**
 * Get custom invoice fields
 */
function invoice_get_custom_fields($invoiceId) {
    $query = "SELECT * FROM mod_invoice_custom_fields WHERE invoice_id = ?";
    $result = full_query($query, [$invoiceId]);
    
    $fields = [];
    while ($row = mysql_fetch_assoc($result)) {
        $fields[$row['field_name']] = $row['field_value'];
    }
    
    return $fields;
}

/**
 * Generate PDF invoice
 */
function invoice_generate_pdf($data, $template, $options = []) {
    // Load TCPDF if available
    if (class_exists('TCPDF')) {
        require_once(__DIR__ . '/pdf/tcpdf_invoice.php');
        return tcpdf_generate_invoice($data, $template, $options);
    }
    
    // Fallback to HTML2PDF or similar
    $html = invoice_generate_html($data, $template, $options);
    
    // Convert HTML to PDF
    $pdf = invoice_html_to_pdf($html);
    
    return $pdf;
}

/**
 * Generate HTML invoice
 */
function invoice_generate_html($data, $template, $options = []) {
    $templateFile = __DIR__ . '/templates/invoice_' . $template . '.tpl';
    
    if (!file_exists($templateFile)) {
        $templateFile = __DIR__ . '/templates/invoice_classic.tpl';
    }
    
    // Assign template variables
    $vars = [
        'invoice' => $data['invoice'],
        'client' => $data['client'],
        'items' => $data['items'],
        'payments' => $data['payments'],
        'totals' => $data['totals'],
        'custom_fields' => $data['custom_fields'],
        'company_name' => $options['company_name'] ?? $GLOBALS['CONFIG']['CompanyName'],
        'company_address' => $options['company_address'] ?? '',
        'company_logo' => $options['company_logo'] ?? '',
        'generated_at' => $data['generated_at'],
        'currency_prefix' => $data['invoice']['currency_prefix'] ?? '$',
        'currency_suffix' => $data['invoice']['currency_suffix'] ?? '',
        'tax_id_label' => $options['tax_id_label'] ?? 'Tax ID',
        'tax_id_value' => $options['tax_id_value'] ?? ''
    ];
    
    // Process template
    ob_start();
    extract($vars);
    include($templateFile);
    $html = ob_get_clean();
    
    return $html;
}

/**
 * Convert HTML to PDF
 */
function invoice_html_to_pdf($html) {
    // Implementation depends on PDF library (dompdf, mPDF, etc.)
    // This is a placeholder
    
    require_once(__DIR__ . '/lib/dompdf/autoload.inc.php');
    
    $dompdf = new Dompdf\Dompdf();
    $dompdf->loadHtml($html);
    $dompdf->setPaper('A4', 'portrait');
    $dompdf->render();
    
    return $dompdf->output();
}

/**
 * Generate email version
 */
function invoice_generate_email($data, $template, $options = []) {
    $html = invoice_generate_html($data, $template, $options);
    
    // Convert to plain text
    $text = invoice_html_to_text($html);
    
    return [
        'subject' => 'Invoice ' . $data['invoice']['invoicenum'] . ' from ' . 
                     ($options['company_name'] ?? $GLOBALS['CONFIG']['CompanyName']),
        'html' => $html,
        'text' => $text
    ];
}

/**
 * Convert HTML to plain text
 */
function invoice_html_to_text($html) {
    // Strip HTML tags and format
    $text = strip_tags($html);
    $text = preg_replace('/\s+/', ' ', $text);
    return trim($text);
}

/**
 * Send invoice email
 */
function invoice_send_email($invoiceId, $options = []) {
    $data = invoice_get_data($invoiceId);
    
    if (!$data) {
        return ['success' => false, 'error' => 'Invoice not found'];
    }
    
    $emailData = invoice_generate_email($data, $options['template'] ?? TEMPLATE_CLASSIC, $options);
    
    $email = new PHPMailer();
    $email->isSMTP();
    $email->Host = $GLOBALS['CONFIG']['SMTPHost'];
    $email->SMTPAuth = true;
    $email->Username = $GLOBALS['CONFIG']['SMTPUsername'];
    $email->Password = $GLOBALS['CONFIG']['SMTPPassword'];
    $email->SMTPSecure = $GLOBALS['CONFIG']['SMTPPort'] == 465 ? 'ssl' : 'tls';
    $email->Port = $GLOBALS['CONFIG']['SMTPPort'];
    
    $email->setFrom($options['from_email'] ?? $GLOBALS['CONFIG']['Email'], 
                    $options['from_name'] ?? $GLOBALS['CONFIG']['CompanyName']);
    $email->addAddress($data['client']['email']);
    
    $email->Subject = $emailData['subject'];
    $email->isHTML(true);
    $email->Body = $emailData['html'];
    $email->AltBody = $emailData['text'];
    
    // Attach PDF
    if (!empty($options['attach_pdf'])) {
        $pdf = invoice_generate_pdf($data, $options['template'] ?? TEMPLATE_CLASSIC);
        $email->addStringAttachment($pdf, 'Invoice-' . $data['invoice']['invoicenum'] . '.pdf');
    }
    
    if ($email->send()) {
        // Log email sent
        invoice_log_email($invoiceId, $data['client']['email']);
        
        // Update status
        update_query('tblinvoices', ['status' => 'Sent'], ['id' => $invoiceId]);
        
        return ['success' => true];
    }
    
    return ['success' => false, 'error' => $email->ErrorInfo];
}

/**
 * Log invoice email
 */
function invoice_log_email($invoiceId, $email) {
    $data = [
        'invoice_id' => $invoiceId,
        'email' => $email,
        'sent_at' => date('Y-m-d H:i:s')
    ];
    insert_query('mod_invoice_emails', $data);
}

/**
 * Create custom invoice
 */
function invoice_create_custom($clientId, $data, $items) {
    global $CONFIG;
    
    $dueDate = date('Y-m-d', strtotime('+' . ($data['due_days'] ?? $CONFIG['InvoiceDueDays']) . ' days'));
    
    $invoiceData = [
        'userid' => $clientId,
        'invoicenum' => invoice_generate_number($data['number_prefix'] ?? null),
        'date' => $data['date'] ?? date('Y-m-d'),
        'duedate' => $dueDate,
        'status' => $data['status'] ?? 'Draft',
        'paymentmethod' => $data['payment_method'] ?? 'banktransfer',
        'notes' => $data['notes'] ?? '',
        'tax_id' => $data['tax_id'] ?? ''
    ];
    
    $invoiceId = insert_query('tblinvoices', $invoiceData);
    
    // Add items
    foreach ($items as $item) {
        invoice_add_item($invoiceId, $item);
    }
    
    // Add custom fields
    if (!empty($data['custom_fields'])) {
        foreach ($data['custom_fields'] as $name => $value) {
            invoice_set_custom_field($invoiceId, $name, $value);
        }
    }
    
    return $invoiceId;
}

/**
 * Generate custom invoice number
 */
function invoice_generate_number($prefix = null) {
    global $CONFIG;
    
    $prefix = $prefix ?? ($CONFIG['InvoicePrefix'] ?? 'INV-');
    $year = date('Y');
    
    $query = "SELECT MAX(CAST(SUBSTRING(invoicenum, LENGTH(?) + 1) AS UNSIGNED)) as max_num 
              FROM tblinvoices 
              WHERE invoicenum LIKE ? 
              AND invoicenum LIKE ?";
    
    $result = full_query($query, [$prefix, $prefix . '%', $prefix . $year . '%']);
    $data = mysql_fetch_assoc($result);
    
    $nextNum = ($data['max_num'] ?? 0) + 1;
    
    return $prefix . $year . str_pad($nextNum, 4, '0', STR_PAD_LEFT);
}

/**
 * Add item to invoice
 */
function invoice_add_item($invoiceId, $item) {
    $data = [
        'invoiceid' => $invoiceId,
        'userid' => $item['userid'] ?? 0,
        'type' => $item['type'] ?? 'Item',
        'relid' => $item['relid'] ?? 0,
        'description' => $item['description'],
        'amount' => $item['amount'],
        'taxed' => $item['taxed'] ?? 0,
        'tax' => $item['tax'] ?? 0,
        'taxrate' => $item['taxrate'] ?? 0
    ];
    
    insert_query('tblinvoiceitems', $data);
}

/**
 * Set custom field on invoice
 */
function invoice_set_custom_field($invoiceId, $name, $value) {
    // Check if exists
    $query = "SELECT id FROM mod_invoice_custom_fields 
              WHERE invoice_id = ? AND field_name = ?";
    $result = full_query($query, [$invoiceId, $name]);
    
    if (mysql_fetch_assoc($result)) {
        update_query('mod_invoice_custom_fields', 
                     ['field_value' => $value], 
                     ['invoice_id' => $invoiceId, 'field_name' => $name]);
    } else {
        insert_query('mod_invoice_custom_fields', [
            'invoice_id' => $invoiceId,
            'field_name' => $name,
            'field_value' => $value
        ]);
    }
}

/**
 * Clone invoice
 */
function invoice_clone($invoiceId, $options = []) {
    $data = invoice_get_data($invoiceId);
    
    if (!$data) {
        return null;
    }
    
    $newData = [
        'date' => $options['date'] ?? date('Y-m-d'),
        'due_days' => $options['due_days'] ?? null,
        'number_prefix' => $options['prefix'] ?? null,
        'notes' => $options['notes'] ?? $data['invoice']['notes'],
        'custom_fields' => $data['custom_fields']
    ];
    
    $items = [];
    foreach ($data['items'] as $item) {
        $items[] = [
            'description' => $item['description'],
            'amount' => $item['amount'],
            'taxed' => $item['taxed'],
            'tax' => $item['tax'] ?? 0,
            'taxrate' => $item['taxrate'] ?? 0
        ];
    }
    
    return invoice_create_custom($data['invoice']['userid'], $newData, $items);
}

/**
 * Void invoice
 */
function invoice_void($invoiceId, $reason = '') {
    update_query('tblinvoices', [
        'status' => 'Cancelled',
        'notes' => dbEscapeString($data['invoice']['notes'] . "\nVoided: " . $reason)
    ], ['id' => $invoiceId]);
    
    // Log void action
    invoice_log_action($invoiceId, 'voided', ['reason' => $reason]);
    
    // Reverse any payments
    invoice_reverse_payments($invoiceId);
}

/**
 * Reverse payments on invoice
 */
function invoice_reverse_payments($invoiceId) {
    $query = "SELECT * FROM mod_invoice_payments WHERE invoice_id = ?";
    $result = full_query($query, [$invoiceId]);
    
    while ($payment = mysql_fetch_assoc($result)) {
        // Create reversal transaction
        insert_query('tblaccounts', [
            'userid' => $payment['client_id'],
            'description' => 'Reversal: ' . $payment['description'],
            'amount' => -$payment['amount'],
            'date' => date('Y-m-d'),
            'invoice_id' => $invoiceId
        ]);
    }
}

/**
 * Log invoice action
 */
function invoice_log_action($invoiceId, $action, $data = []) {
    $fields = ['invoice_id', 'action', 'data', 'created_at'];
    $values = [$invoiceId, $action, json_encode($data), date('Y-m-d H:i:s')];
    
    insert_query('mod_invoice_history', array_combine($fields, $values));
}

/**
 * Get invoice history
 */
function invoice_get_history($invoiceId) {
    $query = "SELECT * FROM mod_invoice_history 
              WHERE invoice_id = ? ORDER BY created_at DESC";
    $result = full_query($query, [$invoiceId]);
    
    $history = [];
    while ($row = mysql_fetch_assoc($result)) {
        $row['data'] = json_decode($row['data'], true);
        $history[] = $row;
    }
    
    return $history;
}
```

### Hooks File: hooks.php

```php
<?php
/**
 * WHMCS Invoice Generator Module Hooks
 */

// Hook: Override default invoice generation
add_hook('InvoiceCreationPreCheck', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    // Apply custom invoice template if set
    $clientData = get_query_vals('tblinvoices', 'userid', ['id' => $invoiceId]);
    $clientGroup = get_client_group($clientData['userid']);
    
    if ($clientGroup && !empty($clientGroup['invoice_template'])) {
        return [
            'template' => $clientGroup['invoice_template'],
            'format' => 'pdf'
        ];
    }
});

// Hook: Customize invoice number generation
add_hook('InvoiceNumOverride', 1, function($params) {
    $clientId = $params['clientId'];
    
    // Check for custom numbering rules
    $client = get_client($clientId);
    
    if ($client['groupid']) {
        $group = get_query_vals('tblclientgroups', 'invoice_prefix', ['id' => $client['groupid']]);
        
        if ($group['invoice_prefix']) {
            return [
                'invoicenum' => invoice_generate_number($group['invoice_prefix'])
            ];
        }
    }
});

// Hook: Modify invoice email before sending
add_hook('InvoiceEmailPreSend', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    $data = invoice_get_data($invoiceId);
    
    // Add custom branding
    $clientGroup = $data['client']['groupid'] ?? 0;
    
    $template = 'classic';
    if ($clientGroup) {
        $groupData = get_query_vals('tblclientgroups', '*', ['id' => $clientGroup]);
        if ($groupData['invoice_template']) {
            $template = $groupData['invoice_template'];
        }
    }
    
    $emailData = invoice_generate_email($data, $template);
    
    return [
        'subject' => $emailData['subject'],
        'body' => $emailData['html'],
        'altbody' => $emailData['text']
    ];
});

// Hook: Add custom fields to invoice view
add_hook('AdminAreaViewInvoice', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    $customFields = invoice_get_custom_fields($invoiceId);
    
    return [
        'custom_fields' => $customFields
    ];
});

// Hook: Generate invoice on demand
add_hook('AdminAreaPageHook', 1, function($params) {
    if ($params['filename'] == 'invoicepdf' && !empty($params['id'])) {
        $data = invoice_get_data($params['id']);
        
        // Apply custom template if set
        $template = $_GET['template'] ?? TEMPLATE_CLASSIC;
        
        $pdf = invoice_generate_pdf($data, $template);
        
        // Output PDF
        header('Content-Type: application/pdf');
        echo $pdf;
        exit;
    }
});

// Hook: Generate bulk invoices
add_hook('AdminAreaPageHook', 2, function($params) {
    if ($params['filename'] == 'bulk-invoices') {
        $invoiceIds = $_POST['invoice_ids'] ?? [];
        $format = $_POST['format'] ?? 'pdf';
        $template = $_POST['template'] ?? TEMPLATE_CLASSIC;
        
        $results = [];
        foreach ($invoiceIds as $invoiceId) {
            $result = invoice_generate($invoiceId, $format, $template);
            $results[] = [
                'invoice_id' => $invoiceId,
                'success' => $result['success'] ?? false,
                'data' => $result
            ];
        }
        
        return ['results' => $results];
    }
});

// Hook: Customize invoice PDF footer
add_hook('InvoicePDFRender', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    // Add custom footer text based on invoice type
    $invoice = get_invoice($invoiceId);
    
    $footer = '';
    if (strpos($invoice['notes'], 'RECURRING') !== false) {
        $footer = "This is a recurring invoice generated automatically.";
    } elseif (strpos($invoice['notes'], 'MANUAL') !== false) {
        $footer = "For questions about this invoice, please contact our billing department.";
    }
    
    return ['footer' => $footer];
});
```

---

## Database Schema

```sql
-- Custom invoice fields
CREATE TABLE `mod_invoice_custom_fields` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `invoice_id` INT NOT NULL,
    `field_name` VARCHAR(100) NOT NULL,
    `field_value` TEXT DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY `idx_invoice_field` (`invoice_id`, `field_name`),
    FOREIGN KEY (`invoice_id`) REFERENCES `tblinvoices`(`id`) ON DELETE CASCADE
);

-- Invoice email log
CREATE TABLE `mod_invoice_emails` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `invoice_id` INT NOT NULL,
    `email` VARCHAR(255) NOT NULL,
    `sent_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_invoice` (`invoice_id`)
);

-- Invoice history log
CREATE TABLE `mod_invoice_history` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `invoice_id` INT NOT NULL,
    `action` VARCHAR(50) NOT NULL,
    `data` JSON DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_invoice` (`invoice_id`)
);

-- Invoice payment records
CREATE TABLE `mod_invoice_payments` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `invoice_id` INT NOT NULL,
    `client_id` INT NOT NULL,
    `amount` DECIMAL(10,2) NOT NULL,
    `description` VARCHAR(255) DEFAULT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Invoice templates configuration
CREATE TABLE `mod_invoice_templates` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `name` VARCHAR(100) NOT NULL,
    `template_code` VARCHAR(50) NOT NULL,
    `is_default` TINYINT(1) DEFAULT 0,
    `client_group_id` INT DEFAULT NULL,
    `header_html` TEXT DEFAULT NULL,
    `footer_html` TEXT DEFAULT NULL,
    `css_styles` TEXT DEFAULT NULL,
    `status` ENUM('active', 'inactive') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY `idx_template` (`template_code`)
);
```

---

## Hook Integrations

| Hook Name | Priority | Purpose |
|-----------|----------|---------|
| `InvoiceCreationPreCheck` | 1 | Apply custom invoice template |
| `InvoiceNumOverride` | 1 | Customize invoice numbering |
| `InvoiceEmailPreSend` | 1 | Modify invoice email content |
| `AdminAreaViewInvoice` | 1 | Add custom fields to view |
| `AdminAreaPageHook` | 1 | Generate custom PDF |
| `AdminAreaPageHook` | 2 | Bulk invoice generation |
| `InvoicePDFRender` | 1 | Add custom PDF footer |

---

## Development Checklist

- [ ] Create module directory structure
- [ ] Set up database tables
- [ ] Implement invoice data retrieval
- [ ] Create PDF generation logic
- [ ] Build HTML templates (classic, modern, minimal)
- [ ] Implement email generation
- [ ] Add custom invoice creation
- [ ] Create invoice cloning function
- [ ] Implement invoice voiding
- [ ] Add custom invoice fields
- [ ] Build admin interface
- [ ] Add client group template assignment
- [ ] Implement invoice history logging
- [ ] Create bulk generation feature
- [ ] Add multi-language support
- [ ] Test PDF generation
- [ ] Verify email sending
- [ ] Add invoice template editor
- [ ] Implement preview functionality
- [ ] Add branding options