# WHMCS Report Module DevKit

A comprehensive custom report module for WHMCS with multiple data sources, customizable templates, and export functionality.

## Features

- Multiple report types (Sales, Clients, Services, etc.)
- Customizable date ranges and grouping
- Multiple export formats (HTML, CSV, PDF, Excel)
- Scheduled report generation
- Email report delivery
- Data source abstraction
- Configurable metrics
- Admin interface for report management

## Installation

1. Copy module files to:
   ```
   modules/addons/whmcs_report_module/
   ```

2. Activate the module in WHMCS Admin > Addon Modules

3. Configure reports via the admin interface

## Report Types

### Sales Report

Metrics available:
- Revenue (daily, weekly, monthly)
- Order count
- Average order value
- Top products
- Revenue by payment method

### Client Report

Metrics available:
- New clients (by period)
- Active clients
- Client retention
- Clients by country
- Signup sources

### Service Report

Metrics available:
- Active services
- New signups
- Cancellations
- By product breakdown
- Retention rate

## Usage

### Generate Report

```php
use ReportModule\ReportBuilder;

$report = new ReportBuilder('sales', [
    'period' => 'month',
    'group_by' => 'day',
    'metrics' => ['revenue', 'orders', 'avg_order_value'],
]);

$data = $report->generate();

// Get formatted output
$array = $report->toArray();
$csv = $report->toCsv();
$json = $report->toJson();
```

### Export Report

```php
use ReportModule\ExportHandler;

// Export as CSV
$csv = ExportHandler::toCsv($data);

// Export as HTML
$html = ExportHandler::toHtml($data, 'Sales Report');

// Export as PDF
$pdf = ExportHandler::toPdf($data, 'Sales Report');
```

### Data Sources

```php
use ReportModule\DataSources;

// Get invoice data
$invoices = DataSources::invoices([
    'status' => 'Paid',
    'start_date' => '2024-01-01',
    'end_date' => '2024-01-31',
]);

// Get client data
$clients = DataSources::clients([
    'status' => 'Active',
    'country' => 'US',
]);

// Get service data
$services = DataSources::services([
    'status' => 'Active',
    'product_id' => 1,
]);
```

## Configuration

### Report Settings

| Setting | Description |
|---------|-------------|
| Period | Time period (week, month, quarter, year) |
| Group By | How to group data (day, week, month) |
| Metrics | Which metrics to include |
| Format | Output format (HTML, CSV, PDF, XLSX) |

### Scheduled Reports

Configure automatic report generation:

1. Go to Admin > Report Module > Scheduled Reports
2. Click "Add Scheduled Report"
3. Configure:
   - Report name
   - Report type
   - Schedule (cron expression)
   - Recipients (email addresses)
   - Format (PDF, CSV, etc.)

## Export Formats

### CSV Export

```php
$csv = $report->toCsv();

// Download
header('Content-Type: text/csv');
header('Content-Disposition: attachment; filename="report.csv"');
echo $csv;
```

### PDF Export

```php
$pdf = ExportHandler::toPdf($data, 'Report Title');

// For actual PDF generation, integrate with:
- TCPDF
- DomPDF
- wkhtmltopdf
```

### Excel Export

```php
$excel = ExportHandler::toExcel($data, 'Report Title');

// For actual Excel generation, use PhpSpreadsheet
```

### HTML Export

```php
$html = ExportHandler::toHtml($data, 'Report Title');
```

## Admin Interface

### Report Dashboard

View summary statistics and quick access to reports:
- Total revenue (today, week, month)
- New clients
- Active services
- Quick export buttons

### Report Builder

Create custom reports:
1. Select report type
2. Choose date range
3. Select metrics
4. Choose grouping
5. Preview and export

### Scheduled Reports

Manage automated reports:
- View scheduled reports
- Edit schedules
- View last run status
- Enable/disable reports

## Database Schema

### reports table
- `id` - Report ID
- `report_name` - Display name
- `report_type` - sales, client, service, etc.
- `config` - JSON configuration
- `is_scheduled` - Enable scheduled execution
- `schedule` - Cron expression
- `recipients` - Email addresses (JSON)
- `last_run` - Last execution time

### report_data table
- `id` - Data ID
- `report_id` - Related report
- `data` - JSON report data
- `generated_at` - Generation timestamp

### scheduled_reports table
- `id` - Schedule ID
- `report_name` - Display name
- `report_type` - Report type
- `schedule` - Cron expression
- `config` - Report configuration
- `recipients` - Email addresses
- `format` - Export format
- `is_active` - Enable/disable
- `last_run` - Last execution
- `next_run` - Next scheduled run

## File Structure

```
whmcs-report-module/
├── report-module.php        # Main module
├── lib/
│   ├── ReportBuilder.php     # Report builder
│   ├── DataSources.php       # Data sources
│   └── ExportHandler.php     # Export functionality
├── reports/
│   ├── SalesReport.php       # Sales report
│   ├── ClientReport.php      # Client report
│   └── ServiceReport.php     # Service report
└── templates/
    └── report-config.tpl     # Admin configuration
```

## Hooks

```php
// Run scheduled reports
add_hook('DailyCronJob', 1, function($vars) {
    // Process scheduled reports
});
```

## Requirements

- WHMCS 7.0+
- PHP 7.4+
- For PDF export: TCPDF, DomPDF, or similar
- For Excel export: PhpSpreadsheet

## Support

For issues and feature requests, please contact the developer.