# WHMCS Salesforce Integration Workflow

## Overview
This workflow implements Salesforce CRM integration for WHMCS.

## Prerequisites
- WHMCS with API access
- Salesforce account with API access
- Connected App credentials

## Step-by-Step Process

### Step 1: Salesforce Integration
```php
<?php
// /includes/integrations/SalesforceIntegration.php

class SalesforceIntegration {
    private $instanceUrl;
    private $accessToken;

    public function __construct()
    {
        $this->instanceUrl = getConfig('salesforce_instance_url');
        $this->accessToken = $this->authenticate();
    }

    private function authenticate(): string
    {
        $ch = curl_init($this->instanceUrl . '/services/oauth2/token');

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query([
                'grant_type' => 'client_credentials',
                'client_id' => getConfig('salesforce_client_id'),
                'client_secret' => getConfig('salesforce_client_secret')
            ])
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        $result = json_decode($response, true);

        return $result['access_token'];
    }

    /**
     * Create/update Salesforce contact
     */
    public function syncContact(int $clientId): array
    {
        $client = getClientsDetails($clientId);

        $data = [
            'FirstName' => $client['firstname'],
            'LastName' => $client['lastname'],
            'Email' => $client['email'],
            'Phone' => $client['phonenumber'],
            'Account.Name' => $client['companyname'] ?: $client['fullname'],
            'MailingCity' => $client['city'],
            'MailingCountry' => $client['country']
        ];

        return $this->apiCall('POST', '/sobjects/Contact', $data);
    }

    /**
     * Create Salesforce opportunity
     */
    public function createOpportunity(int $invoiceId): array
    {
        $invoice = getInvoice($invoiceId);
        $client = getClientsDetails($invoice['userid']);

        $data = [
            'Name' => 'Invoice #' . $invoiceId,
            'StageName' => $invoice['status'] === 'Paid' ? 'Closed Won' : 'Proposal',
            'Amount' => $invoice['total'],
            'CloseDate' => $invoice['duedate'],
            'Description' => 'WHMCS Invoice'
        ];

        return $this->apiCall('POST', '/sobjects/Opportunity', $data);
    }

    private function apiCall(string $method, string $endpoint, array $data): array
    {
        $ch = curl_init($this->instanceUrl . '/services/data/v52.0' . $endpoint);

        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_CUSTOMREQUEST => $method,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->accessToken,
                'Content-Type: application/json'
            ]
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true);
    }
}
```

### Step 2: Salesforce Hooks
```php
<?php
// /includes/hooks/salesforce_hooks.php

$salesforce = new SalesforceIntegration();

add_hook('ClientAdd', 1, function($vars) use ($salesforce) {
    $salesforce->syncContact($vars['userid']);
});

add_hook('InvoicePaid', 1, function($vars) use ($salesforce) {
    $salesforce->createOpportunity($vars['invoiceid']);
});
```

## Related Workflows
- [WHMCS CRM Integration](./whmcs-crm-integration.md)