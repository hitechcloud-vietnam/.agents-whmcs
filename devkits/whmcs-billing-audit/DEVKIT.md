# WHMCS Billing Audit - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_billing_audit_config() { return ['name' => 'Billing Audit', 'description' => 'Billing audit trail', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_billing_audit_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_billing_audit` (`id` INT(11) NOT NULL AUTO_INCREMENT, `entity_type` VARCHAR(50) NOT NULL, `entity_id` INT(11) NOT NULL, `action` VARCHAR(50) NOT NULL, `old_value` TEXT, `new_value` TEXT, `performed_by` INT(11) DEFAULT NULL, `ip_address` VARCHAR(45), `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_billing_audit_deactivate() { return ['status' => 'success']; }

function whmcs_billing_audit_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Billing Audit Trail</h2>';
    
    $audit = select_query('mod_billing_audit', '*', '', 'id', 'DESC', '100');
    echo '<table class="datatable"><thead><tr><th>Date</th><th>Entity</th><th>Action</th><th>User</th><th>IP</th></tr></thead><tbody>';
    while ($a = mysql_fetch_array($audit)) {
        $user = mysql_fetch_array(select_query('tbladmin', 'username', ['id' => $a['performed_by']]));
        echo '<tr><td>' . $a['created_at'] . '</td><td>' . $a['entity_type'] . ' #' . $a['entity_id'] . '</td><td>' . $a['action'] . '</td><td>' . ($user['username'] ?? 'System') . '</td><td>' . $a['ip_address'] . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

function logBillingAudit($entityType, $entityId, $action, $oldValue, $newValue) {
    insert_query('mod_billing_audit', [
        'entity_type' => $entityType,
        'entity_id' => $entityId,
        'action' => $action,
        'old_value' => json_encode($oldValue),
        'new_value' => json_encode($newValue),
        'performed_by' => $_SESSION['adminid'] ?? null,
        'ip_address' => $_SERVER['REMOTE_ADDR']
    ]);
}

add_hook('InvoicePaid', 1, function($vars) { logBillingAudit('invoice', $vars['invoiceid'], 'paid', null, ['total' => $vars['total']]); });
add_hook('InvoiceRefunded', 1, function($vars) { logBillingAudit('invoice', $vars['invoiceid'], 'refunded', null, ['amount' => $vars['amount']]); });
```