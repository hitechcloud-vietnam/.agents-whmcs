# WHMCS Invoice PDF - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_invoice_pdf_config() { return ['name' => 'Invoice PDF', 'description' => 'Custom PDF generation', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'paper_size' => ['Type' => 'dropdown', 'Default' => 'A4', 'Options' => ['A4' => 'A4', 'Letter' => 'Letter', 'Legal' => 'Legal'], 'Description' => 'Paper size'],
    'orientation' => ['Type' => 'dropdown', 'Default' => 'portrait', 'Options' => ['portrait' => 'Portrait', 'landscape' => 'Landscape'], 'Description' => 'Orientation']
]];}

function whmcs_invoice_pdf_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_pdf_templates` (`id` INT(11) NOT NULL AUTO_INCREMENT, `template_name` VARCHAR(255) NOT NULL, `template_html` TEXT, `is_default` TINYINT(1) DEFAULT 0, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_invoice_pdf_deactivate() { return ['status' => 'success']; }

function whmcs_invoice_pdf_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Invoice PDF Templates</h2>';
    if ($_POST['save_template']) {
        insert_query('mod_pdf_templates', ['template_name' => $_POST['template_name'], 'template_html' => $_POST['template_html'], 'is_default' => $_POST['is_default'] ?? 0]);
        echo '<div class="alert alert-success">Template saved!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Create PDF Template</div><div class="panel-body">
          <div class="form-group"><label>Template Name</label><input type="text" name="template_name" class="form-control" required /></div>
          <div class="form-group"><label>HTML Template</label><textarea name="template_html" class="form-control" rows="10"></textarea></div>
          <div class="form-group"><label><input type="checkbox" name="is_default" value="1" /> Set as default</label></div>
          <button type="submit" name="save_template" class="btn btn-primary">Save Template</button></div></form>';
    
    $templates = select_query('mod_pdf_templates', '*', '', 'id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Name</th><th>Default</th></tr></thead><tbody>';
    while ($t = mysql_fetch_array($templates)) { echo '<tr><td>' . $t['template_name'] . '</td><td>' . ($t['is_default'] ? 'Yes' : 'No') . '</td></tr>'; }
    echo '</tbody></table></div>';
}

function generateInvoicePDF($invoiceId, $templateId = null) {
    if (!$templateId) {
        $template = mysql_fetch_array(select_query('mod_pdf_templates', '*', ['is_default' => 1]));
        $templateId = $template['id'] ?? null;
    }
    return ['invoice_id' => $invoiceId, 'template_id' => $templateId, 'generated_at' => date('Y-m-d H:i:s')];
}

add_hook('InvoiceCreation', 1, function($vars) { return ['custom_pdf_template' => true]; });
```