# WHMCS Data Export Workflow

## Overview
This workflow implements data export (right to access) for WHMCS.

## Prerequisites
- WHMCS with client data management
- Admin access for export operations
- Legal/compliance requirements

## Step-by-Step Process

### Step 1: Create Data Export Manager
```php
<?php
// /includes/privacy/DataExportManager.php

class DataExportManager {
    /**
     * Export client data
     */
    public function exportClientData(int $clientId): array
    {
        return [
            'profile' => $this->getProfile($clientId),
            'services' => $this->getServices($clientId),
            'invoices' => $this->getInvoices($clientId),
            'tickets' => $this->getTickets($clientId),
            'consent' => $this->getConsent($clientId),
            'exported_at' => date('Y-m-d H:i:s')
        ];
    }

    private function getProfile(int $clientId): array
    {
        return Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();
    }

    private function getServices(int $clientId): array
    {
        return Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->get();
    }

    private function getInvoices(int $clientId): array
    {
        return Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->get();
    }

    private function getTickets(int $clientId): array
    {
        return Capsule::table('tbltickets')
            ->where('userid', $clientId)
            ->get();
    }

    private function getConsent(int $clientId): array
    {
        return Capsule::table('mod_user_consent')
            ->where('client_id', $clientId)
            ->get();
    }
}
```

### Step 2: Export Hooks
```php
<?php
// /includes/hooks/export_hooks.php

add_hook('ClientAreaPage', 1, function($vars) {
    // Provide data export link in client area
});
```

## Related Workflows
- [WHMCS GDPR Compliance](./whmcs-gdpr-compliance.md)
- [WHMCS Data Deletion](./whmcs-data-deletion.md)