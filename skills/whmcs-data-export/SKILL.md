# WHMCS Data Export Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building data export functionality in WHMCS modules.

## When to Use

- Report generation
- Data backup modules
- GDPR compliance

## Export Patterns

### CSV Export
```php
function exportToCsv(string $query, string $filename): void {
    $data = Capsule::select($query);

    header('Content-Type: text/csv');
    header('Content-Disposition: attachment; filename="' . $filename . '.csv"');

    $output = fopen('php://output', 'w');

    // Headers
    fputcsv($output, array_keys((array) $data[0] ?? []));

    // Data
    foreach ($data as $row) {
        fputcsv($output, (array) $row);
    }

    fclose($output);
}
```

### Excel Export
```php
function exportToExcel(array $data, string $filename): void {
    $spreadsheet = new \PhpOffice\PhpSpreadsheet\Spreadsheet();
    $sheet = $spreadsheet->getActiveSheet();

    // Header row
    $sheet->fromArray(array_keys($data[0] ?? []), null, 'A1');

    // Data rows
    $sheet->fromArray($data, null, 'A2');

    // Download
    $writer = new \PhpOffice\PhpSpreadsheet\Writer\Xlsx($spreadsheet);
    header('Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet');
    header('Content-Disposition: attachment; filename="' . $filename . '.xlsx"');
    $writer->save('php://output');
}
```

### JSON Export
```php
function exportToJson(array $data, string $filename): void {
    header('Content-Type: application/json');
    header('Content-Disposition: attachment; filename="' . $filename . '.json"');
    echo json_encode($data, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
}
```

---

**Related Skills:**
- whmcs-reporting
- whmcs-gdpr-compliance
