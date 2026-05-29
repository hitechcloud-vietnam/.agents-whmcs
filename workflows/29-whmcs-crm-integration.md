# WHMCS CRM Integration Workflow

## Overview
This workflow covers integrating WHMCS with CRM systems for unified customer management.

## Step 1: CRM Integration Service

```php
<?php
// src/Service/CrmIntegrationService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class CrmIntegrationService
{
    private $crmApi;
    private $syncEnabled;

    public function __construct(ExternalApiService $crmApi)
    {
        $this->crmApi = $crmApi;
        $this->syncEnabled = Capsule::config('crm_sync_enabled') ?? false;
    }

    public function syncClientToCrm(int $clientId): array
    {
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();

        if (!$client) {
            return ['success' => false, 'error' => 'Client not found'];
        }

        $crmData = $this->transformClientForCrm($client);

        try {
            // Check if exists in CRM
            $existingCrmId = $this->getCrmIdByEmail($client->email);

            if ($existingCrmId) {
                $result = $this->crmApi->put("/contacts/$existingCrmId", $crmData);
            } else {
                $result = $this->crmApi->post('/contacts', $crmData);
            }

            // Update local record with CRM ID
            Capsule::table('mod_crm_sync')
                ->updateOrInsert(
                    ['client_id' => $clientId],
                    ['crm_id' => $result['id'], 'synced_at' => date('Y-m-d H:i:s')]
                );

            return ['success' => true, 'crm_id' => $result['id']];
        } catch (\Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    public function syncServiceToCrm(int $serviceId): array
    {
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();

        if (!$service) {
            return ['success' => false, 'error' => 'Service not found'];
        }

        $client = Capsule::table('tblclients')->where('id', $service->userid)->first();
        $product = Capsule::table('tblproducts')->where('id', $service->packageid)->first();

        $crmData = [
            'name' => $product->name ?? 'Service',
            'type' => 'service',
            'client_email' => $client->email,
            'status' => $service->domainstatus,
            'amount' => $this->getServicePrice($service),
            'next_due_date' => $service->nextduedate
        ];

        try {
            $result = $this->crmApi->post('/deals', $crmData);

            Capsule::table('mod_crm_service_sync')
                ->updateOrInsert(
                    ['service_id' => $serviceId],
                    ['crm_deal_id' => $result['id'], 'synced_at' => date('Y-m-d H:i:s')]
                );

            return ['success' => true, 'deal_id' => $result['id']];
        } catch (\Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    public function syncPaymentToCrm(int $invoiceId): array
    {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

        if (!$invoice) {
            return ['success' => false, 'error' => 'Invoice not found'];
        }

        // Find client CRM ID
        $syncRecord = Capsule::table('mod_crm_sync')
            ->where('client_id', $invoice->userid)
            ->first();

        if (!$syncRecord) {
            return ['success' => false, 'error' => 'Client not synced to CRM'];
        }

        $paymentData = [
            'contact_id' => $syncRecord->crm_id,
            'amount' => $invoice->total,
            'currency' => 'USD',
            'date' => $invoice->datepaid ?? date('Y-m-d'),
            'description' => "Invoice #{$invoiceId}"
        ];

        try {
            $result = $this->crmApi->post('/payments', $paymentData);
            return ['success' => true, 'payment_id' => $result['id']];
        } catch (\Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function transformClientForCrm($client): array
    {
        return [
            'email' => $client->email,
            'first_name' => $client->firstname,
            'last_name' => $client->lastname,
            'company' => $client->companyname,
            'phone' => $client->phonenumber,
            'address' => [
                'street' => $client->address1,
                'city' => $client->city,
                'state' => $client->state,
                'postal_code' => $client->postcode,
                'country' => $client->country
            ],
            'custom_fields' => [
                'whmcs_client_id' => $client->id,
                'client_status' => $client->status
            ]
        ];
    }

    private function getCrmIdByEmail(string $email): ?string
    {
        try {
            $result = $this->crmApi->get('/contacts', ['email' => $email]);
            return $result['contacts'][0]['id'] ?? null;
        } catch (\Exception $e) {
            return null;
        }
    }

    private function getServicePrice($service): float
    {
        $pricing = Capsule::table('tblpricing')
            ->where('type', 'product')
            ->where('relid', $service->packageid)
            ->first();

        $cycle = strtolower($service->billingcycle);
        return (float)($pricing->{$cycle} ?? $pricing->monthly ?? 0);
    }
}
```

## Step 2: CRM Sync Hooks

```php
<?php
// includes/hooks/crm_hook.php

add_hook('ClientAreaRegistration', 1, function($params) {
    $crmService = new \WHMCS\Module\Addon\YourModule\Service\CrmIntegrationService(
        new \WHMCS\Module\Addon\YourModule\Service\ExternalApiService(
            Capsule::config('crm_api_url'),
            Capsule::config('crm_api_key')
        )
    );

    $crmService->syncClientToCrm($params['userid']);
});

add_hook('OrderCreated', 1, function($params) {
    // Sync service to CRM
});
```

## Verification Checklist

- [ ] CRM service implemented
- [ ] Client sync working
- [ ] Service sync working
- [ ] Payment sync working
- [ ] CRM hooks registered
- [ ] Test sync completed successfully
