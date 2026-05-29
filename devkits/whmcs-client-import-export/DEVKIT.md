# WHMCS Client Import Export - Complete Module

```php
<?php
if (!defined("WHMCS")) { die("Direct access denied"); }

function whmcs_client_import_export_config() {
    return [
        'name' => 'Client Import/Export',
        'description' => 'Bulk import/export clients with CSV support',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'validate_emails' => ['Type' => 'yesno', 'Default' => 'on', 'Description' => 'Validate emails on import'],
            'allow_duplicates' => ['Type' => 'yesno', 'Default' => 'off', 'Description' => 'Allow duplicate emails']
        ]
    ];
}

function whmcs_client_import_export_activate() {
    full_query("CREATE TABLE IF NOT EXISTS `mod_import_history` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `file_name` VARCHAR(255) NOT NULL,
        `total_rows` INT(11) DEFAULT 0,
        `imported` INT(11) DEFAULT 0,
        `failed` INT(11) DEFAULT 0,
        `status` ENUM('pending','processing','completed','failed') DEFAULT 'pending',
        `errors` TEXT,
        `imported_at` DATETIME DEFAULT NULL,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    full_query("CREATE TABLE IF NOT EXISTS `mod_import_field_mapping` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `mapping_name` VARCHAR(255) NOT NULL,
        `field_mappings` TEXT,
        `is_active` TINYINT(1) DEFAULT 1,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4");
    
    return ['status' => 'success'];
}

function whmcs_client_import_export_deactivate() { return ['status' => 'success']; }

function whmcs_client_import_export_output($vars) {
    echo '<div class="whmcs-module-admin"><h2>Client Import/Export</h2>';
    
    // Handle CSV upload
    if ($_FILES['csv_file']) {
        $filename = $_FILES['csv_file']['name'];
        $filepath = '/tmp/' . $filename;
        move_uploaded_file($_FILES['csv_file']['tmp_name'], $filepath);
        
        $handle = fopen($filepath, 'r');
        $headers = fgetcsv($handle);
        $rowCount = 0;
        $imported = 0;
        $failed = 0;
        
        while (($data = fgetcsv($handle)) !== false) {
            $rowCount++;
            $email = $data[array_search('email', $headers)] ?? '';
            
            // Validate
            if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
                $failed++;
                continue;
            }
            
            // Check duplicate
            $existing = mysql_num_rows(select_query('tblclients', 'id', ['email' => $email]));
            if ($existing > 0 && !get_config('allow_duplicates')) {
                $failed++;
                continue;
            }
            
            // Create client
            $firstname = $data[array_search('firstname', $headers)] ?? '';
            $lastname = $data[array_search('lastname', $headers)] ?? '';
            
            insert_query('tblclients', [
                'firstname' => $firstname,
                'lastname' => $lastname,
                'email' => $email,
                'datecreated' => date('Y-m-d H:i:s'),
                'status' => 'Active'
            ]);
            $imported++;
        }
        fclose($filepath);
        
        insert_query('mod_import_history', [
            'file_name' => $filename,
            'total_rows' => $rowCount,
            'imported' => $imported,
            'failed' => $failed,
            'status' => 'completed',
            'imported_at' => date('Y-m-d H:i:s')
        ]);
        
        echo '<div class="alert alert-success">Import completed: ' . $imported . ' imported, ' . $failed . ' failed</div>';
    }
    
    echo '<div class="panel panel-default">
          <div class="panel-heading">Import Clients (CSV)</div>
          <div class="panel-body">
          <form method="post" enctype="multipart/form-data">
          <div class="form-group">
          <label>CSV File (columns: firstname, lastname, email, company, phone)</label>
          <input type="file" name="csv_file" class="form-control" accept=".csv" required />
          </div>
          <button type="submit" class="btn btn-primary">Import</button>
          </form>
          </div></div>';
    
    // Export button
    echo '<div class="panel panel-default" style="margin-top:20px;">
          <div class="panel-heading">Export Clients</div>
          <div class="panel-body">
          <form method="post" action="?module=whmcs_client_import_export&action=export">
          <button type="submit" name="export_csv" class="btn btn-primary">Export All to CSV</button>
          </form>
          </div></div>';
    
    // Import history
    $history = select_query('mod_import_history', '*', '', 'id', 'DESC', '10');
    echo '<table class="datatable" style="margin-top:20px;">
          <thead><tr><th>File</th><th>Total</th><th>Imported</th><th>Failed</th><th>Date</th></tr></thead>
          <tbody>';
    while ($h = mysql_fetch_array($history)) {
        echo '<tr><td>' . $h['file_name'] . '</td>
              <td>' . $h['total_rows'] . '</td>
              <td>' . $h['imported'] . '</td>
              <td>' . $h['failed'] . '</td>
              <td>' . $h['imported_at'] . '</td></tr>';
    }
    echo '</tbody></table></div>';
}

if ($_GET['action'] == 'export' && $_POST['export_csv']) {
    header('Content-Type: text/csv');
    header('Content-Disposition: attachment; filename="clients_export_' . date('Ymd') . '.csv"');
    
    $output = fopen('php://output', 'w');
    fputcsv($output, ['firstname', 'lastname', 'email', 'company', 'phone', 'datecreated']);
    
    $clients = select_query('tblclients', '*', '', 'id');
    while ($c = mysql_fetch_array($clients)) {
        fputcsv($output, [$c['firstname'], $c['lastname'], $c['email'], $c['companyname'], $c['phonenumber'], $c['datecreated']]);
    }
    fclose($output);
    exit;
}
```