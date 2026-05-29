# WHMCS Invoice Email - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_invoice_email_config() { return ['name' => 'Invoice Email', 'description' => 'Custom invoice emails', 'author' => 'DevKit Generator', 'version' => '1.0.0', 'fields' => [
    'auto_send' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Auto-send on invoice creation'],
    'include_pdf' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Include PDF attachment']
]];}

function whmcs_invoice_email_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_invoice_email_templates` (`id` INT(11) NOT NULL AUTO_INCREMENT, `template_name` VARCHAR(255) NOT NULL, `subject` VARCHAR(255), `body` TEXT, `is_default` TINYINT(1) DEFAULT 0, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    full_query("CREATE TABLE IF NOT EXISTS `mod_invoice_email_log` (`id` INT(11) NOT NULL AUTO_INCREMENT, `invoice_id` INT(11) NOT NULL, `sent_to` VARCHAR(255), `sent_at` DATETIME DEFAULT CURRENT_TIMESTAMP, `status` ENUM('sent','failed') DEFAULT 'sent', PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_invoice_email_deactivate() { return ['status' => 'success']; }

function whmcs_invoice_email_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Invoice Email Templates</h2>';
    if ($_POST['save_template']) {
        insert_query('mod_invoice_email_templates', ['template_name' => $_POST['template_name'], 'subject' => $_POST['subject'], 'body' => $_POST['body'], 'is_default' => $_POST['is_default'] ?? 0]);
        echo '<div class="alert alert-success">Template saved!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Create Email Template</div><div class="panel-body">
          <div class="form-group"><label>Template Name</label><input type="text" name="template_name" class="form-control" required /></div>
          <div class="form-group"><label>Subject</label><input type="text" name="subject" class="form-control" placeholder="Invoice #{invoice_num} from {company_name}" /></div>
          <div class="form-group"><label>Body (HTML)</label><textarea name="body" class="form-control" rows="10"></textarea></div>
          <div class="form-group"><label><input type="checkbox" name="is_default" value="1" /> Default template</label></div>
          <button type="submit" name="save_template" class="btn btn-primary">Save Template</button></div></form>';
    
    $templates = select_query('mod_invoice_email_templates', '*', '', 'id');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Name</th><th>Subject</th><th>Default</th></tr></thead><tbody>';
    while ($t = mysql_fetch_array($templates)) { echo '<tr><td>' . $t['template_name'] . '</td><td>' . $t['subject'] . '</td><td>' . ($t['is_default'] ? 'Yes' : 'No') . '</td></tr>'; }
    echo '</tbody></table></div>';
}

function sendInvoiceEmail($invoiceId, $templateId = null) {
    $invoice = mysql_fetch_array(select_query('tblinvoices', '*', ['id' => $invoiceId]));
    $client = mysql_fetch_array(select_query('tblclients', '*', ['id' => $invoice['userid']]));
    
    if (!$templateId) {
        $template = mysql_fetch_array(select_query('mod_invoice_email_templates', '*', ['is_default' => 1]));
        $templateId = $template['id'] ?? null;
    }
    
    $subject = str_replace(['{invoice_num}', '{company_name}'], [$invoice['id'], 'Your Company'], $template['subject']);
    $body = str_replace(['{client_name}', '{invoice_total}'], [$client['firstname'], $invoice['total']], $template['body']);
    
    send_email($client['email'], $subject, $body);
    insert_query('mod_invoice_email_log', ['invoice_id' => $invoiceId, 'sent_to' => $client['email'], 'status' => 'sent']);
}

add_hook('InvoiceCreation', 1, function($vars) { return ['send_invoice_email' => true]; });
```