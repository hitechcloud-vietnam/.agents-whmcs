# WHMCS Data Export Module

Data export module with multiple export formats and scheduling.

## Features

- Multiple data sources (clients, invoices, orders, services, tickets)
- Export formats (CSV, Excel, JSON, XML, PDF)
- Scheduled exports (daily, weekly, monthly)
- Email delivery
- FTP/SFTP upload
- AWS S3 export
- Filter-based exports
- Template-based exports
- Encryption options
- Compression (ZIP, GZIP)
- Partial exports
- Incremental exports
- Export history
- Custom field exports
- Multi-language export

## Installation

1. Copy module to `/path/to/whmcs/modules/addons/dataexport/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure export settings

## Usage

```php
// Create export job
$result = dataexport_CreateExport(array(
    'export_name' => 'Monthly Clients Export',
    'export_key' => 'monthly_clients',
    'data_source' => 'clients',
    'format' => 'csv',
    'filters' => array(
        array('field' => 'status', 'operator' => '==', 'value' => 'Active')
    ),
    'columns' => array('id', 'firstname', 'lastname', 'email', 'company', 'datecreated')
));

// Run export immediately
$result = dataexport_RunExport($exportId);
// Returns: file_path, row_count, file_size

// Schedule export
dataexport_ScheduleExport(array(
    'export_id' => $exportId,
    'schedule' => 'monthly',
    'time' => '02:00',
    'recipients' => array('admin@example.com'),
    'keep_files' => 12
));

// Export to streaming (for large datasets)
$result = dataexport_StreamExport($exportId, function($row) {
    // Process each row as it's exported
    fputcsv(STDOUT, (array)$row);
});

// Export with encryption
$dataexport_EncryptedExport($exportId, array(
    'password' => 'secure_password',
    'algorithm' => 'AES-256-CBC'
));

// Upload to FTP
dataexport_UploadToFTP($exportId, array(
    'host' => 'ftp.example.com',
    'username' => 'user',
    'password' => 'pass',
    'path' => '/exports/'
));

// Upload to S3
dataexport_UploadToS3($exportId, array(
    'bucket' => 'my-bucket',
    'region' => 'us-east-1',
    'key' => 'my-key',
    'secret' => 'my-secret',
    'path' => 'exports/'
));

// Get export history
$history = dataexport_GetHistory($exportId, 30);

// Get export statistics
$stats = dataexport_GetStatistics($exportId);
// Returns: total_runs, total_rows, avg_duration, last_run

// Cancel scheduled export
dataexport_CancelSchedule($scheduleId);

// Delete export
dataexport_DeleteExport($exportId);

// Get available formats
$formats = dataexport_GetFormats();
// Returns: csv, xlsx, json, xml, pdf

// Export multiple sources
dataexport_CreateMultiExport(array(
    'export_name' => 'Full Backup',
    'sources' => array('clients', 'invoices', 'orders'),
    'format' => 'zip',
    'compress' => true
));
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| DefaultFormat | dropdown | csv | Default export format |
| MaxRows | text | 100000 | Maximum rows per export |
| EnableScheduling | yesno | yes | Enable scheduled exports |
| FTPHost | text | - | FTP host |
| FTPUsername | text | - | FTP username |
| FTPPassword | password | - | FTP password |
| FTPPath | text | /exports/ | FTP base path |
| S3Bucket | text | - | AWS S3 bucket |
| S3Region | text | us-east-1 | AWS region |
| EncryptionKey | password | - | Export encryption key |
| CompressExports | yesno | yes | Compress exports |
| RetentionDays | text | 30 | File retention (days) |

## Data Sources

- clients - Client information
- invoices - Invoice data
- orders - Order data
- hosting - Services/Products
- ticket - Support tickets
- addons - Addon modules
- affiliates - Affiliate data
- quotes - Quote data
- activity - Activity log
- custom - Custom queries

## Export Formats

| Format | Extension | Content-Type |
|--------|-----------|--------------|
| CSV | .csv | text/csv |
| Excel | .xlsx | application/vnd.openxmlformats-officedocument.spreadsheetml.sheet |
| JSON | .json | application/json |
| XML | .xml | text/xml |
| PDF | .pdf | application/pdf |

## Database Tables

- `mod_dataexport_definitions` - Export definitions
- `mod_dataexport_schedules` - Scheduled exports
- `mod_dataexport_history` - Export history
- `mod_dataexport_destinations` - Export destinations

## API Functions

| Function | Description |
|----------|-------------|
| `dataexport_CreateExport()` | Create export definition |
| `dataexport_RunExport()` | Run export immediately |
| `dataexport_StreamExport()` | Stream export for large datasets |
| `dataexport_GetAllExports()` | List all exports |
| `dataexport_GetExport()` | Get export details |
| `dataexport_UpdateExport()` | Update export settings |
| `dataexport_DeleteExport()` | Delete export |
| `dataexport_ScheduleExport()` | Schedule export |
| `dataexport_CancelSchedule()` | Cancel scheduled export |
| `dataexport_UploadToFTP()` | Upload to FTP |
| `dataexport_UploadToS3()` | Upload to AWS S3 |
| `dataexport_GetHistory()` | Get export history |
| `dataexport_GetStatistics()` | Get export statistics |
| `dataexport_EncryptedExport()` | Export with encryption |
