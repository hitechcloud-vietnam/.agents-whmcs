# WHMCS Resource Quota Module

Resource quota management for controlling user resource consumption.

## Features

- Per-user and per-product quota limits
- Resource categories (compute, storage, bandwidth, API calls)
- Soft limits with grace periods
- Quota tracking and enforcement
- Usage notifications at threshold levels
- Quota renewal and reset policies
- Quota transfer between users
- Custom quota allocation
- Resource pooling
- Over-quota handling policies

## Installation

1. Copy module to `/path/to/whmcs/modules/addons/resourcequota/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure quota limits and policies

## Usage

```php
// Create quota limit
$result = resourcequota_CreateQuota(array(
    'user_id' => $clientId,
    'resource_name' => 'api_calls',
    'limit_value' => 10000,
    'period' => 'monthly',
    'soft_limit_percent' => 80,
    'enforcement' => 'hard'
));

// Get user quotas
$quotas = resourcequota_GetUserQuotas($clientId);

// Check quota availability
$result = resourcequota_CheckQuota($clientId, 'api_calls', 500);
if (!$result['available']) {
    echo "Quota exceeded: " . $result['message'];
}

// Consume quota resource
$result = resourcequota_ConsumeQuota($clientId, 'api_calls', 100);
// Returns: success, remaining, is_exceeded

// Get quota usage
$usage = resourcequota_GetQuotaUsage($clientId, 'api_calls');
// Returns: used, limit, remaining, percent, period_reset

// Create resource pool
resourcequota_CreatePool(array(
    'pool_name' => 'Shared Resources',
    'total_resources' => 100000,
    'shared_users' => array($userId1, $userId2)
));

// Allocate from pool
resourcequota_AllocateFromPool($poolId, $userId, 5000);

// Get available resources
$available = resourcequota_GetAvailableQuota($clientId, 'storage_gb');

// Reset quota
resourcequota_ResetQuota($quotaId);

// Transfer quota between users
resourcequota_TransferQuota($fromUserId, $toUserId, 'api_calls', 1000);

// Get quota history
$history = resourcequota_GetQuotaHistory($clientId, 'api_calls', 30);

// Update quota limit
resourcequota_UpdateQuota($quotaId, array(
    'limit_value' => 15000,
    'soft_limit_percent' => 85
));

// Get quota notifications
$notifications = resourcequota_GetNotifications($userId, 50);

// Delete quota
resourcequota_DeleteQuota($quotaId);
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| EnableStrictEnforcement | yesno | yes | Strictly enforce quotas |
| DefaultPeriod | dropdown | monthly | Default renewal period |
| NotificationLevels | text | 50,80,90,100 | Threshold levels (%) |
| GracePeriodHours | text | 24 | Grace period for over-quota |
| EnablePoolSharing | yesno | yes | Enable resource pooling |
| DefaultSoftLimit | text | 80 | Default soft limit (%) |

## Resource Types

| Category | Resource | Unit |
|----------|----------|------|
| compute | cpu_cores | cores |
| compute | memory_gb | GB |
| compute | compute_units | units |
| storage | disk_gb | GB |
| storage | backup_gb | GB |
| network | bandwidth_gb | GB |
| network | api_calls | calls |
| network | emails_sent | emails |
| custom | custom_units | units |

## Period Types

| Period | Description |
|--------|-------------|
| hourly | Resets every hour |
| daily | Resets every day |
| weekly | Resets every week |
| monthly | Resets every month |
| yearly | Resets every year |
| never | Never resets |

## Enforcement Modes

| Mode | Description |
|------|-------------|
| hard | Block resource access when exceeded |
| soft | Only warn when exceeded |
| flexible | Allow overage with additional charges |

## Database Tables

- `mod_resourcequota_quotas` - Quota limits
- `mod_resourcequota_usage` - Usage records
- `mod_resourcequota_history` - Usage history
- `mod_resourcequota_pools` - Resource pools
- `mod_resourcequota_notifications` - Notifications
- `mod_resourcequota_renewals` - Renewal policies

## API Functions

| Function | Description |
|----------|-------------|
| `resourcequota_CreateQuota()` | Create quota limit |
| `resourcequota_GetUserQuotas()` | Get all user quotas |
| `resourcequota_CheckQuota()` | Check if resource available |
| `resourcequota_ConsumeQuota()` | Consume resource |
| `resourcequota_GetQuotaUsage()` | Get current usage |
| `resourcequota_UpdateQuota()` | Update quota settings |
| `resourcequota_ResetQuota()` | Reset usage counters |
| `resourcequota_DeleteQuota()` | Remove quota |
| `resourcequota_TransferQuota()` | Transfer between users |
| `resourcequota_CreatePool()` | Create resource pool |
| `resourcequota_AllocateFromPool()` | Allocate pool resources |
| `resourcequota_GetAvailableQuota()` | Get remaining resources |
| `resourcequota_GetQuotaHistory()` | Get usage history |
| `resourcequota_GetNotifications()` | Get threshold notifications |
