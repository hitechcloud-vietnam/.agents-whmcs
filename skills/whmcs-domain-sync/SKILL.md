# WHMCS Domain Sync Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing domain synchronization in WHMCS registrar modules.

## When to Use

- Syncing domain status with registrar
- Checking expiry dates
- Updating domain contacts

## Sync Implementation

```php
<?php
// modules/registrars/{module}/{module}.php

function {module}_Sync(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $info = $api->getDomainInfo($params['domain']);

        $status = mapDomainStatus($info['status']);
        $expiry = formatDate($info['expiry_date']);
        $nextBill = calculateNextBillDate($info['expiry_date']);

        return [
            'status' => $status,
            'expiry' => $expiry,
            'nextinvoicedate' => $nextBill,
            'registrationstatus' => $info['active'] ?? true,
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {module}_TransferSync(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $transfer = $api->getTransferStatus($params['domain']);

        return [
            'status' => mapTransferStatus($transfer['status']),
            'pending' => in_array($transfer['status'], ['pending', 'awaiting']),
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function mapDomainStatus(string $status): string {
    $status = strtolower($status);

    $map = [
        'ok' => 'Active',
        'active' => 'Active',
        'servertransferprohibited' => 'Pending Transfer',
        'clienttransferprohibited' => 'Pending Transfer',
        'expired' => 'Expired',
        'pendingdelete' => 'Redemption',
    ];

    return $map[$status] ?? 'Active';
}

function mapTransferStatus(string $status): string {
    return match ($status) {
        'pending' => 'Pending',
        'approved' => 'Completed',
        'rejected' => 'Failed',
        'cancelled' => 'Cancelled',
        default => 'Pending',
    };
}
```

## Cron Sync

```php
// modules/addons/{module}/cron.php
add_hook('DailyCronJob', 1, function($vars) {
    syncAllDomains();
});

function syncAllDomains(): void {
    $domains = Capsule::table('tbldomains')
        ->where('registrar', 'modulename')
        ->whereIn('status', ['Active', 'Pending Transfer'])
        ->get();

    foreach ($domains as $domain) {
        try {
            $params = getRegistrarParams($domain->userid);
            $params['domain'] = $domain->domain;

            $result = module_Sync($params);

            if (isset($result['error'])) {
                logSyncError($domain->id, $result['error']);
                continue;
            }

            updateDomain($domain->id, [
                'status' => $result['status'],
                'expirydate' => $result['expiry'],
                'nextinvoicedate' => $result['nextinvoicedate'],
            ]);

            logSyncSuccess($domain->id);
        } catch (\Exception $e) {
            logSyncError($domain->id, $e->getMessage());
        }
    }
}
```

## Checklist

- [ ] Sync function returns status/expiry
- [ ] TransferSync for pending transfers
- [ ] Status mapping
- [ ] Date formatting
- [ ] Cron sync setup

---

**Related Skills:**
- whmcs-registrar-builder
- whmcs-cron-automation
- whmcs-hooks-development