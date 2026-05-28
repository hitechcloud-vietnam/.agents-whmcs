# WHMCS Metrics Collector Module

Metrics collection and dashboards with real-time monitoring and alerting.

## Features

- Multiple metric types (gauge, counter, histogram)
- Time series data storage
- Statistical aggregation
- Custom dashboards
- Alerting system
- Health checks
- Data retention policies
- System and WHMCS metrics

## Installation

1. Copy `metricscollector.php` to `/path/to/whmcs/modules/addons/metricscollector/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure collection settings

## Usage

```php
// Record metrics
metricscollector_RecordMetric('my_metric', 42.5, 'gauge', 'items');
metricscollector_Counter('api_requests', 1);
metricscollector_Gauge('active_users', 150, 'users');
metricscollector_Histogram('request_duration', 245.5, 'ms');
metricscollector_Timing('db_query', 125, array('query' => 'users_select'));

// With tags
metricscollector_Counter('page_views', 1, array('page' => '/home', 'source' => 'direct'));

// Get metric data
$metric = metricscollector_GetMetric('my_metric', '2026-05-27', '2026-05-28');
$stats = metricscollector_GetMetricStats('my_metric', '2026-05-27', '2026-05-28');
// Returns: count, sum, min, max, avg

// Get latest value
$latest = metricscollector_GetLatestValue('my_metric');
// Returns: value, timestamp, unit

// List metrics
$metrics = metricscollector_GetMetricsList();

// Get time series
$series = metricscollector_GetTimeSeries('my_metric', null, null, 100);

// Dashboards
$dashboard = metricscollector_GetDashboard('main');
$dashboards = metricscollector_GetDashboards($userId);

metricscollector_SaveDashboard(array(
    'dashboard_key' => 'main',
    'dashboard_name' => 'Main Dashboard',
    'widgets' => array(
        array('type' => 'chart', 'metric' => 'api_requests', 'title' => 'API Requests'),
        array('type' => 'gauge', 'metric' => 'active_users', 'title' => 'Active Users'),
        array('type' => 'stat', 'metric' => 'invoices_unpaid', 'title' => 'Unpaid Invoices')
    ),
    'is_public' => true
));

// Alerts
metricscollector_CreateAlert(array(
    'alert_name' => 'High CPU Load',
    'metric_name' => 'system.load.1min',
    'condition' => '>',
    'threshold' => 5.0,
    'duration_seconds' => 300,
    'severity' => 'critical',
    'action' => 'email',
    'action_config' => array('to' => 'admin@example.com')
));

// Create webhook alert
metricscollector_CreateAlert(array(
    'alert_name' => 'API Errors Spike',
    'metric_name' => 'api.errors',
    'condition' => '>',
    'threshold' => 100,
    'action' => 'webhook',
    'action_config' => array('url' => 'https://hooks.example.com/alerts')
));

// Check alerts
$triggered = metricscollector_CheckAlerts();

// Get alerts
$alerts = metricscollector_GetAlerts(true);

// Health checks
metricscollector_RecordHealthCheck('database', 'healthy', 'All connections OK');
metricscollector_RecordHealthCheck('api', 'degraded', 'Slow responses', array('avg_ms' => 500));

$health = metricscollector_GetHealthStatus();
// Returns: checks, issues, overall_status

// Collect metrics
metricscollector_CollectSystemMetrics();
metricscollector_CollectWHMCSMetrics();

// Cleanup
$deleted = metricscollector_CleanupOldData(90);

// Get summary
$summary = metricscollector_GetSummary(24);
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| CollectionInterval | dropdown | 300 | Interval (seconds) |
| RetentionDays | text | 90 | Data retention |
| EnableRealTime | yesno | yes | Real-time metrics |
| EnablePerformance | yesno | yes | Performance metrics |
| EnableBusiness | yesno | yes | Business metrics |
| DashboardRefresh | text | 30 | Refresh (seconds) |

## Metric Types

| Type | Unit | Description |
|------|------|-------------|
| gauge | varies | Current value |
| counter | count | Incrementing |
| histogram | varies | Distribution |

## Alert Conditions

| Condition | Description |
|-----------|-------------|
| > | Greater than |
| < | Less than |
| >= | Greater or equal |
| <= | Less or equal |
| == | Equal |
| != | Not equal |

## Alert Actions

| Action | Description |
|--------|-------------|
| email | Send email |
| webhook | HTTP webhook |
| log | Log to activity |

## Database Tables

- `mod_metricscollector_data` - Raw metrics
- `mod_metricscollector_aggregates` - Aggregated data
- `mod_metricscollector_dashboards` - Dashboard configs
- `mod_metricscollector_alerts` - Alert definitions
- `mod_metricscollector_health` - Health checks
