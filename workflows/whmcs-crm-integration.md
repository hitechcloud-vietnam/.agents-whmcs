# WHMCS CRM Integration Workflow

## Overview
This workflow implements CRM integration for WHMCS.

## Prerequisites
- WHMCS with API access
- CRM system credentials
- Admin access for integration settings

## Step-by-Step Process

### Step 1: Create CRM Integration Manager
```php
<?php
// /includes/integrations/CRMIntegration.php

class CRMIntegration {
    private $crmApiUrl;
    private $crmApiKey;

    public function __construct()
    {
        $this->crmApiUrl = getConfig('crm_api_url');
        $this->crmApiKey = getConfig('crm_api_key');
    }

    /**
     * Sync client to CRM
     */
    public function syncClient(int $clientId): array
    {
        $client = getClientsDetails($clientId);

        $crmData = [
            'email' => $client['email'],
            'first_name' => $client['firstname'],
            'last_name' => $client['lastname'],
            'company' => $client['companyname'],
            'phone' => $client['phonenumber'],
            'address' => $client['address1'],
            'city' => $client['city'],
            'state' => $client['state'],
            'country' => $client['country']
        ];

        return $this->apiCall('POST', '/contacts', $crmData);
    }

    /**
     * Sync invoice to CRM
     */
    public function syncInvoice(int $invoiceId): array
    {
        $invoice = getInvoice($invoiceId);

        $crmData = [
            'invoice_id' => $invoiceId,
            'amount' => $invoice['total'],
            'status' => $invoice['status'],
            'client_email' => getClientEmail($invoice['userid'])
        ];

        return $this->apiCall('POST', '/invoices', $crmData);
    }

    private function apiCall(string $method, string $endpoint, array $data): array
    {
        $ch = curl_init($this->crmApiUrl . $endpoint);

        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_CUSTOMREQUEST => $method,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->crmApiKey,
                'Content-Type: application/json'
            ]
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true);
    }
}
```

### Step 2: Integration Hooks
```php
<?php
// /includes/hooks/crm_hooks.php

$crm = new CRMIntegration();

add_hook('ClientAdd', 1, function($vars) use ($crm) {
    $crm->syncClient($vars['userid']);
});

add_hook('InvoicePaid', 1, function($vars) use ($crm) {
    $crm->syncInvoice($vars['invoiceid']);
});
```

## Related Workflows
- [WHMCS Sync Automation](./whmcs-sync-automation.md)
- [WHMCS Data Export](./whmcs-data-export.md)