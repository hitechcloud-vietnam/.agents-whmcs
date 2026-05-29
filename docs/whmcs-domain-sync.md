# WHMCS Domain Sync

## Overview

Domain sync in WHMCS synchronizes domain registration data with the registrar, ensuring accurate status, expiry dates, and DNS information.

## Sync Configuration

### Enable Domain Sync

**Configuration > Domains > Domain Sync**

```php
// Sync settings
[
    'auto_sync' => true,
    'sync_frequency' => 'daily',         // hourly, daily, weekly
    'sync_modules' => true,
    'sync_registrar' => true
]
```

## Sync Types

### Registrar Sync

```php
// Sync with registrar
[
    'type' => 'registrar',
    'module' => 'enom',
    'sync_status' => true,
    'sync_expiry' => true,
    'sync_dns' => true,
    'sync_lock' => true
]
```

### Internal Sync

```php
// Internal WHMCS sync
[
    'type' => 'internal',
    'sync_status' => true,
    'sync_auto_renew' => true,
    'sync_nameservers' => true
]
```

## Sync Data

### Synced Information

| Data | Description |
|------|-------------|
| Status | Active, Expired, Transfer |
| Expiry Date | Registration expiration |
| Nameservers | DNS servers |
| Registrar Lock | Transfer lock status |
| Auto-Renew | Auto-renewal setting |
| WHOIS | Registrant information |

## Sync Cron

### Automated Sync

```bash
# Domain sync cron
0 */6 * * * php -q /whmcs/crons/domainsync.php
```

### Sync Process

1. Query registrar for domain data
2. Compare with WHMCS records
3. Update discrepancies
4. Log changes
5. Notify on status changes

## Status Synchronization

### Status Updates

```php
// Sync domain status
[
    'domain_id' => 1,
    'registrar_status' => 'active',
    'whmcs_status' => 'active',
    'match' => true
]
```

### Handle Changes

```php
// Status changed
[
    'domain_id' => 1,
    'old_status' => 'active',
    'new_status' => 'expired',
    'action' => 'notify_admin',
    'suspend_related' => true
]
```

## Expiry Sync

### Expiry Date Updates

```php
// Sync expiry dates
[
    'domain_id' => 1,
    'registrar_expiry' => '2025-05-15',
    'whmcs_expiry' => '2025-05-15',
    'match' => true
]
```

### Mismatch Handling

```php
// Handle expiry mismatch
[
    'domain_id' => 1,
    'registrar_expiry' => '2026-05-15',  // Extended via renewal
    'whmcs_expiry' => '2025-05-15',
    'update_whmcs' => true,
    'log_change' => true
]
```

## DNS Sync

### Nameserver Sync

```php
// Sync DNS
[
    'domain_id' => 1,
    'registrar_ns' => ['ns1.com', 'ns2.com'],
    'whmcs_ns' => ['ns1.com', 'ns2.com'],
    'match' => true
]
```

## Sync Logs

### Log All Changes

```php
// Sync history
[
    'domain_id' => 1,
    'sync_date' => '2024-05-15',
    'changes' => [
        ['field' => 'expiry', 'old' => '2024-05-15', 'new' => '2025-05-15']
    ]
]
```

## Sync Notifications

### Alert on Changes

```php
// Notify on sync changes
[
    'notify_on_change' => true,
    'notify_on_expiry' => true,
    'alert_threshold_days' => 30
]
```

## API Functions

```php
// Sync domain
$result = localAPI('SyncDomain', [
    'domainid' => 1
]);

// Sync all domains
$result = localAPI('SyncAllDomains');

// Get sync status
$result = localAPI('GetDomainSyncStatus', [
    'domainid' => 1
]);
```

## Hooks

```php
// Hook: DomainSynced
add_hook('DomainSynced', 1, function($vars) {
    // $vars['domainid']
    // $vars['changes']
    // Handle sync changes
});
```

## Best Practices

1. **Regular sync**: Run daily at minimum
2. **Monitor logs**: Review sync changes
3. **Handle failures**: Retry failed syncs
4. **Alert on changes**: Notify on status updates
5. **Update WHMCS**: Keep WHMCS current with registrar

## Related Documentation

- [Domain Sync Automation](./whmcs-domain-sync-automation.md)
- [Domain Renewal](./whmcs-domain-renewal.md)
- [Domain Registration](./whmcs-domain-registration.md)
- [TLD Sync](./whmcs-tld-sync.md)