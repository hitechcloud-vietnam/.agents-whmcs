# WHMCS Billing Export - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_billing_export_config() { return ['name' => 'Billing Export', 'description' => 'Export billing data', 'author' => 'DevKit Generator', 'version' => '1.0.0']; }

function whmcs_billing_export_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_export_history` (`id` INT(11) NOT NULL AUTO_INCREMENT, `export_type` VARCHAR(50) NOT NULL, `format` VARCHAR(20), `file_name` VARCHAR(255), `records` INT(11), `exported_at` DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    return ['status' => 'success'];
}

function whmcs_billing_export_deactivate() { return ['status' => 'success']; }

function whmcs_billing_export_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Billing Export</h2>';
    
    if ($_POST['export']) {
        $format = $_POST['format'];
        $data = select_query('tblinvoices', '*', '', 'id', 'DESC');
        
        if ($format == 'csv') {
            header('Content-Type: text/csv');
            header('Content-Disposition: attachment; filename="billing_export.csv"');
            $output = fopen('php://output', 'w');
            fputcsv($output, ['ID', 'Client', 'Date', 'Due Date', 'Total', 'Status']);
            $count = 0;
            while ($row = mysql_fetch_array($data)) {
                $c = mysql_fetch_array(select_query('tblclients', 'email', ['id' => $row['userid']]));
                fputcsv($output, [$row['id'], $c['email'], $row['date'], $row['duedate'], $row['total'], $row['status']]);
                $count++;
            }
            fclose($output);
        } elseif ($format == 'json') {
            header('Content-Type: application/json');
            header('Content-Disposition: attachment; filename="billing_export.json"');
            $export = [];
            while ($row = mysql_fetch_array($data)) { $export[] = $row; }
            echo json_encode($export);
        }
        
        insert_query('mod_export_history', ['export_type' => 'invoices', 'format' => $format, 'records' => $count]);
        exit;
    }
    
    echo '<form method="post" class="panel panel-default"><div class="panel-heading">Export Billing Data</div><div class="panel-body">
          <div class="form-group"><label>Format</label><select name="format" class="form-control"><option value="csv">CSV</option><option value="json">JSON</option><option value="xml">XML</option></select></div>
          <button type="submit" name="export" class="btn btn-primary">Export</button></div></form>';
    
    $history = select_query('mod_export_history', '*', '', 'id', 'DESC', '20');
    echo '<table class="datatable" style="margin-top:20px;"><thead><tr><th>Type</th><th>Format</th><th>Records</th><th>Date</th></tr></thead><tbody>';
    while ($h = mysql_fetch_array($history)) { echo '<tr><td>' . $h['export_type'] . '</td><td>' . $h['format'] . '</td><td>' . $h['records'] . '</td><td>' . $h['exported_at'] . '</td></tr>'; }
    echo '</tbody></table></div>';
}
```