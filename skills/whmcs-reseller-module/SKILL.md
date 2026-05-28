# WHMCS Reseller Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building reseller/provisioning tier modules.

## When to Use

- Creating reseller hosting modules
-Revenue sharing between tiers
- White-label provisioning

## Reseller Module Pattern

```php
<?php
// modules/servers/{reseller}/{reseller}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {reseller}_MetaData(): array {
    return [
        'DisplayName' => 'Reseller Hosting',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
    ];
}

function {reseller}_ConfigOptions(array $params): array {
    return [
        'ResellerPlan' => [
            'Type' => 'dropdown',
            'Options' => 'starter,professional,enterprise',
        ],
        'Whitelabel' => [
            'Type' => 'yesno',
            'Description' => 'Enable white-label branding',
        ],
        'DiskQuota' => [
            'Type' => 'text',
            'Default' => '100',
            'Description' => 'Disk quota in GB',
        ],
    ];
}

function {reseller}_CreateAccount(array $params): string {
    $api = new \Reseller\ApiClient($params);

    try {
        // Create reseller account
        $reseller = $api->createReseller([
            'username' => $params['username'],
            'domain' => $params['domain'],
            'plan' => $params['configoption1'],
            'whitelabel' => ($params['configoption2'] == 'on'),
        ]);

        // Create initial sub-accounts allocation
        Capsule::table('mod_reseller_allocations')->insert([
            'reseller_id' => $reseller['id'],
            'disk_quota' => ($params['configoption3'] ?? 100) * 1024 * 1024 * 1024,
            'bandwidth_quota' => 1000 * 1024 * 1024 * 1024,
            'used_disk' => 0,
            'used_bandwidth' => 0,
        ]);

        // Store credentials
        saveCustomFieldValue($params['serviceid'], 'reseller_id', $reseller['id']);
        saveCustomFieldValue($params['serviceid'], 'api_key', $reseller['api_key']);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {reseller}_SuspendAccount(array $params): string {
    $resellerId = getCustomFieldValue($params['serviceid'], 'reseller_id');

    try {
        $api = new \Reseller\ApiClient($params);
        $api->suspendReseller($resellerId);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {reseller}_ChangePackage(array $params): string {
    $resellerId = getCustomFieldValue($params['serviceid'], 'reseller_id');

    try {
        $api = new \Reseller\ApiClient($params);
        $api->updateResellerPlan($resellerId, $params['configoption1']);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {reseller}_UsageUpdate(array $params): array {
    // Return usage stats for the reseller
    $resellerId = getCustomFieldValue($params['serviceid'], 'reseller_id');
    $allocation = Capsule::table('mod_reseller_allocations')
        ->where('reseller_id', $resellerId)
        ->first();

    $usage = [
        'disk' => [
            'used' => $allocation->used_disk ?? 0,
            'limit' => $allocation->disk_quota ?? 0,
            'percent' => 0,
        ],
        'bandwidth' => [
            'used' => $allocation->used_bandwidth ?? 0,
            'limit' => $allocation->bandwidth_quota ?? 0,
            'percent' => 0,
        ],
    ];

    if ($allocation->disk_quota > 0) {
        $usage['disk']['percent'] = ($allocation->used_disk / $allocation->disk_quota) * 100;
    }
    if ($allocation->bandwidth_quota > 0) {
        $usage['bandwidth']['percent'] = ($allocation->used_bandwidth / $allocation->bandwidth_quota) * 100;
    }

    return $usage;
}

function {reseller}_ClientArea(array $params): array {
    $resellerId = getCustomFieldValue($params['serviceid'], 'reseller_id');
    $usage = {reseller}_UsageUpdate($params);

    return [
        'pagetitle' => 'Reseller Dashboard',
        'templatefile' => 'reseller_dashboard',
        'vars' => [
            'reseller_id' => $resellerId,
            'disk_used' => round($usage['disk']['used'] / 1024 / 1024 / 1024, 2),
            'disk_limit' => round($usage['disk']['limit'] / 1024 / 1024 / 1024, 2),
            'bandwidth_used' => round($usage['bandwidth']['used'] / 1024 / 1024 / 1024, 2),
            'bandwidth_limit' => round($usage['bandwidth']['limit'] / 1024 / 1024 / 1024, 2),
            'disk_percent' => $usage['disk']['percent'],
            'bandwidth_percent' => $usage['bandwidth']['percent'],
        ],
    ];
}
```

### Reseller Client Area Template

```smarty
<div class="reseller-dashboard">
    <h2>Reseller Dashboard</h2>

    <div class="usage-panel">
        <h3>Disk Usage</h3>
        <div class="progress-bar">
            <div class="progress-fill" style="width: {$disk_percent}%"></div>
        </div>
        <p>{$disk_used} GB / {$disk_limit} GB ({$disk_percent|string_format:"%.1f"}%)</p>
    </div>

    <div class="usage-panel">
        <h3>Bandwidth Usage</h3>
        <div class="progress-bar">
            <div class="progress-fill" style="width: {$bandwidth_percent}%"></div>
        </div>
        <p>{$bandwidth_used} GB / {$bandwidth_limit} GB ({$bandwidth_percent|string_format:"%.1f"}%)</p>
    </div>

    <div class="actions">
        <a href="?m=reseller&action=create_account" class="btn btn-primary">
            Create Sub-Account
        </a>
        <a href="?m=reseller&action=usage_history" class="btn">
            Usage History
        </a>
    </div>
</div>
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-clientarea-builder
- whmcs-reporting
