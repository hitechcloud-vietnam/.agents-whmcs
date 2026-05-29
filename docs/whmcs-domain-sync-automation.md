# WHMCS Domain Sync Automation

## Overview

Domain sync automation in WHMCS automatically synchronizes domain data with registrars, keeping status, expiry, and DNS information current.

## Automation Configuration

### Cron Setup

```bash
# Domain sync automation
0 */6 * * * php -q /whmcs/crons/domainsync.php
```

### Sync Settings

```php
// Automation configuration
[
    'enabled' => true,
    'frequency' => 'every_6_hours',
    'sync_status' => true,
    'sync_expiry' => true,
    'sync_dns' => true,
    'sync_lock' => true,
    'alert_on_change' => true
]
```

## Sync Operations

### Automatic Operations

```php
// What gets synced
[
    'domain_status' => true,
    'expiry_date' => true,
    'nameservers' => true,
    'registrar_lock' => true,
    'auto_renew' => true,
    'whois_contact' => false
]
```

### Sync Process

1. Query registrar for all domains
2. Compare with WHMCS database
3. Update discrepancies
4. Log all changes
5. Alert on status changes

## Alert Configuration

### Change Alerts

```php
// Notify on changes
[
    'alert_on_status_change' => true,
    'alert_on_expiry_change' => true,
    'alert_on_ns_change' => true,
    'alert_email' => 'admin@example.com',
    'alert_threshold_days' => 30
]
```

## Sync Reports

### Daily Sync Report

```smarty
Subject: Domain Sync Report - {$date}

Domains Synced: 150
Changes Detected: 5
Status Changes: 2
Expiry Updates: 3

View Details: {$admin_link}
```

## Error Handling

### Handle Failures

```php
// Sync error handling
[
    'retry_failed' => true,
    'max_retries' => 3,
    'retry_interval_hours' => 1,
    'alert_on_failure' => true
]
```

## API Functions

```php
// Run sync
$result = localAPI('RunDomainSync');

// Get sync status
$result = localAPI('GetDomainSyncStatus');
```

## Best Practices

1. **Regular sync**: Run multiple times daily
2. **Monitor logs**: Review sync reports
3. **Handle errors**: Retry failed syncs
4. **Alert changes**: Notify on status updates
5. **Update WHMCS**: Keep database current

## Related Documentation

- [Domain Sync](./whmcs-domain-sync.md)
- [Domain Renewal](./whmcs-domain-renewal.md)
- [Automation Settings](./whmcs-automation-settings.md)