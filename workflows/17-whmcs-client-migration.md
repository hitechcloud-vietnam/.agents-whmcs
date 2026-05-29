# WHMCS Client Migration Workflow

## Overview
This workflow handles client data migration between WHMCS installations or during system upgrades.

## Step 1: Migration Service

```php
<?php
// src/Service/MigrationService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class MigrationService
{
    public function exportClient(int $clientId): array
    {
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();
        if (!$client) {
            return ['success' => false, 'error' => 'Client not found'];
        }

        $data = [
            'client' => (array)$client,
            'contacts' => $this->getContacts($clientId),
            'services' => $this->getServices($clientId),
            'domains' => $this->getDomains($clientId),
            'invoices' => $this->getInvoices($clientId),
            'orders' => $this->getOrders($clientId),
            'tickets' => $this->getTickets($clientId),
            'notes' => $this->getNotes($clientId)
        ];

        return [
            'success' => true,
            'data' => $data,
            'exported_at' => date('Y-m-d H:i:s')
        ];
    }

    public function importClient(array $data): array
    {
        try {
            Capsule::connection()->transaction(function() use ($data) {
                // Import client
                $clientId = Capsule::table('tblclients')->insertGetId($data['client']);

                // Import related data with new client ID
                if (!empty($data['contacts'])) {
                    foreach ($data['contacts'] as &$contact) {
                        unset($contact['id']);
                        $contact['userid'] = $clientId;
                    }
                    Capsule::table('tblcontacts')->insert($data['contacts']);
                }

                // ... import other related data
            });

            return ['success' => true];
        } catch (\Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function getContacts(int $clientId): array
    {
        return Capsule::table('tblcontacts')->where('userid', $clientId)->get()->toArray();
    }

    private function getServices(int $clientId): array
    {
        return Capsule::table('tblhosting')->where('userid', $clientId)->get()->toArray();
    }

    private function getDomains(int $clientId): array
    {
        return Capsule::table('tbldomains')->where('userid', $clientId)->get()->toArray();
    }

    private function getInvoices(int $clientId): array
    {
        return Capsule::table('tblinvoices')->where('userid', $clientId)->get()->toArray();
    }

    private function getOrders(int $clientId): array
    {
        return Capsule::table('tblorders')->where('userid', $clientId)->get()->toArray();
    }

    private function getTickets(int $clientId): array
    {
        return Capsule::table('tbltickets')->where('userid', $clientId)->get()->toArray();
    }

    private function getNotes(int $clientId): array
    {
        return Capsule::table('tblnotes')->where('userid', $clientId)->get()->toArray();
    }
}
```

## Verification Checklist

- [ ] Migration service implemented
- [ ] Export functionality working
- [ ] Import functionality working
- [ ] Data integrity verified
