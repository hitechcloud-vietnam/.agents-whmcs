# WHMCS Tax Module DevKit

## Header

**Purpose:** Advanced tax calculation module that handles complex tax scenarios including multi-jurisdiction tax rates, VAT/GST compliance, tax exemptions, and tax holiday periods.

**Module Type:** Tax/Compliance Module

**Use Case:** Hosting companies operating in multiple jurisdictions, needing custom tax rules, or requiring integration with external tax services like Avalara or TaxJar.

---

## Complete Code Template

### File Structure
```
whmcs-tax-module/
├── README.md
├── DEVKIT.md
├── tax.php            # Main tax calculation logic
├── hooks.php           # WHMCS hook integrations
├── services/
│   └── external_tax_service.php
└── templates/
    └── admin_tax.tpl
```

### Main Module File: tax.php

```php
<?php
/**
 * WHMCS Tax Calculation Module
 * 
 * Provides advanced tax calculation for multiple jurisdictions.
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Module Configuration
define('TAX_MODULE_VERSION', '1.0.0');

// Tax Types
define('TAX_TYPE_VAT', 'vat');
define('TAX_TYPE_GST', 'gst');
define('TAX_TYPE_SALES', 'sales');
define('TAX_TYPE_USAGE', 'usage');

// Tax Calculation Methods
define('TAX_CALC_INCLUSIVE', 'inclusive');
define('TAX_CALC_EXCLUSIVE', 'exclusive');

// Tax Jurisdictions
define('TAX_JURISDICTION_COUNTRY', 'country');
define('TAX_JURISDICTION_STATE', 'state');
define('TAX_JURISDICTION_COUNTY', 'county');
define('TAX_JURISDICTION_CITY', 'city');

// Tax Status
define('TAX_STATUS_ACTIVE', 'active');
define('TAX_STATUS_EXEMPT', 'exempt');
define('TAX_STATUS_SUSPENDED', 'suspended');

/**
 * Calculate tax for an item
 */
function tax_calculate($clientId, $amount, $productId = 0, $options = []) {
    $clientData = tax_get_client_info($clientId);
    
    if (!$clientData) {
        return ['tax_amount' => 0, 'total' => $amount, 'taxes' => []];
    }
    
    // Check for tax exemption
    if (tax_is_exempt($clientId, $productId)) {
        return [
            'tax_amount' => 0,
            'total' => $amount,
            'taxes' => [],
            'exempt' => true,
            'exemption_reason' => tax_get_exemption_reason($clientId)
        ];
    }
    
    // Get applicable tax rules
    $taxRules = tax_get_applicable_rules($clientData, $productId);
    
    // Calculate each tax
    $taxes = [];
    $totalTax = 0;
    
    foreach ($taxRules as $rule) {
        $taxAmount = tax_calculate_single($amount, $rule);
        $taxes[] = [
            'name' => $rule['name'],
            'rate' => $rule['rate'],
            'amount' => $taxAmount,
            'jurisdiction' => $rule['jurisdiction'],
            'jurisdiction_name' => $rule['jurisdiction_name']
        ];
        $totalTax += $taxAmount;
    }
    
    return [
        'tax_amount' => round($totalTax, 2),
        'total' => round($amount + $totalTax, 2),
        'taxes' => $taxes,
        'exempt' => false
    ];
}

/**
 * Get client tax information
 */
function tax_get_client_info($clientId) {
    $query = "SELECT c.*, s.name as state_name, co.name as country_name, 
              co.country_code
              FROM tblclients c
              LEFT JOIN tblstates s ON c.state = s.code AND s.countryid = c.country
              LEFT JOIN tblcountries co ON c.country = co.id
              WHERE c.id = ?";
    $result = full_query($query, [$clientId]);
    return mysql_fetch_assoc($result);
}

/**
 * Check if client/product is tax exempt
 */
function tax_is_exempt($clientId, $productId = 0) {
    // Check client exemption
    $query = "SELECT * FROM mod_tax_exemptions 
              WHERE client_id = ? AND (product_id = ? OR product_id = 0)
              AND (expiry_date IS NULL OR expiry_date >= CURDATE())
              AND status = 'active'
              LIMIT 1";
    $result = full_query($query, [$clientId, $productId]);
    
    if (mysql_fetch_assoc($result)) {
        return true;
    }
    
    // Check client group exemption
    $clientGroupQuery = "SELECT group_id FROM mod_tax_exempt_groups 
                        WHERE client_id = ? AND status = 'active' LIMIT 1";
    $groupResult = full_query($clientGroupQuery, [$clientId]);
    if (mysql_fetch_assoc($groupResult)) {
        return true;
    }
    
    return false;
}

/**
 * Get exemption reason
 */
function tax_get_exemption_reason($clientId) {
    $query = "SELECT reason FROM mod_tax_exemptions 
              WHERE client_id = ? AND status = 'active'
              ORDER BY product_id DESC LIMIT 1";
    $result = full_query($query, [$clientId]);
    $data = mysql_fetch_assoc($result);
    return $data['reason'] ?? 'Tax exempt';
}

/**
 * Get applicable tax rules for client/location
 */
function tax_get_applicable_rules($clientData, $productId = 0) {
    $rules = [];
    
    // Get country-level tax
    $countryRules = tax_get_country_rules($clientData['country'], $productId);
    $rules = array_merge($rules, $countryRules);
    
    // Get state-level tax
    if (!empty($clientData['state'])) {
        $stateRules = tax_get_state_rules(
            $clientData['country'], 
            $clientData['state'], 
            $productId
        );
        $rules = array_merge($rules, $stateRules);
    }
    
    // Get city-level tax
    if (!empty($clientData['city'])) {
        $cityRules = tax_get_city_rules(
            $clientData['country'],
            $clientData['state'],
            $clientData['city'],
            $productId
        );
        $rules = array_merge($rules, $cityRules);
    }
    
    // Sort by priority
    usort($rules, function($a, $b) {
        return $b['priority'] - $a['priority'];
    });
    
    return $rules;
}

/**
 * Get country-level tax rules
 */
function tax_get_country_rules($countryCode, $productId = 0) {
    $query = "SELECT * FROM mod_tax_rules 
              WHERE country_code = ? 
              AND jurisdiction_type = 'country'
              AND (product_id = ? OR product_id = 0)
              AND status = 'active'
              AND (start_date IS NULL OR start_date <= CURDATE())
              AND (end_date IS NULL OR end_date >= CURDATE())
              ORDER BY priority DESC";
    $result = full_query($query, [$countryCode, $productId]);
    
    $rules = [];
    while ($row = mysql_fetch_assoc($result)) {
        $rules[] = array_merge($row, ['jurisdiction_name' => $countryCode]);
    }
    
    return $rules;
}

/**
 * Get state-level tax rules
 */
function tax_get_state_rules($countryCode, $stateCode, $productId = 0) {
    $query = "SELECT * FROM mod_tax_rules 
              WHERE country_code = ? AND state_code = ?
              AND jurisdiction_type IN ('state', 'country')
              AND (product_id = ? OR product_id = 0)
              AND status = 'active'
              ORDER BY 
                CASE jurisdiction_type WHEN 'state' THEN 1 ELSE 2 END,
                priority DESC";
    $result = full_query($query, [$countryCode, $stateCode, $productId]);
    
    $rules = [];
    while ($row = mysql_fetch_assoc($result)) {
        $rules[] = array_merge($row, [
            'jurisdiction_name' => $row['state_name'] ?? $stateCode
        ]);
    }
    
    return $rules;
}

/**
 * Get city-level tax rules
 */
function tax_get_city_rules($countryCode, $stateCode, $city, $productId = 0) {
    $query = "SELECT * FROM mod_tax_rules 
              WHERE country_code = ? AND state_code = ? AND city = ?
              AND jurisdiction_type = 'city'
              AND (product_id = ? OR product_id = 0)
              AND status = 'active'
              ORDER BY priority DESC";
    $result = full_query($query, [$countryCode, $stateCode, $city, $productId]);
    
    $rules = [];
    while ($row = mysql_fetch_assoc($result)) {
        $rules[] = array_merge($row, ['jurisdiction_name' => $city]);
    }
    
    return $rules;
}

/**
 * Calculate single tax amount
 */
function tax_calculate_single($amount, $rule) {
    $rate = $rule['rate'];
    
    // Compound tax calculation
    if ($rule['is_compound']) {
        // Compound taxes are calculated on top of other taxes
        return round($amount * ($rate / 100), 2);
    }
    
    // Simple tax calculation
    return round($amount * ($rate / 100), 2);
}

/**
 * Calculate tax included in amount (reverse tax)
 */
function tax_calculate_reverse($totalAmount, $taxRate) {
    return round($totalAmount / (1 + ($taxRate / 100)) * ($taxRate / 100), 2);
}

/**
 * Get VAT number validation
 */
function tax_validate_vat_number($vatNumber, $countryCode = null) {
    // Remove spaces and country prefix
    $vatNumber = preg_replace('/\s+/', '', $vatNumber);
    
    if (!$countryCode && strlen($vatNumber) > 2) {
        $countryCode = substr($vatNumber, 0, 2);
        $vatNumber = substr($vatNumber, 2);
    }
    
    // Call external validation service or implement VIES check
    $result = tax_check_vies($countryCode, $vatNumber);
    
    return $result;
}

/**
 * Check VAT against EU VIES system
 */
function tax_check_vies($countryCode, $vatNumber) {
    $client = new SoapClient('http://ec.europa.eu/taxation_customs/vies/checkVatService.wsdl');
    
    try {
        $result = $client->checkVat([
            'countryCode' => strtoupper($countryCode),
            'vatNumber' => $vatNumber
        ]);
        
        return [
            'valid' => true,
            'name' => $result->name,
            'address' => $result->address ?? null,
            'company' => $result->company ?? null
        ];
    } catch (Exception $e) {
        return [
            'valid' => false,
            'error' => $e->getMessage()
        ];
    }
}

/**
 * Check for tax holiday periods
 */
function tax_check_holiday($countryCode, $stateCode = null, $date = null) {
    $date = $date ?? date('Y-m-d');
    
    $query = "SELECT * FROM mod_tax_holidays 
              WHERE country_code = ?
              AND (state_code = ? OR state_code IS NULL)
              AND start_date <= ?
              AND end_date >= ?
              AND status = 'active'";
    $result = full_query($query, [$countryCode, $stateCode, $date, $date]);
    
    if ($holiday = mysql_fetch_assoc($result)) {
        return [
            'is_holiday' => true,
            'name' => $holiday['name'],
            'rate_override' => $holiday['rate_override']
        ];
    }
    
    return ['is_holiday' => false];
}

/**
 * Create tax rule
 */
function tax_create_rule($data) {
    $fields = [
        'name', 'country_code', 'state_code', 'city', 'jurisdiction_type',
        'rate', 'tax_type', 'is_compound', 'product_id', 'priority',
        'start_date', 'end_date', 'status', 'created_at'
    ];
    
    $values = [
        $data['name'], $data['country_code'], $data['state_code'] ?? null,
        $data['city'] ?? null, $data['jurisdiction_type'],
        $data['rate'], $data['tax_type'] ?? 'sales',
        $data['is_compound'] ?? 0, $data['product_id'] ?? 0,
        $data['priority'] ?? 50, $data['start_date'] ?? null,
        $data['end_date'] ?? null, TAX_STATUS_ACTIVE, date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_tax_rules', array_combine($fields, $values));
    return mysql_insert_id();
}

/**
 * Update tax rule
 */
function tax_update_rule($ruleId, $data) {
    $allowedFields = ['name', 'rate', 'priority', 'status', 'start_date', 'end_date'];
    $updateData = [];
    
    foreach ($allowedFields as $field) {
        if (isset($data[$field])) {
            $updateData[$field] = $data[$field];
        }
    }
    
    $updateData['updated_at'] = date('Y-m-d H:i:s');
    update_query('mod_tax_rules', $updateData, ['id' => $ruleId]);
}

/**
 * Add tax exemption
 */
function tax_add_exemption($clientId, $productId, $reason, $expiryDate = null) {
    $data = [
        'client_id' => $clientId,
        'product_id' => $productId,
        'reason' => $reason,
        'expiry_date' => $expiryDate,
        'status' => 'active',
        'created_at' => date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_tax_exemptions', $data);
}

/**
 * Create tax report for jurisdiction
 */
function tax_generate_report($countryCode, $startDate, $endDate, $stateCode = null) {
    $query = "SELECT 
                th.invoice_id,
                th.invoice_num,
                th.date,
                th.client_name,
                th.client_vat,
                th.amount,
                th.tax_amount,
                th.tax_rate,
                tr.name as tax_name,
                tr.country_code,
                tr.state_code
              FROM mod_tax_history th
              JOIN mod_tax_rules tr ON th.tax_rule_id = tr.id
              WHERE tr.country_code = ?
              AND (tr.state_code = ? OR ? IS NULL)
              AND th.date BETWEEN ? AND ?
              ORDER BY th.date DESC";
    
    $result = full_query($query, [$countryCode, $stateCode, $stateCode, $startDate, $endDate]);
    
    $report = [];
    while ($row = mysql_fetch_assoc($result)) {
        $report[] = $row;
    }
    
    return $report;
}

/**
 * Calculate tax summary for period
 */
function tax_get_summary($countryCode, $startDate, $endDate) {
    $query = "SELECT 
                SUM(th.tax_amount) as total_tax,
                COUNT(DISTINCT th.client_id) as unique_clients,
                COUNT(*) as total_transactions,
                tr.name as tax_name,
                tr.rate
              FROM mod_tax_history th
              JOIN mod_tax_rules tr ON th.tax_rule_id = tr.id
              WHERE tr.country_code = ?
              AND th.date BETWEEN ? AND ?
              GROUP BY tr.id
              ORDER BY total_tax DESC";
    
    $result = full_query($query, [$countryCode, $startDate, $endDate]);
    
    $summary = [];
    while ($row = mysql_fetch_assoc($result)) {
        $summary[] = $row;
    }
    
    return $summary;
}

/**
 * Log tax calculation for audit
 */
function tax_log_calculation($data) {
    $fields = [
        'invoice_id', 'invoice_num', 'client_id', 'client_name', 'client_vat',
        'amount', 'tax_amount', 'tax_rate', 'tax_rule_id', 'country_code',
        'state_code', 'product_id', 'date', 'created_at'
    ];
    
    $values = [
        $data['invoice_id'], $data['invoice_num'], $data['client_id'],
        $data['client_name'], $data['client_vat'] ?? null,
        $data['amount'], $data['tax_amount'], $data['tax_rate'],
        $data['tax_rule_id'], $data['country_code'], $data['state_code'] ?? null,
        $data['product_id'] ?? 0, $data['date'], date('Y-m-d H:i:s')
    ];
    
    insert_query('mod_tax_history', array_combine($fields, $values));
}

/**
 * Validate tax rule configuration
 */
function tax_validate_rule($data, &$errors) {
    if (empty($data['name'])) {
        $errors[] = "Tax rule name is required";
    }
    
    if (empty($data['country_code'])) {
        $errors[] = "Country code is required";
    }
    
    if (!isset($data['rate']) || $data['rate'] < 0) {
        $errors[] = "Valid tax rate is required";
    }
    
    if ($data['rate'] > 100) {
        $errors[] = "Tax rate cannot exceed 100%";
    }
    
    return count($errors) === 0;
}
```

### Hooks File: hooks.php

```php
<?php
/**
 * WHMCS Tax Module Hooks
 */

// Hook: Calculate tax on invoice item
add_hook('CalculateTax', 1, function($params) {
    $clientId = $params['clientId'];
    $amount = $params['amount'];
    $productId = $params['productId'] ?? 0;
    
    $result = tax_calculate($clientId, $amount, $productId);
    
    return [
        'tax_amount' => $result['tax_amount'],
        'total' => $result['total'],
        'taxes' => $result['taxes'],
        'exempt' => $result['exempt']
    ];
});

// Hook: Apply tax when creating invoice
add_hook('InvoiceCreationPreCheck', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    $clientId = get_query_val('tblinvoices', 'userid', ['id' => $invoiceId]);
    
    // Get invoice items and apply tax
    $items = select_query('tblinvoiceitems', '*', ['invoiceid' => $invoiceId]);
    
    while ($item = mysql_fetch_assoc($items)) {
        $amount = $item['amount'];
        $taxResult = tax_calculate($clientId, $amount, $item['relid']);
        
        // Update item with tax
        if ($taxResult['tax_amount'] > 0 && !$item['taxed']) {
            update_query('tblinvoiceitems', [
                'taxed' => 1
            ], ['id' => $item['id']]);
            
            // Log tax calculation
            tax_log_calculation([
                'invoice_id' => $invoiceId,
                'invoice_num' => get_query_val('tblinvoices', 'invoicenum', ['id' => $invoiceId]),
                'client_id' => $clientId,
                'client_name' => get_client_name($clientId),
                'amount' => $amount,
                'tax_amount' => $taxResult['tax_amount'],
                'tax_rate' => $taxResult['taxes'][0]['rate'] ?? 0,
                'tax_rule_id' => $taxResult['taxes'][0]['id'] ?? 0,
                'country_code' => get_client_country($clientId),
                'product_id' => $item['relid'],
                'date' => date('Y-m-d')
            ]);
        }
    }
});

// Hook: Validate VAT number on client update
add_hook('ClientUpdate', 1, function($params) {
    if (!empty($params['taxId'])) {
        $clientData = tax_get_client_info($params['clientId']);
        $result = tax_validate_vat_number($params['taxId'], $clientData['country']);
        
        if ($result['valid']) {
            // Update client with validated VAT info
            update_query('tblclients', [
                'tax_id' => $params['taxId'],
                'vat_validated' => 1
            ], ['id' => $params['clientId']]);
            
            // Mark client as VAT exempt
            tax_add_exemption($params['clientId'], 0, 'Valid VAT number: ' . $params['taxId']);
        }
    }
});

// Hook: Apply reverse charge VAT for B2B
add_hook('InvoiceCreationPreCheck', 2, function($params) {
    $invoiceId = $params['invoiceid'];
    $clientId = get_query_val('tblinvoices', 'userid', ['id' => $invoiceId]);
    $clientData = tax_get_client_info($clientId);
    
    // Check if client has valid VAT number and is in different EU country
    if (!empty($clientData['tax_id']) && $clientData['country'] != 'US') {
        // Reverse charge VAT applies - no tax charged to customer
        update_query('tblinvoices', [
            'notes' => dbEscapeString("VAT reverse charge applied - Customer VAT: " . $clientData['tax_id'])
        ], ['id' => $invoiceId]);
    }
});

// Hook: Check tax holidays
add_hook('CalculateTax', 2, function($params) {
    $clientData = tax_get_client_info($params['clientId']);
    $holiday = tax_check_holiday(
        $clientData['country_code'],
        $clientData['state'] ?? null
    );
    
    if ($holiday['is_holiday'] && $holiday['rate_override'] !== null) {
        // Return reduced tax rate
        return [
            'tax_amount' => round($params['amount'] * ($holiday['rate_override'] / 100), 2),
            'holiday_name' => $holiday['name'],
            'rate_override' => $holiday['rate_override']
        ];
    }
});

// Hook: Generate tax report on invoice payment
add_hook('InvoicePaid', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    
    // Generate tax liability report entry
    $query = "SELECT SUM(amount) as total, SUM(tax) as total_tax 
              FROM tblinvoiceitems WHERE invoiceid = ?";
    $result = full_query($query, [$invoiceId]);
    $data = mysql_fetch_assoc($result);
    
    // Log for tax reporting
    logTaxLiability($invoiceId, $data['total'], $data['total_tax'], date('Y-m-d'));
});
```

### Admin Template: templates/admin_tax.tpl

```html
{extends file="admin/template.tpl"}

{block name="content"}
<div class="tax-admin">
    <div class="header">
        <h2>Tax Configuration</h2>
        <button class="btn btn-primary" onclick="showAddRuleModal()">Add Tax Rule</button>
    </div>

    <div class="tax-tabs">
        <button class="tab active" data-tab="rules">Tax Rules</button>
        <button class="tab" data-tab="jurisdictions">Jurisdictions</button>
        <button class="tab" data-tab="exemptions">Exemptions</button>
        <button class="tab" data-tab="holidays">Tax Holidays</button>
        <button class="tab" data-tab="reports">Reports</button>
    </div>

    <div id="tab-rules" class="tab-content">
        <table class="tax-table">
            <thead>
                <tr>
                    <th>Name</th>
                    <th>Country</th>
                    <th>State</th>
                    <th>City</th>
                    <th>Rate</th>
                    <th>Type</th>
                    <th>Priority</th>
                    <th>Status</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody>
                {foreach from=$rules item=rule}
                <tr>
                    <td>{$rule.name}</td>
                    <td>{$rule.country_code}</td>
                    <td>{$rule.state_code|default:'-'}</td>
                    <td>{$rule.city|default:'-'}</td>
                    <td>{$rule.rate}%</td>
                    <td>{$rule.tax_type}</td>
                    <td>{$rule.priority}</td>
                    <td>
                        <span class="status-badge status-{$rule.status}">{$rule.status}</span>
                    </td>
                    <td>
                        <button onclick="editRule({$rule.id})">Edit</button>
                        <button onclick="deleteRule({$rule.id})">Delete</button>
                    </td>
                </tr>
                {/foreach}
            </tbody>
        </table>
    </div>

    <div id="tab-exemptions" class="tab-content" style="display:none;">
        <form class="exemption-form">
            <div class="form-row">
                <select name="client_id">
                    <option value="">Select Client</option>
                    {foreach from=$clients item=c}
                    <option value="{$c.id}">{$c.firstname} {$c.lastname}</option>
                    {/foreach}
                </select>
                <input type="text" name="reason" placeholder="Exemption Reason">
                <input type="date" name="expiry_date" placeholder="Expiry Date (Optional)">
                <button type="button" class="btn" onclick="addExemption()">Add Exemption</button>
            </div>
        </form>

        <table class="exemptions-table">
            <thead>
                <tr>
                    <th>Client</th>
                    <th>Product</th>
                    <th>Reason</th>
                    <th>Expiry</th>
                    <th>Status</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody>
                {foreach from=$exemptions item=e}
                <tr>
                    <td>{$e.client_name}</td>
                    <td>{$e.product_name|default:'All Products'}</td>
                    <td>{$e.reason}</td>
                    <td>{$e.expiry_date|default:'Never'}</td>
                    <td>{$e.status}</td>
                    <td>
                        <button onclick="removeExemption({$e.id})">Remove</button>
                    </td>
                </tr>
                {/foreach}
            </tbody>
        </table>
    </div>

    <div id="tab-reports" class="tab-content" style="display:none;">
        <form class="report-form">
            <div class="form-row">
                <select name="country">
                    <option value="">Select Country</option>
                    {foreach from=$countries item=c}
                    <option value="{$c.code}">{$c.name}</option>
                    {/foreach}
                </select>
                <input type="date" name="start_date">
                <input type="date" name="end_date">
                <button type="button" class="btn" onclick="generateTaxReport()">Generate Report</button>
            </div>
        </form>

        <div class="report-summary">
            <div class="summary-card">
                <h4>Total Tax Collected</h4>
                <p class="value">{$currency_prefix}{$summary.total_tax}{$currency_suffix}</p>
            </div>
            <div class="summary-card">
                <h4>Transactions</h4>
                <p class="value">{$summary.total_transactions}</p>
            </div>
            <div class="summary-card">
                <h4>Unique Clients</h4>
                <p class="value">{$summary.unique_clients}</p>
            </div>
        </div>
    </div>
</div>
{/block}
```

---

## Database Schema

```sql
-- Tax rules configuration
CREATE TABLE `mod_tax_rules` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `name` VARCHAR(255) NOT NULL,
    `country_code` VARCHAR(2) NOT NULL,
    `state_code` VARCHAR(50) DEFAULT NULL,
    `city` VARCHAR(100) DEFAULT NULL,
    `jurisdiction_type` ENUM('country', 'state', 'county', 'city') DEFAULT 'country',
    `rate` DECIMAL(6,4) NOT NULL,
    `tax_type` ENUM('vat', 'gst', 'sales', 'usage') DEFAULT 'sales',
    `is_compound` TINYINT(1) DEFAULT 0,
    `product_id` INT DEFAULT 0,
    `priority` INT DEFAULT 50,
    `start_date` DATE DEFAULT NULL,
    `end_date` DATE DEFAULT NULL,
    `status` ENUM('active', 'inactive') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
    INDEX `idx_country_state` (`country_code`, `state_code`),
    INDEX `idx_jurisdiction` (`jurisdiction_type`),
    INDEX `idx_product_status` (`product_id`, `status`)
);

-- Tax exemptions
CREATE TABLE `mod_tax_exemptions` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `client_id` INT NOT NULL,
    `product_id` INT DEFAULT 0,
    `reason` VARCHAR(255) NOT NULL,
    `expiry_date` DATE DEFAULT NULL,
    `status` ENUM('active', 'inactive') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_client_product` (`client_id`, `product_id`),
    FOREIGN KEY (`client_id`) REFERENCES `tblclients`(`id`) ON DELETE CASCADE
);

-- Tax exemption groups
CREATE TABLE `mod_tax_exempt_groups` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `client_id` INT NOT NULL,
    `group_name` VARCHAR(100) NOT NULL,
    `status` ENUM('active', 'inactive') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (`client_id`) REFERENCES `tblclients`(`id`) ON DELETE CASCADE
);

-- Tax holidays
CREATE TABLE `mod_tax_holidays` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `name` VARCHAR(255) NOT NULL,
    `country_code` VARCHAR(2) NOT NULL,
    `state_code` VARCHAR(50) DEFAULT NULL,
    `start_date` DATE NOT NULL,
    `end_date` DATE NOT NULL,
    `rate_override` DECIMAL(6,4) DEFAULT NULL,
    `status` ENUM('active', 'inactive') DEFAULT 'active',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_country_date` (`country_code`, `start_date`, `end_date`)
);

-- Tax calculation history
CREATE TABLE `mod_tax_history` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `invoice_id` INT NOT NULL,
    `invoice_num` VARCHAR(50) DEFAULT NULL,
    `client_id` INT NOT NULL,
    `client_name` VARCHAR(255) DEFAULT NULL,
    `client_vat` VARCHAR(50) DEFAULT NULL,
    `amount` DECIMAL(10,2) NOT NULL,
    `tax_amount` DECIMAL(10,2) NOT NULL,
    `tax_rate` DECIMAL(6,4) DEFAULT NULL,
    `tax_rule_id` INT DEFAULT NULL,
    `country_code` VARCHAR(2) DEFAULT NULL,
    `state_code` VARCHAR(50) DEFAULT NULL,
    `product_id` INT DEFAULT 0,
    `date` DATE NOT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_invoice` (`invoice_id`),
    INDEX `idx_client` (`client_id`),
    INDEX `idx_date` (`date`),
    INDEX `idx_country_date` (`country_code`, `date`)
);
```

---

## Hook Integrations

| Hook Name | Priority | Purpose |
|-----------|----------|---------|
| `CalculateTax` | 1 | Calculate tax for order/invoice |
| `CalculateTax` | 2 | Check tax holidays |
| `InvoiceCreationPreCheck` | 1 | Apply tax to invoice items |
| `InvoiceCreationPreCheck` | 2 | Apply reverse charge VAT |
| `ClientUpdate` | 1 | Validate VAT numbers |
| `InvoicePaid` | 1 | Generate tax liability report |

---

## Development Checklist

- [ ] Create module directory structure
- [ ] Set up database tables
- [ ] Implement tax calculation engine
- [ ] Add multi-jurisdiction support
- [ ] Create VAT validation (VIES)
- [ ] Implement tax exemptions
- [ ] Add tax holiday support
- [ ] Build tax reporting system
- [ ] Create compound tax handling
- [ ] Build admin configuration interface
- [ ] Add reverse charge VAT logic
- [ ] Implement tax audit logging
- [ ] Test various tax scenarios
- [ ] Verify VAT number validation
- [ ] Test tax exemption logic
- [ ] Add tax export functionality
- [ ] Implement tax reconciliation
- [ ] Test with multiple jurisdictions
- [ ] Add tax API integration
- [ ] Verify compliance with tax regulations