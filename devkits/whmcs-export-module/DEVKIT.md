# WHMCS Export Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-export-module/
├── export.php             # Export controller
├── lib/
│   ├── ExportBuilder.php   # Export builder
│   ├── Formatters/
│   │   ├── CsvFormatter.php
│   │   ├── JsonFormatter.php
│   │   └── XmlFormatter.php
│   └── QueryBuilder.php    # Custom query builder
└── templates/
    └── admin.tpl           # Admin interface
```

## Export Controller Template

```php
<?php
/**
 * WHMCS Export Module: {Module}
 * DevKit Template
 * 
 * Installation: Upload to modules/addons/{module}/
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * Config function
 */
function {module}_config(): array {
    return [
        'name' => '{Export Module}',
        'description' => 'Export WHMCS data to various formats',
        'version' => '1.0',
        'author' => '{Author}',
    ];
}

/**
 * Output function
 */
function {module}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'list';
    
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
    }
    
    switch ($action) {
        case 'export':
            {module}_executeExport();
            break;
        case 'preview':
            {module}_showPreview();
            break;
        case 'schedule':
            {module}_scheduleExport();
            break;
        case 'download':
            {module}_downloadExport();
            break;
        default:
            {module}_showDashboard();
    }
}

/**
 * Show Dashboard
 */
function {module}_showDashboard(): void {
    echo <<<'HTML'
<div class="export-module">
    <div class="row">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Data Export</h3>
                </div>
                <div class="panel-body">
                    <p>Export WHMCS data to CSV, JSON, XML, or Excel formats.</p>
                </div>
            </div>
        </div>
    </div>
    
    <div class="row">
        <div class="col-md-4">
            <div class="panel panel-primary">
                <div class="panel-heading">
                    <h3 class="panel-title">Export Types</h3>
                </div>
                <div class="list-group">
                    <a href="#" class="list-group-item export-type" data-type="clients">
                        <i class="fa fa-users"></i> Clients
                    </a>
                    <a href="#" class="list-group-item export-type" data-type="services">
                        <i class="fa fa-server"></i> Services
                    </a>
                    <a href="#" class="list-group-item export-type" data-type="invoices">
                        <i class="fa fa-file-invoice-dollar"></i> Invoices
                    </a>
                    <a href="#" class="list-group-item export-type" data-type="orders">
                        <i class="fa fa-shopping-cart"></i> Orders
                    </a>
                    <a href="#" class="list-group-item export-type" data-type="tickets">
                        <i class="fa fa-ticket-alt"></i> Tickets
                    </a>
                    <a href="#" class="list-group-item export-type" data-type="domains">
                        <i class="fa fa-globe"></i> Domains
                    </a>
                    <a href="#" class="list-group-item export-type" data-type="products">
                        <i class="fa fa-box"></i> Products
                    </a>
                    <a href="#" class="list-group-item export-type" data-type="custom">
                        <i class="fa fa-database"></i> Custom Query
                    </a>
                </div>
            </div>
        </div>
        
        <div class="col-md-8">
            <div class="panel panel-default export-form" style="display:none;">
                <div class="panel-heading">
                    <h3 class="panel-title" id="export-title">Export Clients</h3>
                </div>
                <div class="panel-body">
                    <form method="post" action="?module={module}&action=export">
                        <input type="hidden" name="export_type" id="export_type" value="">
                        <input type="hidden" name="csrf_token" value="{$csrf_token}">
                        
                        <div class="form-group">
                            <label for="format">Export Format</label>
                            <select name="format" id="format" class="form-control">
                                <option value="csv">CSV (Comma Separated)</option>
                                <option value="json">JSON</option>
                                <option value="xml">XML</option>
                            </select>
                        </div>
                        
                        <div class="form-group">
                            <label>Date Range (Optional)</label>
                            <div class="row">
                                <div class="col-md-6">
                                    <input type="date" name="date_from" class="form-control" 
                                           placeholder="From date">
                                </div>
                                <div class="col-md-6">
                                    <input type="date" name="date_to" class="form-control" 
                                           placeholder="To date">
                                </div>
                            </div>
                        </div>
                        
                        <div class="form-group">
                            <label for="fields">Fields to Export</label>
                            <div class="row">
                                <div class="col-md-12 field-selector" id="field-selector">
                                    <!-- Populated dynamically -->
                                </div>
                            </div>
                        </div>
                        
                        <div class="form-group">
                            <label>
                                <input type="checkbox" name="include_headers" value="1" checked>
                                Include headers (CSV)
                            </label>
                        </div>
                        
                        <div class="form-group">
                            <label>
                                <input type="checkbox" name="compress" value="1">
                                Compress to ZIP
                            </label>
                        </div>
                        
                        <div class="form-group">
                            <a href="?module={module}" class="btn btn-default">Cancel</a>
                            <button type="submit" class="btn btn-primary">
                                <i class="fa fa-download"></i> Export
                            </button>
                            <button type="button" class="btn btn-info" id="preview-btn">
                                <i class="fa fa-eye"></i> Preview
                            </button>
                        </div>
                    </form>
                </div>
            </div>
        </div>
    </div>
</div>

<script>
$(document).ready(function() {
    $('.export-type').on('click', function(e) {
        e.preventDefault();
        var type = $(this).data('type');
        var title = $(this).text();
        
        $('#export-type').val(type);
        $('#export-title').text('Export ' + title);
        $('.export-form').show();
        
        loadFieldSelector(type);
    });
    
    function loadFieldSelector(type) {
        var fields = {
            clients: ['id', 'firstname', 'lastname', 'email', 'companyname', 'country', 'datecreated'],
            services: ['id', 'userid', 'domain', 'packageid', 'regdate', 'nextduedate', 'domainstatus', 'billingcycle'],
            invoices: ['id', 'userid', 'invoicenum', 'date', 'duedate', 'total', 'status'],
            orders: ['id', 'userid', 'date', 'amount', 'status', 'paymentmethod'],
            tickets: ['id', 'userid', 'tid', 'subject', 'status', 'priority', 'created'],
            domains: ['id', 'userid', 'domain', 'registrationdate', 'nextduedate', 'domainstatus', 'registrar'],
            products: ['id', 'name', 'type', 'description', 'paytype', 'monthly', 'quarterly', 'annually']
        };
        
        var selectedFields = fields[type] || fields.clients;
        var html = '';
        
        selectedFields.forEach(function(field) {
            html += '<div class="col-md-4">' +
                    '<label class="checkbox-inline">' +
                    '<input type="checkbox" name="fields[]" value="' + field + '" checked> ' + field +
                    '</label></div>';
        });
        
        $('#field-selector').html(html);
    }
});
</script>
HTML;
}

/**
 * Execute Export
 */
function {module}_executeExport(): void {
    $type = $_POST['export_type'] ?? 'clients';
    $format = $_POST['format'] ?? 'csv';
    $fields = $_POST['fields'] ?? [];
    $dateFrom = $_POST['date_from'] ?? null;
    $dateTo = $_POST['date_to'] ?? null;
    $compress = !empty($_POST['compress']);
    
    if (empty($fields)) {
        $fields = {module}_getDefaultFields($type);
    }
    
    $builder = new \{Module}\ExportBuilder();
    $data = $builder->build($type, $fields, $dateFrom, $dateTo);
    
    $formatter = {module}_getFormatter($format);
    $content = $formatter->format($data, $fields);
    
    $filename = "export_{$type}_" . date('Y-m-d_His');
    
    if ($compress) {
        $zip = new ZipArchive();
        $zipFile = sys_get_temp_dir() . "/{$filename}.zip";
        
        if ($zip->open($zipFile, ZipArchive::CREATE) === true) {
            $zip->addFromString("{$filename}.{$format}", $content);
            $zip->close();
        }
        
        header('Content-Type: application/zip');
        header('Content-Disposition: attachment; filename="' . $filename . '.zip"');
        header('Content-Length: ' . filesize($zipFile));
        readfile($zipFile);
        unlink($zipFile);
    } else {
        $mimeTypes = [
            'csv' => 'text/csv',
            'json' => 'application/json',
            'xml' => 'application/xml',
        ];
        
        header('Content-Type: ' . $mimeTypes[$format]);
        header('Content-Disposition: attachment; filename="' . $filename . '.' . $format . '"');
        echo $content;
    }
    
    logActivity("{Module}: Exported {$type} data as {$format}");
    exit;
}

/**
 * Get Default Fields
 */
function {module}_getDefaultFields(string $type): array {
    $defaults = [
        'clients' => ['id', 'firstname', 'lastname', 'email', 'companyname', 'country', 'datecreated'],
        'services' => ['id', 'userid', 'domain', 'regdate', 'nextduedate', 'domainstatus'],
        'invoices' => ['id', 'userid', 'invoicenum', 'date', 'total', 'status'],
    ];
    
    return $defaults[$type] ?? ['id', 'name'];
}

/**
 * Get Formatter
 */
function {module}_getFormatter(string $format): \{Module}\Formatter {
    $class = '\\{Module}\\' . ucfirst($format) . 'Formatter';
    
    if (!class_exists($class)) {
        throw new \Exception("Formatter not found: {$format}");
    }
    
    return new $class();
}
```

## Export Builder Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class ExportBuilder {
    
    private array $typeMaps = [
        'clients' => 'tblclients',
        'services' => 'tblhosting',
        'invoices' => 'tblinvoices',
        'orders' => 'tblorders',
        'tickets' => 'tbltickets',
        'domains' => 'tbldomains',
        'products' => 'tblproducts',
    ];
    
    public function build(string $type, array $fields, ?string $dateFrom = null, ?string $dateTo = null): array {
        $table = $this->typeMaps[$type] ?? null;
        
        if (!$table) {
            throw new \Exception("Unknown export type: {$type}");
        }
        
        $query = Capsule::table($table);
        
        // Apply date filters
        $dateField = $this->getDateField($type);
        if ($dateField && $dateFrom) {
            $query->where($dateField, '>=', $dateFrom);
        }
        if ($dateField && $dateTo) {
            $query->where($dateField, '<=', $dateTo . ' 23:59:59');
        }
        
        // Join with related tables if needed
        if ($type === 'services') {
            $query->select([
                'tblhosting.*',
                'tblclients.firstname',
                'tblclients.lastname',
                'tblclients.email',
                'tblproducts.name as product_name',
            ])
            ->leftJoin('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
            ->leftJoin('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id');
        }
        
        if ($type === 'invoices') {
            $query->select([
                'tblinvoices.*',
                'tblclients.firstname',
                'tblclients.lastname',
                'tblclients.email',
            ])
            ->leftJoin('tblclients', 'tblinvoices.userid', '=', 'tblclients.id');
        }
        
        return $query->get()->toArray();
    }
    
    private function getDateField(string $type): ?string {
        $dateFields = [
            'clients' => 'created_at',
            'services' => 'regdate',
            'invoices' => 'date',
            'orders' => 'date',
            'tickets' => 'created',
            'domains' => 'registrationdate',
        ];
        
        return $dateFields[$type] ?? null;
    }
    
    public function buildCustomQuery(string $sql): array {
        $results = Capsule::select($sql);
        
        // Convert to array of objects
        $data = [];
        foreach ($results as $row) {
            $data[] = (object) $row;
        }
        
        return $data;
    }
    
    public function getTableSchema(string $table): array {
        $columns = Capsule::select("SHOW COLUMNS FROM {$table}");
        
        return array_map(function($col) {
            return [
                'field' => $col->Field,
                'type' => $col->Type,
                'null' => $col->Null,
                'key' => $col->Key,
                'default' => $col->Default,
            ];
        }, $columns);
    }
}
```

## Formatters

### CSV Formatter

```php
<?php
namespace {Module};

class CsvFormatter {
    
    private string $delimiter = ',';
    private string $enclosure = '"';
    private bool $includeHeaders = true;
    
    public function format(array $data, array $fields): string {
        $output = fopen('php://memory', 'r+');
        
        // Write headers
        if ($this->includeHeaders) {
            fputcsv($output, $fields, $this->delimiter, $this->enclosure);
        }
        
        // Write data
        foreach ($data as $row) {
            $rowData = [];
            foreach ($fields as $field) {
                $rowData[] = $this->formatValue($row->{$field} ?? '');
            }
            fputcsv($output, $rowData, $this->delimiter, $this->enclosure);
        }
        
        rewind($output);
        $content = stream_get_contents($output);
        fclose($output);
        
        return $content;
    }
    
    private function formatValue($value): string {
        if (is_null($value)) {
            return '';
        }
        
        if ($value instanceof \DateTime) {
            return $value->format('Y-m-d H:i:s');
        }
        
        return (string) $value;
    }
    
    public function setDelimiter(string $delimiter): void {
        $this->delimiter = $delimiter;
    }
    
    public function setIncludeHeaders(bool $include): void {
        $this->includeHeaders = $include;
    }
}
```

### JSON Formatter

```php
<?php
namespace {Module};

class JsonFormatter {
    
    private int $options = JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE;
    
    public function format(array $data, array $fields): string {
        $output = [];
        
        foreach ($data as $row) {
            $item = [];
            foreach ($fields as $field) {
                $item[$field] = $this->formatValue($row->{$field} ?? null);
            }
            $output[] = $item;
        }
        
        return json_encode($output, $this->options);
    }
    
    private function formatValue($value) {
        if (is_null($value)) {
            return null;
        }
        
        if ($value instanceof \DateTime) {
            return $value->format('Y-m-d H:i:s');
        }
        
        return $value;
    }
    
    public function setOptions(int $options): void {
        $this->options = $options;
    }
}
```

### XML Formatter

```php
<?php
namespace {Module};

class XmlFormatter {
    
    public function format(array $data, array $fields): string {
        $xml = new \SimpleXMLElement('<?xml version="1.0" encoding="UTF-8"?><export></export>');
        
        foreach ($data as $row) {
            $item = $xml->addChild('item');
            
            foreach ($fields as $field) {
                $value = $this->formatValue($row->{$field} ?? null);
                $item->addChild($field, htmlspecialchars((string) $value));
            }
        }
        
        return $xml->asXML();
    }
    
    private function formatValue($value): string {
        if (is_null($value)) {
            return '';
        }
        
        if ($value instanceof \DateTime) {
            return $value->format('Y-m-d H:i:s');
        }
        
        return (string) $value;
    }
}
```

## Checklist

```
Pre-Dev:
□ Define export types (clients, services, invoices, etc.)
□ Identify all available fields for each type
□ Plan output formats (CSV, JSON, XML, Excel)
□ Design filtering options
□ Plan large dataset handling

Development:
□ Create module structure
□ Implement ExportBuilder class
□ Add table mapping
□ Create CsvFormatter class
□ Create JsonFormatter class
□ Create XmlFormatter class
□ Add date range filtering
□ Add field selection
□ Create admin template
□ Add ZIP compression
□ Add download handling
□ Create progress tracking

Testing:
□ Test CSV export
□ Test JSON export
□ Test XML export
□ Verify date filtering
□ Test field selection
□ Verify encoding
□ Test with large datasets
□ Test ZIP compression
□ Verify file downloads
□ Test with special characters
```