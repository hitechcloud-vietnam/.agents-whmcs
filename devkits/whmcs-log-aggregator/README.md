# WHMCS Log Aggregator Module

Centralized logging system with multiple log sources, filtering, and export capabilities.

## Features

- Multiple log levels (debug to emergency)
- File and database logging
- Syslog forwarding
- Log filtering and search
- Statistics and analytics
- Log archiving
- Export to CSV/JSON
- Alert rules (email, webhook, Slack)
- Request tracking

## Installation

1. Copy `logaggregator.php` to `/path/to/whmcs/modules/addons/logaggregator/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure logging settings

## Usage

```php
// Basic logging
logaggregator_Info('User logged in', 'auth', array('user_id' => 123));
logaggregator_Warning('Payment failed', 'payment', array('invoice_id' => 456, 'error' => 'Declined'));
logaggregator_Error('API request failed', 'api', array('endpoint' => '/api/v1/orders', 'code' => 500));

// With context
logaggregator_Log('error', 'Database connection failed', 'core', array(
    'host' => 'db.example.com',
    'port' => 3306,
    'timeout' => 5
));

// Shorthand functions
logaggregator_Debug('Debug message');
logaggregator_Notice('Notice message');
logaggregator_Alert('Alert message');
logaggregator_Emergency('Emergency - system down');

// Query logs
$logs = logaggregator_GetLogs(array(
    'level' => 'error',
    'source' => 'payment',
    'from_date' => '2026-05-01',
    'to_date' => '2026-05-28',
    'search' => 'failed'
), 100);

// Get statistics
$stats = logaggregator_GetLogStats(7);
// Returns: by_level_source, by_day

// Count logs
$count = logaggregator_GetLogCount(array('level' => 'error'));

// Get sources
$sources = logaggregator_GetSources();

// Add custom source
logaggregator_AddSource(array(
    'source_key' => 'custom_module',
    'source_name' => 'Custom Module',
    'description' => 'My custom module logs',
    'min_level' => 'info'
));

// Add alert rule
logaggregator_AddRule(array(
    'rule_name' => 'Payment Failures',
    'match_source' => 'payment',
    'match_level' => 'error',
    'match_pattern' => '/failed/i',
    'action' => 'email',
    'action_value' => 'admin@example.com'
));

// Get rules
$rules = logaggregator_GetRules();

// Archive logs
$result = logaggregator_ArchiveLogs('2026-04-01', '2026-04-30');
// Returns: success, archive_id, filename, logs_archived

// Get archives
$archives = logaggregator_GetArchives();

// Download archive
$archive = logaggregator_DownloadArchive($archiveId);

// Export logs
$export = logaggregator_ExportLogs('csv', array('level' => 'error', 'days' => 7));
// Returns: success, filepath, filename

// Clear old logs
$deleted = logaggregator_ClearLogs('2026-04-01');
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| DefaultLevel | dropdown | info | Default log level |
| EnableFileLog | yesno | yes | Enable file logging |
| EnableDBLog | yesno | yes | Enable DB logging |
| LogPath | text | /storage/logs/custom | Log file path |
| MaxFileSize | text | 104857600 | Max file size (bytes) |
| RetentionDays | text | 30 | Log retention days |
| EnableSyslog | yesno | no | Enable syslog |
| SyslogHost | text | - | Syslog host |
| SyslogPort | text | 514 | Syslog port |

## Log Levels

| Level | Description |
|-------|-------------|
| debug | Debug messages |
| info | Informational |
| notice | Significant events |
| warning | Warnings |
| error | Errors |
| critical | Critical issues |
| alert | Immediate action |
| emergency | System down |

## Default Sources

| Source | Description |
|--------|-------------|
| core | WHMCS Core |
| module | Module activity |
| api | API requests |
| auth | Authentication |
| payment | Payment processing |
| admin | Admin actions |

## Database Tables

- `mod_logaggregator_logs` - Log entries
- `mod_logaggregator_sources` - Log sources
- `mod_logaggregator_archives` - Archived logs
- `mod_logaggregator_rules` - Alert rules
