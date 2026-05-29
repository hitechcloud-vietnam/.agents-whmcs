# WHMCS Data Deletion Workflow

## Overview
This workflow implements data deletion (right to be forgotten) compliance for WHMCS.

## Prerequisites
- WHMCS with client data management
- Legal/compliance access for deletion requests
- Data anonymization capabilities

## Step-by-Step Process

### Step 1: Data Deletion Request Handler
```php
<?php
// /includes/privacy/DataDeletionManager.php

class DataDeletionManager {
    /**
     * Process client data deletion request
     */
    public function processDeletionRequest(int $clientId, array $options = []): array
    {
        $options = array_merge([
            'anonymize' => true,
            'retain_financial' => true,
            'notify' => true
        ], $options);

        // Check for legal holds
        if ($this->hasLegalHold($clientId)) {
            throw new Exception('Client data is under legal hold and cannot be deleted');
        }

        // Log deletion request
        $requestId = Capsule::table('mod_deletion_requests')->insertGetId([
            'client_id' => $clientId,
            'status' => 'processing',
            'options' => json_encode($options),
            'requested_at' => date('Y-m-d H:i:s'),
            'requested_by' => adminId() ?? 'system'
        ]);

        // Perform deletion/anonymization
        $results = $this->executeDeletion($clientId, $options);

        // Update request status
        Capsule::table('mod_deletion_requests')
            ->where('id', $requestId)
            ->update([
                'status' => 'completed',
                'results' => json_encode($results),
                'completed_at' => date('Y-m-d H:i:s')
            ]);

        // Send notification if required
        if ($options['notify']) {
            sendAdminEmail('Data Deletion Completed', $results);
        }

        return $results;
    }

    private function executeDeletion(int $clientId, array $options): array
    {
        $results = [];

        // Anonymize client profile
        $results['profile'] = $this->anonymizeProfile($clientId);

        // Handle service data
        $results['services'] = $this->anonymizeServices($clientId, $options['retain_financial']);

        // Handle invoices (retain for legal compliance)
        if ($options['retain_financial']) {
            $results['invoices'] = $this->anonymizeInvoices($clientId);
        }

        // Handle support tickets
        $results['tickets'] = $this->anonymizeTickets($clientId);

        // Delete activity logs
        $results['activity_logs'] = $this->deleteActivityLogs($clientId);

        // Delete sessions
        $results['sessions'] = $this->deleteSessions($clientId);

        return $results;
    }

    private function anonymizeProfile(int $clientId): int
    {
        $anonymizedEmail = 'deleted_' . $clientId . '_' . uniqid() . '@anonymized.local';

        return Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update([
                'firstname' => 'Deleted',
                'lastname' => 'User',
                'companyname' => '',
                'email' => $anonymizedEmail,
                'address1' => '',
                'address2' => '',
                'city' => '',
                'state' => '',
                'postcode' => '',
                'country' => '',
                'phonenumber' => '',
                'password' => '',
                'notes' => 'Anonymized due to data deletion request',
                'status' => 'Inactive'
            ]);
    }

    private function anonymizeServices(int $clientId, bool $retainFinancial): int
    {
        Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->update([
                'domain' => 'deleted_' . $clientId,
                'username' => '',
                'password' => '',
                'notes' => 'Service data anonymized'
            ]);

        return Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->count();
    }

    private function anonymizeInvoices(int $clientId): int
    {
        // Keep invoices for tax/legal purposes but anonymize
        Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->update([
                'notes' => 'Invoice data retained for legal compliance - owner anonymized'
            ]);

        return Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->count();
    }

    private function anonymizeTickets(int $clientId): int
    {
        Capsule::table('tbltickets')
            ->where('userid', $clientId)
            ->update([
                'name' => 'Deleted User',
                'email' => 'deleted@anonymized.local'
            ]);

        Capsule::table('tblticketreplies')
            ->join('tbltickets', 'tblticketreplies.ticketid', '=', 'tbltickets.id')
            ->where('tbltickets.userid', $clientId)
            ->update(['admin' => 'System']);

        return Capsule::table('tbltickets')
            ->where('userid', $clientId)
            ->count();
    }

    private function deleteActivityLogs(int $clientId): int
    {
        return Capsule::table('tblactivitylog')
            ->where('userid', $clientId)
            ->delete();
    }

    private function deleteSessions(int $clientId): int
    {
        return Capsule::table('tblsessions')
            ->where('userid', $clientId)
            ->delete();
    }

    private function hasLegalHold(int $clientId): bool
    {
        return Capsule::table('mod_legal_holds')
            ->where('client_id', $clientId)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->exists();
    }
}
```

### Step 2: Deletion Request Hooks
```php
<?php
// /includes/hooks/deletion_hooks.php

add_hook('DailyCronJob', 1, function($vars) {
    // Process pending deletion requests
    $pendingRequests = Capsule::table('mod_deletion_requests')
        ->where('status', 'pending')
        ->get();

    foreach ($pendingRequests as $request) {
        try {
            $deletionManager = new DataDeletionManager();
            $deletionManager->processDeletionRequest(
                $request->client_id,
                json_decode($request->options, true)
            );
        } catch (Exception $e) {
            Capsule::table('mod_deletion_requests')
                ->where('id', $request->id)
                ->update([
                    'status' => 'failed',
                    'error' => $e->getMessage()
                ]);
        }
    }

    return ['processed' => count($pendingRequests)];
});
```

## Deletion Requirements

| Data Type | Deletion | Anonymization | Retention |
|-----------|----------|---------------|-----------|
| Personal Info | Required | Alternative | Never |
| Financial Records | Retain | Alternative | 7 years |
| Support Tickets | - | Required | 3 years |
| Activity Logs | Required | - | 1 year |
| Consent Records | Required | Alternative | 5 years |

## Related Workflows
- [WHMCS Data Retention](./whmcs-data-retention.md)
- [WHMCS GDPR Compliance](./whmcs-gdpr-compliance.md)