# WHMCS Report Builder Module

Custom report builder with drag-drop interface and flexible data sources.

## Features

- Multiple data sources (invoices, clients, orders, services, tickets)
- Custom column selection
- Filter builder (date ranges, statuses, user groups)
- Grouping and aggregation
- Sort and order options
- Chart generation (bar, line, pie)
- Scheduled reports (daily, weekly, monthly)
- Report templates
- Export formats (CSV, Excel, PDF, JSON)
- Report sharing
- Dashboard widgets
- Custom calculations
- Drill-down reports

## Installation

1. Copy module to `/path/to/whmcs/modules/addons/reportbuilder/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Start building reports

## Usage

```php
// Create new report
$result = reportbuilder_CreateReport(array(
    'report_name' => 'Monthly Revenue Report',
    'report_key' => 'monthly_revenue',
    'data_source' => 'invoices',
    'columns' => array('invoice_number', 'date', 'client_name', 'total'),
    'filters' => array(
        array('field' => 'status', 'operator' => '==', 'value' => 'Paid'),
        array('field' => 'date', 'operator' => '>=', 'value' => '2026-01-01')
    ),
    'group_by' => 'month',
    'aggregations' => array('total' => 'sum'),
    'sort_order' => 'date',
    'sort_direction' => 'desc'
));

// Get report data
$data = reportbuilder_GetReportData($reportId, $filters);

// Generate quick report
$quick = reportbuilder_QuickReport('clients', array(
    'columns' => array('firstname', 'lastname', 'email'),
    'limit' => 100
));

// Create filter
reportbuilder_CreateFilter(array(
    'filter_name' => 'Active Clients',
    'data_source' => 'clients',
    'conditions' => array(
        array('field' => 'status', 'operator' => '==', 'value' => 'Active')
    )
));

// Get saved filters
$filters = reportbuilder_GetFilters('clients');

// Apply saved filter
$result = reportbuilder_ApplyFilter($filterId, $reportId);

// Create aggregation
reportbuilder_CreateAggregation(array(
    'name' => 'Total Revenue',
    'data_source' => 'invoices',
    'field' => 'total',
    'function' => 'sum',
    'label' => 'revenue_sum'
));

// Generate chart
$chart = reportbuilder_GenerateChart($reportId, array(
    'chart_type' => 'bar',
    'x_axis' => 'month',
    'y_axis' => 'revenue_sum',
    'title' => 'Revenue by Month'
));

// Export report
$export = reportbuilder_ExportReport($reportId, 'csv', $filters);
// Returns: file_path, file_data, file_name, content_type

// Schedule report
reportbuilder_ScheduleReport(array(
    'report_id' => $reportId,
    'schedule' => 'daily',
    'time' => '09:00',
    'recipients' => array('admin@example.com'),
    'format' => 'pdf',
    'filters' => $defaultFilters
));

// Get report run history
$history = reportbuilder_GetReportHistory($reportId, 30);

// Share report
reportbuilder_ShareReport($reportId, $userId, 'view');

// Create dashboard widget
$result = reportbuilder_CreateWidget(array(
 widget_name' => 'New Clients Today',wih
    'report_id' => $reportId,
    'chart_type' => 'counter',
    'refresh_interval' => 300
));

// Get dashboard widgets
$widgets = reportbuilder_GetWidgets();

// Add custom calculation
reportbuilder_AddCalculation(array(
    'name' => 'Profit Margin',
    'expression' => '(revenue - cost) / revenue * 100',
    'data_type' => 'percent'
));
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| DefaultExportFormat | dropdown | csv | Default export format |
| MaxRowsPerReport | text | 10000 | Maximum rows |
| EnableScheduling | yesno | yes | Enable scheduled reports |
| EnableCharts | yesno | yes | Enable chart generation |
| ChartColors | text | #3498db,#e74c3c,#2ecc71 | Chart colors |
| EnableSharing | yesno | yes | Enable report sharing |
| DefaultDateRange | dropdown | last_30_days | Default date range |
| EnableCaching | yesno | yes | Cache report results |

## Data Sources

| Source | Description | Available Fields |
|--------|-------------|------------------|
| clients | Client information | id, firstname, lastname, email, company, status, datecreated, country |
| invoices | Invoice data | id, invoicenum, userid, date, duedate, total, subtotal, tax, status |
| orders | Order data | id, ordernum, userid, date, amount, status, paymentmethod |
| hosting | Services/Products | id, userid, domain, regdate, nextduedate, amount, status |
| ticket | Support tickets | id, tid, userid, date, subject, status, priority |
| affiliates | Affiliate data | id, userid, date, revenue, commission, withdrawals |
| quotes | Quote data | id, subject, userid, date, total, status |
| activity | Activity log | id, userid, date, description, action, details |

## Filter Operators

| Operator | Description | Example |
|-----------|-------------|---------|
| == | Equal | status == 'Active' |
| != | Not equal | status != 'Closed' |
| > | Greater than | total > 100 |
| < | Less than | total < 1000 |
| >= | Greater or equal | date >= '2026-01-01' |
| <= | Less or equal | date <= '2026-12-31' |
| contains | Contains substring | email contains '@gmail' |
| starts_with | Starts with | name starts_with 'John' |
| ends_with | Ends with | email ends_with '.com' |
| in | In list | status in ['Active','Pending'] |
| between | Between values | total between 100 and 500 |
| is_null | Is null | custom_field is_null |
| is_not_null | Is not null | custom_field is_not_null |

## Aggregation Functions

| Function | Description |
|----------|-------------|
| sum | Sum of values |
| avg | Average value |
| count | Count of records |
| min | Minimum value |
| max | Maximum value |
| count_distinct | Count distinct values |

## Chart Types

| Type | Description |
|------|-------------|
| bar | Bar chart |
| line | Line chart |
| pie | Pie chart |
| doughnut | Doughnut chart |
| counter | Single value display |
| table | Data table view |

## Database Tables

- `mod_reportbuilder_reports` - Report definitions
- `mod_reportbuilder_filters` - Saved filters
- `mod_reportbuilder_schedules` - Scheduled reports
- `mod_reportbuilder_history` - Report run history
- `mod_reportbuilder_charts` - Chart configurations
- `mod_reportbuilder_widgets` - Dashboard widgets
- `mod_reportbuilder_aggregations` - Custom aggregations
- `mod_reportbuilder_sharing` - Shared reports

## API Functions

| Function | Description |
|----------|-------------|
| `reportbuilder_CreateReport()` | Create new report |
| `reportbuilder_GetReportData()` | Get report results |
| `reportbuilder_QuickReport()` | Quick report generation |
| `reportbuilder_CreateFilter()` | Create saved filter |
| `reportbuilder_ApplyFilter()` | Apply filter to report |
| `reportbuilder_GenerateChart()` | Generate chart |
| `reportbuilder_ExportReport()` | Export to format |
| `reportbuilder_ScheduleReport()` | Schedule report |
| `reportbuilder_ShareReport()` | Share with users |
| `reportbuilder_CreateWidget()` | Create widget |
| `reportbuilder_GetWidgets()` | Get dashboard widgets |
| `reportbuilder_GetReportHistory()` | Get run history |
| `reportbuilder_AddCalculation()` | Add custom calculation |
