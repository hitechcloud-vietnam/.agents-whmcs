# WHMCS Data Retention Workflow

## Overview
This workflow implements data retention policies and automated cleanup for WHMCS.

## Prerequisites
- WHMCS with database access
- Admin access for retention settings
- Legal/compliance requirements

## Step-by-Step Process

### Step 1: Data Retention Framework
```
RETENTION POLICIES:

Financial Records (7 years):
- Invoices
- Payment records
- Tax documents

Customer Data (Duration of service + 5 years):
- Profile information
- Service records
- Communication logs

Activity Logs (1 year):
- Login history
- Admin actions
- Security events

Temporary Data (30-90 days):
- Session data
- Cache files
- Failed login attempts

Marketing Data (Until consent withdrawn):
- Email preferences
- Campaign history
```

### Step 2: Retention Policy Configuration
```php
<?php
// /includes/retention/DataRetentionManager.php

class DataRetentionManager {
    private $policies = [
        'invoices' => [
            'table' => 'tblinvoices',
            'retention_days' => 2555, // 7 years
            'archive_before_delete' => true
        ],
        'activity_logs' => [
            'table' => 'tblactivitylog',
            'retention_days' => 365,
            'archive_before_delete' => false
        ],
        'support_tickets' => [
            'table' => 'tbltickets',
            'retention_days' => 1095, // 3 years after closure
            'archive_before_delete' => true
        ],
        'session_data' => [
            'table' => 'tblsessions',
            'retention_days' => 30,
            'archive_before_delete' => false
        ]
    ];

    /**
     * Run retention policy
     */
    public function runRetention(string $policyName): array
    {
        $policy = $this->policies[$policyName] ?? null;

        if (!$policy) {
            throw new Exception("Unknown retention policy: {$policyName}");
        }

        $cutoffDate = date('Y-m-d H:i:s', strtotime("-{$policy['retention_days']} days"));

        if ($policy['archive_before_delete']) {
            $this->archiveData($policy['table'], $cutoffDate);
        }

        $deleted = $this->deleteExpiredData($policy['table'], $cutoffDate);

        return [
            'policy' => $policyName,
            'cutoff_date' => $cutoffDate,
            'records_deleted' => $deleted
        ];
    }

    private function archiveData(string $table, string $cutoffDate): void
    {
        $data = Capsule::table($table)
            ->where('created_at', '<', $cutoffDate)
            ->get();

        foreach ($data as $record) {
            Capsule::table('mod_archived_data')->insert([
                'original_table' => $table,
                'data' => json_encode($record),
                'archived_at' => date('Y-m-d H:i:s')
            ]);
        }
    }

    private function deleteExpiredData(string $table, string $cutoffDate): int
    {
        return Capsule::table($table)
            ->where('created_at', '<', $cutoffDate)
            ->delete();
    }
}
```

### Step 3: Retention Cron Hook
```php
<?php
// /includes/hooks/retention_hooks.php

use WHMCS\Retention\DataRetentionManager;

add_hook('DailyCronJob', 1, function($vars) {
    $retentionManager = new DataRetentionManager();

    $results = [];

    // Run retention for each policy
    foreach (['activity_logs', 'session_data'] as $policy) {
        $results[$policy] = $retentionManager->runRetention($policy);
    }

    // Log results
    Capsule::table('mod_retention_log')->insert([
        'policies' => json_encode($results),
        'executed_at' => date('Y-m-d H:i:s')
    ]);

    // Alert on issues
    foreach ($results as $policy => $result) {
        if ($result['records_deleted'] > 10000) {
            sendAdminAlert("High retention cleanup: {$policy} - {$result['records_deleted']} records");
        }
    }

    return $results;
});

add_hook('MonthlyCronJob', 1, function($vars) {
    $retentionManager = new DataRetentionManager();

    // Monthly: sensitive data (invoices, tickets)
    $monthlyResults = [];

    foreach (['invoices', 'support_tickets'] as $policy) {
        $monthlyResults[$policy] = $retentionManager->runRetention($policy);
    }

    return $monthlyResults;
});
```

### Step 4: Legal Hold Override
```php
<?php
// Handle legal holds

class LegalHoldManager {
    private $holdTables = ['tblinvoices', 'tbltickets', 'tblactivitylog'];

    /**
     * Place client data on legal hold
     */
    public function placeHold(int $clientId, string $reason, string $expiresAt): void
    {
        Capsule::table('mod_legal_holds')->insert([
            'client_id' => $clientId,
            'reason' => $reason,
            'expires_at' => $expiresAt,
            'placed_by' => adminId(),
            'placed_at' => date('Y-m-d H:i:s')
        ]);
    }

    /**
     * Check if client is on hold
     */
    public function isOnHold(int $clientId): bool
    {
        return Capsule::table('mod_legal_holds')
            ->where('client_id', $clientId)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->exists();
    }

    /**
     * Skip retention for held data
     */
    public function skipRetention(string $table, array $clientIds): array
    {
        if (empty($clientIds)) {
            return [];
        }

        return Capsule::table($table)
            ->whereIn('userid', $clientIds)
            ->get()
            ->pluck('id')
            ->toArray();
    }
}
```

## Retention Schedule

| Data Type | Retention Period | Archive |
|-----------|-----------------|---------|
| Invoices | 7 years | Yes |
| Activity Logs | 1 year | No |
| Support Tickets | 3 years | Yes |
| Session Data | 30 days | No |
| Marketing Data | Until withdrawn | No |

## Related Workflows
- [WHMCS Data Deletion](./whmcs-data-deletion.md)
- [WHMCS GDPR Compliance](./whmcs-gdpr-compliance.md)