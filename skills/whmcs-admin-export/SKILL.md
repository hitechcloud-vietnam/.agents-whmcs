# WHMCS Admin Export

## Overview
Guide for implementing data export functionality in WHMCS admin area. Covers CSV, Excel export, and scheduled exports.

## Export System

### Export Handler

```php
<?php
// /includes/hooks/admin_export.php

add_hook("AdminExportData", 1, function(array $params) {
    $type = $params["type"];
    $format = $params["format"] ?? "csv";
    $filters = $params["filters"] ?? [];
    
    // Get data
    $data = getExportData($type, $filters);
    
    // Generate export
    switch ($format) {
        case "csv":
            return exportToCSV($data, $type);
        case "xlsx":
            return exportToExcel($data, $type);
        case "json":
            return exportToJSON($data, $type);
        default:
            return ["error" => "Unsupported format"];
    }
});

function getExportData(string $type, array $filters): array
{
    switch ($type) {
        case "clients":
            return getClientsForExport($filters);
        case "services":
            return getServicesForExport($filters);
        case "invoices":
            return getInvoicesForExport($filters);
        case "orders":
            return getOrdersForExport($filters);
        default:
            return [];
    }
}

function getClientsForExport(array $filters): array
{
    $query = Capsule::table("tblclients")
        ->select([
            "tblclients.id",
            "tblclients.email",
            "tblclients.firstname",
            "tblclients.lastname",
            "tblclients.companyname",
            "tblclients.phonenumber",
            "tblclients.address1",
            "tblclients.city",
            "tblclients.state",
            "tblclients.postcode",
            "tblclients.country",
            "tblclients.status",
            "tblclients.datecreated",
            "tblclients.lastlogin"
        ]);
    
    // Apply filters
    if (!empty($filters["status"])) {
        $query->where("tblclients.status", $filters["status"]);
    }
    
    if (!empty($filters["date_from"])) {
        $query->where("tblclients.datecreated", ">=", $filters["date_from"]);
    }
    
    if (!empty($filters["date_to"])) {
        $query->where("tblclients.datecreated", "<=", $filters["date_to"]);
    }
    
    if (!empty($filters["search"])) {
        $search = $filters["search"];
        $query->where(function($q) use ($search) {
            $q->where("tblclients.email", "like", "%{$search}%")
              ->orWhere("tblclients.firstname", "like", "%{$search}%")
              ->orWhere("tblclients.lastname", "like", "%{$search}%");
        });
    }
    
    return $query->get()->toArray();
}
```

### CSV Export

```php
function exportToCSV(array $data, string $type): array
{
    if (empty($data)) {
        return ["error" => "No data to export"];
    }
    
    // Set headers for download
    $filename = $type . "_export_" . date("Y-m-d_His") . ".csv";
    
    header("Content-Type: text/csv");
    header("Content-Disposition: attachment; filename=\"{$filename}\"");
    header("Cache-Control: no-cache, no-store, must-revalidate");
    header("Pragma: no-cache");
    header("Expires: 0");
    
    $output = fopen("php://output", "w");
    
    // BOM for UTF-8
    fprintf($output, chr(0xEF).chr(0xBB).chr(0xBF));
    
    // Headers
    if (!empty($data)) {
        fputcsv($output, array_keys((array)$data[0]));
    }
    
    // Data rows
    foreach ($data as $row) {
        fputcsv($output, (array)$row);
    }
    
    fclose($output);
    exit;
}
```

### Excel Export

```php
function exportToExcel(array $data, string $type): array
{
    if (empty($data)) {
        return ["error" => "No data to export"];
    }
    
    require_once __DIR__ . "/lib/SpreadsheetWriter.php";
    
    $filename = $type . "_export_" . date("Y-m-d_His") . ".xlsx";
    
    $spreadsheet = new \PhpOffice\PhpSpreadsheet\Spreadsheet();
    $sheet = $spreadsheet->getActiveSheet();
    
    // Headers
    $headers = array_keys((array)$data[0]);
    $col = 1;
    foreach ($headers as $header) {
        $sheet->setCellValueByColumnAndRow($col, 1, ucfirst(str_replace("_", " ", $header)));
        $col++;
    }
    
    // Data
    $row = 2;
    foreach ($data as $record) {
        $col = 1;
        foreach ((array)$record as $value) {
            $sheet->setCellValueByColumnAndRow($col, $row, $value);
            $col++;
        }
        $row++;
    }
    
    // Auto-size columns
    foreach (range("A", $sheet->getHighestColumn()) as $col) {
        $sheet->getColumnDimension($col)->setAutoSize(true);
    }
    
    // Output
    header("Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
    header("Content-Disposition: attachment; filename=\"{$filename}\"");
    
    $writer = new \PhpOffice\PhpSpreadsheet\Writer\Xlsx($spreadsheet);
    $writer->save("php://output");
    exit;
}
```

### JSON Export

```php
function exportToJSON(array $data, string $type): array
{
    $filename = $type . "_export_" . date("Y-m-d_His") . ".json";
    
    header("Content-Type: application/json");
    header("Content-Disposition: attachment; filename=\"{$filename}\"");
    
    echo json_encode([
        "export_type" => $type,
        "exported_at" => date("Y-m-d H:i:s"),
        "record_count" => count($data),
        "data" => $data
    ], JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
    
    exit;
}
```

## Export Template

```smarty
<!-- /admin/templates/export_data.tpl -->
<div class="export-container">
    <h2>Export Data</h2>
    
    <form method="get" action="export.php" class="export-form">
        <input type="hidden" name="token" value="{$token}">
        
        <div class="form-group">
            <label for="export_type">Export Type</label>
            <select name="type" id="export_type" class="form-control" required>
                <option value="">Select type...</option>
                <option value="clients">Clients</option>
                <option value="services">Services</option>
                <option value="invoices">Invoices</option>
                <option value="orders">Orders</option>
                <option value="transactions">Transactions</option>
            </select>
        </div>
        
        <div class="form-group">
            <label for="format">Export Format</label>
            <select name="format" id="format" class="form-control" required>
                <option value="csv">CSV (Comma Separated)</option>
                <option value="xlsx">Excel (XLSX)</option>
                <option value="json">JSON</option>
            </select>
        </div>
        
        <div class="form-group">
            <label>Filters</label>
            
            <div class="filter-row">
                <input type="date" name="date_from" class="form-control" 
                       placeholder="Date From">
                <input type="date" name="date_to" class="form-control" 
                       placeholder="Date To">
            </div>
            
            <select name="status" class="form-control">
                <option value="">All Statuses</option>
                <option value="Active">Active</option>
                <option value="Inactive">Inactive</option>
                <option value="Closed">Closed</option>
            </select>
        </div>
        
        <div class="form-group">
            <label>
                <input type="checkbox" name="include_custom_fields" value="1" checked>
                Include custom fields
            </label>
        </div>
        
        <div class="form-group">
            <label>
                <input type="checkbox" name="compress" value="1">
                Compress export (ZIP)
            </label>
        </div>
        
        <button type="submit" class="btn btn-primary">
            <i class="fa fa-download"></i> Export
        </button>
    </form>
    
    <div class="export-info">
        <h4>Export Information</h4>
        <ul>
            <li>Maximum records per export: 100,000</li>
            <li>Large exports may take several minutes</li>
            <li>You will be notified when export is ready</li>
        </ul>
    </div>
</div>
```

## Best Practices

1. **Large Exports**: Process in batches for large datasets
2. **Compression**: Offer ZIP compression for large files
3. **Async Export**: Queue large exports for background processing
4. **Email Notification**: Notify when export is ready
5. **BOM**: Include UTF-8 BOM for Excel compatibility
6. **Security**: Require authentication for exports
7. **Retention**: Auto-delete exported files after download
8. **Progress**: Show progress for large exports
