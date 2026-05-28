# WHMCS Client Migration Module

Handles client data migration between WHMCS installations.

## Features

- Export client data (services, domains, invoices, tickets, contacts)
- Import with automatic merging
- Progress tracking
- Error logging
- Batch processing

## Installation

Copy module to `/path/to/whmcs/modules/servers/clientmigration/` and activate.

## Usage

```php
// Export single client
$export = clientmigration_ExportClient($clientId, array(
    'include_services' => true,
    'include_domains' => true,
    'include_invoices' => true
));

// Export multiple clients
$exports = clientmigration_ExportMultiple($clientIds, $options);

// Prepare import
$result = clientmigration_PrepareImport($export['data']);

// Process import
$result = clientmigration_ProcessImport($importKey, array('merge_mode' => true));

// Check status
$status = clientmigration_GetImportStatus($importKey);

// Get logs
$logs = clientmigration_GetLogs($importId);
```

## API Functions

| Function | Description |
|----------|-------------|
| `clientmigration_ExportClient()` | Export single client |
| `clientmigration_ExportMultiple()` | Export multiple clients |
| `clientmigration_PrepareImport()` | Prepare import data |
| `clientmigration_ProcessImport()` | Process import |
| `clientmigration_GetImportStatus()` | Get import status |
| `clientmigration_GetLogs()` | Get migration logs |
