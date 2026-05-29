# WHMCS Sync Issue Debug Workflow

## Overview
This workflow guides you through debugging synchronization issues in WHMCS modules.

## Prerequisites
- Sync logs
- External system access
- Database access

## Step-by-Step Guide

### Step 1: Check Sync Status
```sql
SELECT * FROM mod_yourmodule_sync_log 
WHERE status IN ('pending', 'failed')
ORDER BY created_at DESC
LIMIT 50;
```

### Step 2: Enable Detailed Logging
```php
public function syncClient(int $clientId): bool
{
    $this->log->info("Starting sync", ['client_id' => $clientId]);

    try {
        // Get client data
        $client = \WHMCS\Database\Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();
        $this->log->debug("Client data retrieved", ['data' => $client]);

        // Call external API
        $result = $this->api->sync($client);
        $this->log->debug("API response", ['result' => $result]);

        // Update status
        $this->updateSyncStatus($clientId, 'completed');

        return true;

    } catch (Exception $e) {
        $this->log->error("Sync failed", [
            'client_id' => $clientId,
            'error' => $e->getMessage(),
        ]);
        $this->updateSyncStatus($clientId, 'failed', $e->getMessage());

        return false;
    }
}
```

### Step 3: Run Manual Sync
```bash
#!/bin/bash
# manual-sync.sh

CLIENT_ID="${1}"

php /var/www/whmcs/modules/addons/yourmodule/scripts/sync.php --client=$CLIENT_ID --verbose
```

### Step 4: Check External System
```bash
# Check external API
curl -X GET "https://api.external.com/clients/$CLIENT_ID"

# Compare data
# WHMCS:
SELECT * FROM tblclients WHERE id = $CLIENT_ID;

# External:
# [Check via API or admin panel]
```

## Sync Issue Debug Checklist

### Investigation
- [ ] Sync logs reviewed
- [ ] Failed syncs identified
- [ ] Error messages captured
- [ ] External system checked

### Resolution
- [ ] Data mismatch fixed
- [ ] API credentials verified
- [ ] Retry mechanism triggered
- [ ] Manual sync completed

### Prevention
- [ ] Monitoring set up
- [ ] Alerts configured
- [ ] Retry automated
