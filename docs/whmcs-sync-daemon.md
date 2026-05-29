# WHMCS Sync Daemon Documentation

## Overview

The WHMCS Sync Daemon ensures domain and registration data stays synchronized between WHMCS and registrars. It runs continuously to detect and reconcile changes.

## Architecture

### Components

```
+----------------+     +----------------+     +----------------+
|  Sync Daemon   | --> |  Queue Manager | --> |  Registrar API |
|  (Background)  |     |  (Jobs)        |     |  (External)    |
+----------------+     +----------------+     +----------------+
       |                       |
       v                       v
+----------------+     +----------------+
|  WHMCS Database| <-- |  Event Handler |
|  (Local)       |     |  (Triggers)    |
+----------------+     +----------------+
```

### Daemon Types

1. **Domain Sync Daemon** - Domain status and expiration sync
2. **Contact Sync Daemon** - WHOIS contact data sync
3. **Nameserver Sync Daemon** - DNS record synchronization
4. **Pricing Sync Daemon** - TLD pricing updates

## Installation

### System Requirements

- WHMCS 8.0 or higher
- PHP 8.1+ with pcntl extension
- MySQL 8.0+ or MariaDB 10.4+
- Minimum 512MB RAM
- Cron access

### Installation Steps

1. **Install daemon files:**
   ```bash
   cd /whmcs/modules/servers/sync-daemon
   composer install
   ```

2. **Create database tables:**
   ```sql
   CREATE TABLE mod_sync_daemon_queue (
       id INT AUTO_INCREMENT PRIMARY KEY,
       domain_id INT NOT NULL,
       action VARCHAR(50) NOT NULL,
       priority INT DEFAULT 0,
       status ENUM('pending', 'processing', 'completed', 'failed') DEFAULT 'pending',
       attempts INT DEFAULT 0,
       last_attempt DATETIME,
       next_attempt DATETIME,
       data JSON,
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
       INDEX idx_status_priority (status, priority),
       INDEX idx_domain_action (domain_id, action)
   ) ENGINE=InnoDB;
   ```

3. **Configure daemon settings:**
   ```php
   // configuration.php
   $syncDaemonConfig = [
       'enabled' => true,
       'workers' => 4,
       'queue_limit' => 100,
       'batch_size' => 25,
       'retry_delay' => 300,
       'max_retries' => 5,
       'log_level' => 'info'
   ];
   ```

## Configuration

### Global Settings

Navigate to **Configuration > System Settings > Sync Daemon**

```php
// Sync Daemon Configuration
$synConfig = [
    // Daemon Behavior
    'daemon_enabled' => true,
    'worker_processes' => 4,
    'queue_polling_interval' => 30,
    'batch_size' => 50,
    'concurrent_registrars' => 3,

    // Sync Intervals
    'domain_sync_interval' => 3600,      // 1 hour
    'expiration_sync_interval' => 7200,  // 2 hours
    'nameserver_sync_interval' => 1800, // 30 minutes
    'pricing_sync_interval' => 86400,    // 24 hours

    // Retry Settings
    'auto_retry' => true,
    'max_retry_attempts' => 5,
    'retry_backoff_multiplier' => 2,
    'initial_retry_delay' => 60,

    // Alerts
    'alert_on_failure' => true,
    'alert_threshold' => 10,
    'alert_email' => 'admin@example.com',

    // Performance
    'cache_enabled' => true,
    'cache_ttl' => 300,
    'connection_timeout' => 30,
    'request_timeout' => 60
];
```

### Registrar-Specific Settings

```php
// Per-registrar sync configuration
$registrarSyncSettings = [
    'enom' => [
        'enabled' => true,
        'rate_limit' => 100,           // requests per minute
        'api_endpoint' => 'https://api.enom.com',
        'sync_all_domains' => true,
        'include_expired' => false
    ],
    'godaddy' => [
        'enabled' => true,
        'rate_limit' => 60,
        'api_endpoint' => 'https://api.godaddy.com',
        'require_auth' => true,
        'pagination_size' => 500
    ]
];
```

## Daemon Commands

### Start Daemon

```bash
# Start daemon in foreground
php /whmcs/sync-daemon/daemon.php start

# Start daemon in background
php /whmcs/sync-daemon/daemon.php start --daemonize

# Start with specific config
php /whmcs/sync-daemon/daemon.php start --config=/path/to/config.php
```

### Stop Daemon

```bash
# Graceful stop
php /whmcs/sync-daemon/daemon.php stop

# Force stop
php /whmcs/sync-daemon/daemon.php stop --force
```

### Daemon Status

```bash
# Check daemon status
php /whmcs/sync-daemon/daemon.php status

# Extended status
php /whmcs/sync-daemon/daemon.php status --verbose
```

## Domain Sync Process

### Sync Workflow

```
1. Fetch domain list from registrar
2. Compare with WHMCS database
3. Identify differences (status, expiry, nameservers)
4. Queue sync operations
5. Execute sync operations
6. Update WHMCS database
7. Trigger events for changes
8. Log all operations
```

### Sync Data Points

| Data Point | Description | Update Frequency |
|------------|-------------|------------------|
| Domain Status | Active, Expired, Cancelled | Every sync |
| Expiration Date | Domain expiry | Daily |
| Registrant Info | WHOIS contact | On change |
| Nameservers | DNS servers | On change |
| Auth Code | Transfer auth | On request |
| Transfer Status | Pending transfers | Every sync |

### Sync Worker Implementation

```php
namespace WHMCS\Sync\Workers;

class DomainSyncWorker
{
    private $registrar;
    private $batchSize = 50;

    public function process(): void
    {
        $domains = $this->getDomainsToSync();

        foreach ($domains as $domain) {
            $this->syncDomain($domain);
        }
    }

    private function syncDomain(Domain $domain): void
    {
        try {
            // Get registrar data
            $registrarData = $this->registrar->getDomainInfo($domain->name);

            // Compare with local data
            $changes = $this->detectChanges($domain, $registrarData);

            if (!empty($changes)) {
                // Apply changes to WHMCS
                $this->applyChanges($domain, $changes);

                // Log changes
                $this->logSync($domain, $changes);

                // Trigger events
                $this->triggerEvents($domain, $changes);
            }

            // Update sync timestamp
            $domain->last_synced_at = date('Y-m-d H:i:s');
            $domain->save();

        } catch (SyncException $e) {
            $this->handleSyncError($domain, $e);
        }
    }

    private function detectChanges(Domain $domain, array $registrarData): array
    {
        $changes = [];

        // Status change
        if ($domain->status !== $registrarData['status']) {
            $changes['status'] = [
                'from' => $domain->status,
                'to' => $registrarData['status']
            ];
        }

        // Expiration change
        if ($domain->expirydate !== $registrarData['expiry_date']) {
            $changes['expiry_date'] = [
                'from' => $domain->expirydate,
                'to' => $registrarData['expiry_date']
            ];
        }

        // Nameserver change
        if ($domain->nameservers !== $registrarData['nameservers']) {
            $changes['nameservers'] = [
                'from' => $domain->nameservers,
                'to' => $registrarData['nameservers']
            ];
        }

        return $changes;
    }
}
```

## Queue Management

### Queue Priorities

| Priority | Level | Description |
|----------|-------|-------------|
| 1 | Critical | Expired domain sync, transfer completion |
| 2 | High | Status change, urgent updates |
| 3 | Normal | Regular sync operations |
| 4 | Low | Batch sync, pricing updates |

### Queue Commands

```bash
# View queue status
php /whmcs/sync-daemon/queue.php status

# Process queue manually
php /whmcs/sync-daemon/queue.php process

# Clear failed items
php /whmcs/sync-daemon/queue.php clear-failed

# Retry failed items
php /whmcs/sync-daemon/queue.php retry
```

### Queue Table Schema

```sql
-- Queue table for sync operations
CREATE TABLE mod_sync_daemon_queue (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    sync_type VARCHAR(50) NOT NULL,
    domain_id INT UNSIGNED,
    registrar VARCHAR(50),
    priority TINYINT DEFAULT 3,
    status ENUM('pending', 'processing', 'completed', 'failed', 'retry') DEFAULT 'pending',
    attempts TINYINT UNSIGNED DEFAULT 0,
    max_attempts TINYINT UNSIGNED DEFAULT 5,
    error_message TEXT,
    payload JSON,
    scheduled_at TIMESTAMP NULL,
    started_at TIMESTAMP NULL,
    completed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_status_priority (status, priority),
    INDEX idx_sync_type (sync_type),
    INDEX idx_domain_id (domain_id),
    INDEX idx_scheduled (scheduled_at)
);
```

## Expiration Management

### Expiration Sync Rules

```php
// Expiration sync configuration
$expirationSync = [
    // Days before expiration to sync
    'sync_days_before' => [60, 30, 14, 7, 3, 1],

    // Actions at each milestone
    'actions' => [
        60 => ['send_reminder', 'enable_renewal'],
        30 => ['send_reminder', 'apply_discount'],
        14 => ['send_reminder', 'flag_account'],
        7  => ['final_reminder', 'suspend_optional'],
        3  => ['urgent_reminder'],
        1  => ['expiration_warning']
    ],

    // Post-expiration actions
    'post_expiration' => [
        0  => ['expire_domain', 'suspend_services'],
        30 => ['delete_domain_record'],
        60 => ['remove_from_dns']
    ]
];
```

### Expiration Status Flow

```
Active (180 days)
    |
    v
60 Days Before --> Sync + Reminder Email
    |
    v
30 Days Before --> Sync + Reminder + Mark for Attention
    |
    v
14 Days Before --> Final Reminder + Optional Suspension
    |
    v
7 Days Before --> Urgent Warning
    |
    v
1 Day Before --> Pre-Expiration Alert
    |
    v
EXPIRATION DAY --> Expire + Suspend Services + Lock
    |
    v
Grace Period (30 days) --> Redemption + Restore Available
    |
    v
Pending Delete (5 days) --> Cannot Restore
    |
    v
DELETED
```

## Monitoring

### Health Checks

```php
// Daemon health check endpoint
public function healthCheck(): array
{
    return [
        'status' => 'healthy',
        'uptime' => $this->getUptime(),
        'memory_usage' => memory_get_usage(true),
        'queue_depth' => $this->getQueueDepth(),
        'last_sync' => $this->getLastSyncTime(),
        'errors_24h' => $this->getErrorCount(24),
        'workers' => $this->getWorkerStatus()
    ];
}
```

### Monitoring Dashboard

Access at: `/admin/sync-daemon/dashboard.php`

Display:
- Daemon status (running/stopped)
- Queue depth by priority
- Sync rate (domains/hour)
- Error rate
- Last successful sync
- Registrar API health
- Memory/CPU usage

### Alert Thresholds

```php
// Alert configuration
$alerts = [
    'queue_depth_warning' => 1000,
    'queue_depth_critical' => 5000,
    'error_rate_warning' => 0.05,    // 5%
    'error_rate_critical' => 0.15,   // 15%
    'sync_lag_warning' => 7200,      // 2 hours
    'sync_lag_critical' => 14400,    // 4 hours
    'memory_usage_warning' => 0.75, // 75%
    'memory_usage_critical' => 0.90  // 90%
];
```

## Logging

### Log Levels

| Level | Description | When Used |
|-------|-------------|-----------|
| DEBUG | Detailed debugging info | Development |
| INFO | Normal operations | All sync operations |
| WARNING | Attention needed | Retry, slow responses |
| ERROR | Operation failed | API errors, timeouts |
| CRITICAL | System failure | Daemon crash, DB connection |

### Log Format

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "info",
  "worker": "domain_sync_1",
  "action": "sync_domain",
  "domain": "example.com",
  "changes": {
    "status": {"from": "active", "to": "expired"}
  },
  "duration_ms": 245,
  "registrar": "enom"
}
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Daemon not starting | Missing dependencies | Run composer install |
| Queue backed up | API rate limiting | Increase interval |
| Sync taking too long | Large domain count | Increase workers |
| Memory exhausted | Memory leak | Restart daemon |
| Stale data | Sync not running | Check cron, restart |

### Debug Mode

```bash
# Enable debug logging
php /whmcs/sync-daemon/daemon.php restart --debug

# Trace specific domain
php /whmcs/sync-daemon/daemon.php trace --domain=example.com

# Test registrar connection
php /whmcs/sync-daemon/test-connection.php --registrar=enom
```

## Performance Tuning

### Optimization Tips

1. **Batch processing** - Process domains in batches of 50-100
2. **Connection pooling** - Reuse API connections
3. **Caching** - Cache WHOIS and pricing data
4. **Rate limiting** - Respect registrar limits
5. **Parallel workers** - Use multiple processes

### Benchmarking

```bash
# Benchmark sync performance
php /whmcs/sync-daemon/benchmark.php \
    --domains=1000 \
    --workers=4 \
    --iterations=10
```

## See Also

- [Sync API Reference](./whmcs-sync-api.md)
- [Registrar Commands](./whmcs-registrar-commands.md)
- [Expiration Management](../billing/expiration-settings.md)
