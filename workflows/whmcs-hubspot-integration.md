# WHMCS HubSpot Integration Workflow

## Overview
This workflow implements HubSpot CRM integration for WHMCS.

## Prerequisites
- WHMCS with API access
- HubSpot account with API access
- Admin access for integration settings

## Step-by-Step Process

### Step 1: HubSpot Integration
```php
<?php
// /includes/integrations/HubSpotIntegration.php

class HubSpotIntegration {
    private $apiKey;

    public function __construct()
    {
        $this->apiKey = getConfig('hubspot_api_key');
    }

    /**
     * Sync client to HubSpot
     */
    public function syncContact(int $clientId): array
    {
        $client = getClientsDetails($clientId);

        $data = [
            'properties' => [
                'email' => $client['email'],
                'firstname' => $client['firstname'],
                'lastname' => $client['lastname'],
                'company' => $client['companyname'],
                'phone' => $client['phonenumber']
            ]
        ];

        return $this->apiCall('POST', '/crm/v3/objects/contacts', $data);
    }

    private function apiCall(string $method, string $endpoint, array $data): array
    {
        $ch = curl_init('https://api.hubapi.com' . $endpoint);

        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_CUSTOMREQUEST => $method,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json'
            ]
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true);
    }
}
```

### Step 2: HubSpot Hooks
```php
<?php
// /includes/hooks/hubspot_hooks.php

$hubspot = new HubSpotIntegration();

add_hook('ClientAdd', 1, function($vars) use ($hubspot) {
    $hubspot->syncContact($vars['userid']);
});
```

## Related Workflows
- [WHMCS CRM Integration](./whmcs-crm-integration.md)