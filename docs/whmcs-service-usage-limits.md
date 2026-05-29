# WHMCS Service Usage Limits

## Overview

Service usage limits in WHMCS define resource constraints for services, including disk space, bandwidth, and other measurable resources.

## Limit Types

### Resource Limits

```php
// Define resource limits
[
    'service_id' => 1,
    'limits' => [
        'disk_gb' => 10,
        'bandwidth_gb' => 100,
        'emails' => 100,
        'databases' => 5,
        'subdomains' => 10
    ]
]
```

### Usage Tracking

```php
// Current usage
[
    'service_id' => 1,
    'disk_used' => 5,
    'disk_limit' => 10,
    'bandwidth_used' => 50,
    'bandwidth_limit' => 100
]
```

## Limit Configuration

### Set Limits

**Admin: Clients > Services > Resource Limits**

```php
// Configure limits
[
    'service_id' => 1,
    'disk_quota_gb' => 10,
    'bandwidth_quota_gb' => 100,
    'email_quota' => 100,
    'ftp_accounts' => 5
]
```

## Usage Warnings

### Warning Thresholds

```php
// Notify at thresholds
[
    'disk_warning' => 80,              // 80% of limit
    'bandwidth_warning' => 90,         // 90% of limit
    'notify_client' => true,
    'notify_admin' => false
]
```

## Limit Enforcement

### Block When Exceeded

```php
// Enforce limits
[
    'enforce_disk' => true,
    'enforce_bandwidth' => true,
    'action_on_exceed' => 'suspend'
]
```

## API Functions

```php
// Get usage limits
$result = localAPI('GetServiceUsageLimits', [
    'serviceid' => 1
]);

// Update limits
$result = localAPI('UpdateServiceUsageLimits', [
    'serviceid' => 1,
    'disk_quota_gb' => 20
]);
```

## Best Practices

1. **Set reasonable limits**: Match customer needs
2. **Monitor usage**: Track consumption
3. **Warn clients**: Alert before limits hit
4. **Offer upgrades**: Provide upgrade paths

## Related Documentation

- [Service Suspension](./whmcs-service-suspension.md)
- [Client Limitations](./whmcs-client-limitations.md)
- [Usage Billing](./whmcs-usage-billing.md)
- [Service Modification](./whmcs-service-modification.md)