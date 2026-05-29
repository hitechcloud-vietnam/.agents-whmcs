# WHMCS Invoice Merge - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_invoice_merge_config() { return ['name' => 'Invoice Merge', 'description' => 'Merge multiple invoices', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_invoice_merge_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_merged_invoices` (`id` INT(11) NOT NULL AUTO_INCREMENT, `new_invoice_id` INT(11), `source_invoice_ids` TEXT, `merged_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_invoice_merge_deactivate() { return ['status' => 'success']; }

function whmcs_invoice_merge_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Invoice Merge</h2>';
    if ($_POST['merge_invoices']) {
        $ids = explode(',', $_POST['invoice_ids']);
        $firstId = $ids[0];
        $items = ''; $total = 0;
        foreach ($ids as $id) {
            $inv = mysql_fetch_array(select_query('tblinvoices', '*', ['id' => trim($id)]));
            if ($inv) { $items .= $inv['id'] . ','; $total += $inv['total']; }
        }
        update_query('tblinvoices', ['total' => $total], ['id' => $firstId]);
        insert_query('mod_merged_invoices', ['new_invoice_id' => $firstId, 'source_invoice_ids' => rtrim($items, ',')]);
        echo '<div class="alert alert-success">Invoices merged into #' . $firstId . '!</div>';
    }
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Merge Invoices</div><div class="panel-body">
          <div class="form-group"><label>Invoice IDs (comma-separated)</label><input type="text" name="invoice_ids" class="form-control" placeholder="1,2,3" required /></div>
          <button type="submit" name="merge_invoices" class="btn btn-primary">Merge</button></div></form>';
}
```