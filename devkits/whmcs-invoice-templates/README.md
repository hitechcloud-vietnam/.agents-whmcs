# WHMCS Invoice Templates Module

```php
<?php
/**
 * WHMCS Invoice Templates Module
 * 
 * Custom invoice template management with multiple
 * templates and preview.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function invoicetemplates_MetaData() {
    return array('DisplayName' => 'Invoice Templates', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function invoicetemplates_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Invoice Templates'),
        'DefaultTemplate' => array('Type' => 'text', 'Size' => '50', 'Default' => 'default', 'Description' => 'Default template key'),
        'PDFEngine' => array('Type' => 'dropdown', 'Options' => 'tcpdf,dompdf,wkhtmltopdf', 'Default' => 'tcpdf', 'Description' => 'PDF generation engine'),
        'PaperSize' => array('Type' => 'dropdown', 'Options' => 'A4,Letter,A5,Legal', 'Default' => 'A4', 'Description' => 'Paper size'),
        'Orientation' => array('Type' => 'dropdown', 'Options' => 'portrait,landscape', 'Default' => 'portrait', 'Description' => 'Page orientation'),
        'EnableTranslations' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable translations'),
        'CacheEnabled' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Cache compiled templates'),
        'LogoMaxWidth' => array('Type' => 'text', 'Size' => '10', 'Default' => '200', 'Description' => 'Logo max width (px)'),
        'LogoMaxHeight' => array('Type' => 'text', 'Size' => '10', 'Default' => '100', 'Description' => 'Logo max height (px)')
    );
}

function invoicetemplates_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_invoicetemplates_templates', "
            CREATE TABLE `mod_invoicetemplates_templates` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `template_key` VARCHAR(100) UNIQUE NOT NULL,
                `template_name` VARCHAR(255) NOT NULL,
                `category` VARCHAR(50) DEFAULT 'standard',
                `header_html` TEXT NULL,
                `body_html` TEXT NULL,
                `footer_html` TEXT NULL,
                `css` TEXT NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `is_default` TINYINT(1) DEFAULT 0,
                `preview_data` JSON NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                INDEX `idx_category` (`category`),
                INDEX `idx_active` (`is_active`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_invoicetemplates_versions', "
            CREATE TABLE `mod_invoicetemplates_versions` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `template_id` INT NOT NULL,
                `version_number` VARCHAR(20) NOT NULL,
                `change_notes` TEXT NULL,
                `header_html` TEXT NULL,
                `body_html` TEXT NULL,
                `footer_html` TEXT NULL,
                `css` TEXT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_template` (`template_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_invoicetemplates_translations', "
            CREATE TABLE `mod_invoicetemplates_translations` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `template_id` INT NOT NULL,
                `language` VARCHAR(20) NOT NULL,
                `translations` JSON NOT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_template_lang` (`template_id`, `language`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_invoicetemplates_settings', "
            CREATE TABLE `mod_invoicetemplates_settings` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `setting_key` VARCHAR(100) UNIQUE NOT NULL,
                `setting_value` TEXT NULL,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_invoicetemplates_client_settings', "
            CREATE TABLE `mod_invoicetemplates_client_settings` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT UNIQUE NOT NULL,
                `template_id` INT NULL,
                `language` VARCHAR(20) DEFAULT 'en',
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_user` (`user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        $defaultHtml = '<!DOCTYPE html><html><head><style>' . "\n" .
            '.invoice-header { text-align: center; margin-bottom: 30px; }' . "\n" .
            '.invoice-table { width: 100%; border-collapse: collapse; margin: 20px 0; }' . "\n" .
            '.invoice-table th, .invoice-table td { border: 1px solid #ddd; padding: 10px; text-align: left; }' . "\n" .
            '.invoice-table th { background-color: #f5f5f5; }' . "\n" .
            '.invoice-total { text-align: right; margin-top: 20px; font-size: 18px; font-weight: bold; }' . "\n" .
            '.invoice-footer { text-align: center; margin-top: 50px; padding-top: 20px; border-top: 1px solid #ddd; }' . "\n" .
            '</style></head><body>' . "\n" .
            '<div class="invoice-header"><h1>{$company_name}</h1><p>{$company_address}</p></div>' . "\n" .
            '<div class="invoice-info"><p><strong>Invoice:</strong> {$invoice_number}</p>' . "\n" .
            '<p><strong>Date:</strong> {$invoice_date}</p><p><strong>Due:</strong> {$invoice_due_date}</p></div>' . "\n" .
            '<div class="client-info"><p><strong>Bill To:</strong><br>{$client_name}<br>{$client_address}</p></div>' . "\n" .
            '<table class="invoice-table"><thead><tr><th>Description</th><th>Qty</th><th>Price</th><th>Total</th></tr></thead>' . "\n" .
            '<tbody>{$items_rows}</tbody></table>' . "\n" .
            '<div class="invoice-totals"><p>Subtotal: {$invoice_subtotal}</p>' . "\n" .
            '<p>Tax: {$invoice_tax}</p><p><strong>Total: {$invoice_total}</strong></p></div>' . "\n" .
            '<div class="invoice-footer"><p>{$notes}</p><p>Thank you for your business!</p></div>' . "\n" .
            '</body></html>';
        Capsule::table('mod_invoicetemplates_templates')->insert(array(
            'template_key' => 'default', 'template_name' => 'Default Template', 'category' => 'standard',
            'header_html' => $defaultHtml, 'is_default' => 1
        ));
        return array('status' => 'success', 'description' => 'Invoice Templates module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function invoicetemplates_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function invoicetemplates_CreateTemplate($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        if (!empty($data['is_default'])) { Capsule::table('mod_invoicetemplates_templates')->update(array('is_default' => 0)); }
        $templateId = Capsule::table('mod_invoicetemplates_templates')->insertGetId(array(
            'template_key' => $data['template_key'], 'template_name' => $data['template_name'],
            'category' => $data['category'] ?? 'standard', 'header_html' => $data['header_html'] ?? '',
            'body_html' => $data['body_html'] ?? '', 'footer_html' => $data['footer_html'] ?? '',
            'css' => $data['css'] ?? '', 'is_default' => $data['is_default'] ?? 0
        ));
        return array('success' => true, 'template_id' => $templateId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function invoicetemplates_GetTemplate($templateId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_invoicetemplates_templates')->where('id', $templateId)->orWhere('template_key', $templateId)->first();
}

function invoicetemplates_GetTemplates($filters = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_invoicetemplates_templates');
    if (!empty($filters['category'])) { $query->where('category', $filters['category']); }
    if (isset($filters['is_active'])) { $query->where('is_active', $filters['is_active']); }
    return $query->get();
}

function invoicetemplates_UpdateTemplate($templateId, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        if (!empty($data['is_default'])) { Capsule::table('mod_invoicetemplates_templates')->update(array('is_default' => 0)); }
        $update = array_filter(array(
            'template_name' => $data['template_name'] ?? null,
            'template_key' => $data['template_key'] ?? null,
            'category' => $data['category'] ?? null,
            'header_html' => $data['header_html'] ?? null,
            'body_html' => $data['body_html'] ?? null,
            'footer_html' => $data['footer_html'] ?? null,
            'css' => $data['css'] ?? null,
            'is_active' => isset($data['is_active']) ? (int)$data['is_active'] : null,
            'is_default' => isset($data['is_default']) ? (int)$data['is_default'] : null
        ), function($v) { return $v !== null; });
        Capsule::table('mod_invoicetemplates_templates')->where('id', $templateId)->update($update);
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function invoicetemplates_CloneTemplate($templateId, $newName) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $original = invoicetemplates_GetTemplate($templateId);
        if (!$original) { return array('success' => false, 'error' => 'Template not found'); }
        $newKey = strtolower(preg_replace('/[^a-zA-Z0-9]/', '_', $newName)) . '_' . substr(md5(uniqid()), 0, 8);
        $newId = Capsule::table('mod_invoicetemplates_templates')->insertGetId(array(
            'template_key' => $newKey, 'template_name' => $newName, 'category' => $original->category,
            'header_html' => $original->header_html, 'body_html' => $original->body_html,
            'footer_html' => $original->footer_html, 'css' => $original->css
        ));
        return array('success' => true, 'template_id' => $newId, 'template_key' => $newKey);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function invoicetemplates_DeleteTemplate($templateId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_invoicetemplates_templates')->where('id', $templateId)->update(array('is_active' => 0));
    return array('success' => true);
}

function invoicetemplates_SetDefault($templateId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_invoicetemplates_templates')->update(array('is_default' => 0));
    Capsule::table('mod_invoicetemplates_templates')->where('id', $templateId)->update(array('is_default' => 1));
    return array('success' => true);
}

function invoicetemplates_GetVariables() {
    return array(
        'client_name', 'client_email', 'client_address', 'client_phone', 'client_company',
        'invoice_number', 'invoice_date', 'invoice_due_date', 'invoice_paid_date',
        'invoice_subtotal', 'invoice_tax', 'invoice_total', 'invoice_currency',
        'invoice_tax_rate', 'invoice_tax_rate_2', 'invoice_status', 'invoice_type',
        'payment_method', 'payment_details', 'items_rows', 'notes',
        'company_name', 'company_address', 'company_email', 'company_phone',
        'company_logo', 'company_url', 'custom_fields'
    );
}

function invoicetemplates_GetPlaceholders() {
    return array(
        '{$client_name}', '{$client_email}', '{$client_address}', '{$client_company}',
        '{$invoice_number}', '{$invoice_date}', '{$invoice_due_date}',
        '{$invoice_subtotal}', '{$invoice_tax}', '{$invoice_total}',
        '{$company_name}', '{$company_address}', '{$company_logo}',
        '{$items_rows}', '{$notes}'
    );
}

function invoicetemplates_GetCategories() {
    return array('standard', 'proforma', 'credit_note', 'receipt', 'dunning', 'quote', 'customized');
}

function invoicetemplates_GeneratePreview($templateId, $invoiceId = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $template = invoicetemplates_GetTemplate($templateId);
    if (!$template) { return array('success' => false, 'error' => 'Template not found'); }
    $vars = invoicetemplates_GetPreviewData($invoiceId);
    $html = $template->header_html;
    foreach ($vars as $key => $value) { $html = str_replace('{$' . $key . '}', htmlspecialchars($value), $html); }
    return array('success' => true, 'html' => $html);
}

function invoicetemplates_GetPreviewData($invoiceId) {
    if ($invoiceId) {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
        $client = Capsule::table('tblclients')->where('id', $invoice->userid)->first();
        $items = Capsule::table('tblinvoiceitems')->where('invoiceid', $invoiceId)->get();
        $itemsHtml = '';
        foreach ($items as $item) { $itemsHtml .= "<tr><td>{$item->description}</td><td>{$item->qty}</td><td>\${$item->unitprice}</td><td>\${$item->amount}</td></tr>"; }
        return array(
            'client_name' => $client ? $client->firstname . ' ' . $client->lastname : 'John Doe',
            'client_email' => $client ? $client->email : 'john@example.com',
            'client_address' => $client ? nl2br($client->address1 . "\n" . $client->city . ', ' . $client->state . ' ' . $client->postcode) : '123 Main St',
            'invoice_number' => $invoice ? $invoice->invoicenum ?? $invoice->id : 'INV-0001',
            'invoice_date' => $invoice ? date('Y-m-d', strtotime($invoice->date)) : date('Y-m-d'),
            'invoice_due_date' => $invoice ? date('Y-m-d', strtotime($invoice->duedate)) : date('Y-m-d', strtotime('+30 days')),
            'invoice_subtotal' => $invoice ? formatCurrency($invoice->subtotal) : '$100.00',
            'invoice_tax' => $invoice ? formatCurrency($invoice->tax) : '$10.00',
            'invoice_total' => $invoice ? formatCurrency($invoice->total) : '$110.00',
            'invoice_currency' => $invoice ? $invoice->currency : 'USD',
            'items_rows' => $itemsHtml ?: '<tr><td>Sample Item</td><td>1</td><td>$100.00</td><td>$100.00</td></tr>',
            'company_name' => 'Your Company Name',
            'company_address' => '123 Business St, City, Country'
        );
    }
    return array('client_name' => 'John Doe', 'client_email' => 'john@example.com',
        'client_address' => '123 Main St, City, ST 12345', 'invoice_number' => 'INV-0001',
        'invoice_date' => date('Y-m-d'), 'invoice_due_date' => date('Y-m-d', strtotime('+30 days')),
        'invoice_subtotal' => '$100.00', 'invoice_tax' => '$10.00', 'invoice_total' => '$110.00',
        'invoice_currency' => 'USD', 'items_rows' => '<tr><td>Sample Item</td><td>1</td><td>$100.00</td><td>$100.00</td></tr>',
        'company_name' => 'Your Company Name', 'company_address' => '123 Business St, City, Country');
}

function invoicetemplates_SetInvoiceTemplate($invoiceId, $templateId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('tblinvoices')->where('id', $invoiceId)->update(array('template_id' => $templateId));
    return array('success' => true);
}

function invoicetemplates_GetInvoiceTemplate($invoiceId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
    if ($invoice && !empty($invoice->template_id)) {
        return invoicetemplates_GetTemplate($invoice->template_id);
    }
    return Capsule::table('mod_invoicetemplates_templates')->where('is_default', 1)->first();
}

function invoicetemplates_SetClientTemplate($userId, $templateId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_invoicetemplates_client_settings')->updateOrInsert(
        array('user_id' => $userId), array('template_id' => $templateId)
    );
    return array('success' => true);
}

function invoicetemplates_GetClientTemplate($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $setting = Capsule::table('mod_invoicetemplates_client_settings')->where('user_id', $userId)->first();
    if ($setting && $setting->template_id) {
        return invoicetemplates_GetTemplate($setting->template_id);
    }
    return Capsule::table('mod_invoicetemplates_templates')->where('is_default', 1)->first();
}

function invoicetemplates_AddTranslation($templateId, $language, $translations) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_invoicetemplates_translations')->updateOrInsert(
            array('template_id' => $templateId, 'language' => $language),
            array('translations' => json_encode($translations))
        );
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function invoicetemplates_ExportTemplate($templateId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $template = invoicetemplates_GetTemplate($templateId);
    if (!$template) { return array('success' => false, 'error' => 'Template not found'); }
    return array('success' => true, 'data' => json_encode($template));
}

function invoicetemplates_ImportTemplate($jsonData) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $data = json_decode($jsonData, true);
        if (!$data) { return array('success' => false, 'error' => 'Invalid JSON'); }
        $data['template_key'] = $data['template_key'] . '_import_' . substr(md5(uniqid()), 0, 8);
        $newId = Capsule::table('mod_invoicetemplates_templates')->insertGetId(array(
            'template_key' => $data['template_key'], 'template_name' => $data['template_name'],
            'category' => $data['category'] ?? 'standard', 'header_html' => $data['header_html'] ?? '',
            'body_html' => $data['body_html'] ?? '', 'footer_html' => $data['footer_html'] ?? '',
            'css' => $data['css'] ?? ''
        ));
        return array('success' => true, 'template_id' => $newId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function invoicetemplates_CreateVersion($templateId, $changeNotes) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $template = invoicetemplates_GetTemplate($templateId);
        if (!$template) { return array('success' => false, 'error' => 'Template not found'); }
        $lastVersion = Capsule::table('mod_invoicetemplates_versions')->where('template_id', $templateId)->max('version_number');
        $version = '1.0';
        if ($lastVersion) {
            $parts = explode('.', $lastVersion);
            $version = ($parts[0] + 1) . '.0';
        }
        $versionId = Capsule::table('mod_invoicetemplates_versions')->insertGetId(array(
            'template_id' => $templateId, 'version_number' => $version, 'change_notes' => $changeNotes,
            'header_html' => $template->header_html, 'body_html' => $template->body_html,
            'footer_html' => $template->footer_html, 'css' => $template->css
        ));
        return array('success' => true, 'version_id' => $versionId, 'version' => $version);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function invoicetemplates_GetTemplateVersions($templateId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_invoicetemplates_versions')->where('template_id', $templateId)->orderBy('created_at', 'desc')->get();
}

function invoicetemplates_RestoreVersion($versionId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $version = Capsule::table('mod_invoicetemplates_versions')->where('id', $versionId)->first();
        if (!$version) { return array('success' => false, 'error' => 'Version not found'); }
        Capsule::table('mod_invoicetemplates_templates')->where('id', $version->template_id)->update(array(
            'header_html' => $version->header_html, 'body_html' => $version->body_html,
            'footer_html' => $version->footer_html, 'css' => $version->css
        ));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}
```
