# WHMCS CRM Integration

Complete guide for integrating WHMCS with CRM systems.

## Overview

Connect WHMCS with CRM platforms for unified customer management.

## CRM Data Sync

### Base Sync Class

```php
<?php
/**
 * Base CRM synchronization class
 */
abstract class BaseCRMIntegration
{
    protected string $apiUrl;
    protected string $apiKey;
    protected string $apiSecret;
    
    public function __construct(array $config)
    {
        $this->apiUrl = rtrim($config['api_url'], '/');
        $this->apiKey = $config['api_key'];
        $this->apiSecret = $config['api_secret'];
    }
    
    /**
     * Make authenticated API request
     */
    protected function request(string $method, string $endpoint, array $data = []): array
    {
        $ch = curl_init($this->apiUrl . $endpoint);
        
        $headers = [
            'Authorization: Bearer ' . $this->apiKey,
            'Content-Type: application/json',
        ];
        
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => $headers,
        ]);
        
        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        } elseif ($method === 'PUT') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'PUT');
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        } elseif ($method === 'DELETE') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'DELETE');
        }
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return json_decode($response, true) ?? [];
    }
    
    /**
     * Sync client to CRM
     */
    abstract public function syncClient(array $whmcsClient): string;
    
    /**
     * Get CRM contact ID by email
     */
    abstract public function getContactByEmail(string $email): ?array;
}
```

### Salesforce Integration

```php
<?php
/**
 * Salesforce CRM integration
 */
class SalesforceCRM extends BaseCRMIntegration
{
    private string $instanceUrl;
    
    public function __construct(array $config)
    {
        parent::__construct($config);
        $this->instanceUrl = $config['instance_url'];
    }
    
    /**
     * Create or update contact in Salesforce
     */
    public function syncClient(array $whmcsClient): string
    {
        // Check if contact exists
        $existing = $this->getContactByEmail($whmcsClient['email']);
        
        $data = [
            'FirstName' => $whmcsClient['firstname'],
            'LastName' => $whmcsClient['lastname'],
            'Email' => $whmcsClient['email'],
            'Phone' => $whmcsClient['phonenumber'] ?? '',
            'Company' => $whmcsClient['companyname'] ?? '',
            'MailingStreet' => $whmcsClient['address1'] ?? '',
            'MailingCity' => $whmcsClient['city'] ?? '',
            'MailingState' => $whmcsClient['state'] ?? '',
            'MailingPostalCode' => $whmcsClient['postcode'] ?? '',
            'MailingCountry' => $whmcsClient['country'] ?? '',
            'Description' => 'WHMOS Client ID: ' . $whmcsClient['id'],
        ];
        
        if ($existing) {
            // Update existing
            $result = $this->request('PATCH', "/services/data/v52.0/sobjects/Contact/{$existing['Id']}", $data);
            return $existing['Id'];
        } else {
            // Create new
            $result = $this->request('POST', '/services/data/v52.0/sobjects/Contact', $data);
            return $result['id'];
        }
    }
    
    /**
     * Search contact by email
     */
    public function getContactByEmail(string $email): ?array
    {
        $query = urlencode("SELECT Id, Email FROM Contact WHERE Email = '{$email}' LIMIT 1");
        $result = $this->request('GET', "/services/data/v52.0/query/?q={$query}");
        
        return $result['totalSize'] > 0 ? $result['records'][0] : null;
    }
    
    /**
     * Link service to contact
     */
    public function linkService(string $contactId, array $service): string
    {
        $data = [
            'Contact__c' => $contactId,
            'Name' => $service['domain'] ?? 'Service ' . $service['id'],
            'Service_ID__c' => $service['id'],
            'Product__c' => $service['product']['name'] ?? '',
            'Status__c' => $service['domainstatus'],
        ];
        
        $result = $this->request('POST', '/services/data/v52.0/sobjects/Service__c', $data);
        return $result['id'];
    }
}
```

### HubSpot Integration

```php
<?php
/**
 * HubSpot CRM integration
 */
class HubSpotCRM extends BaseCRMIntegration
{
    /**
     * Create or update contact
     */
    public function syncClient(array $whmcsClient): string
    {
        $properties = [
            'email' => $whmcsClient['email'],
            'firstname' => $whmcsClient['firstname'],
            'lastname' => $whmcsClient['lastname'],
            'phone' => $whmcsClient['phonenumber'] ?? '',
            'company' => $whmcsClient['companyname'] ?? '',
            'address' => ($whmcsClient['address1'] ?? '') . ', ' . 
                        ($whmcsClient['city'] ?? '') . ', ' . 
                        ($whmcsClient['state'] ?? '') . ' ' . 
                        ($whmcsClient['postcode'] ?? ''),
            'whmcs_client_id' => (string)$whmcsClient['id'],
        ];
        
        // Try to update existing
        $existing = $this->getContactByEmail($whmcsClient['email']);
        
        if ($existing) {
            $this->request('PATCH', "/crm/v3/objects/contacts/{$existing['id']}", [
                'properties' => $properties,
            ]);
            return $existing['id'];
        }
        
        // Create new
        $result = $this->request('POST', '/crm/v3/objects/contacts', [
            'properties' => $properties,
        ]);
        
        return $result['id'];
    }
    
    /**
     * Get contact by email
     */
    public function getContactByEmail(string $email): ?array
    {
        $result = $this->request('GET', "/crm/v3/objects/contacts/search", [
            'filterGroups' => [[
                'filters' => [[
                    'propertyName' => 'email',
                    'operator' => 'EQ',
                    'value' => $email,
                ]],
            ]],
        ]);
        
        return !empty($result['results']) ? $result['results'][0] : null;
    }
    
    /**
     * Create deal for service
     */
    public function createDeal(array $service, string $contactId): string
    {
        $data = [
            'properties' => [
                'dealname' => $service['domain'] ?? 'Service ' . $service['id'],
                'pipeline' => 'default',
                'dealstage' => 'closedwon',
                'amount' => $service['firstpaymentamount'] ?? 0,
                'closedate' => date('Y-m-d', strtotime($service['nextduedate'] ?? '+1 month')),
            ],
            'associations' => [
                [
                    'to' => ['id' => $contactId],
                    'types' => [['name' => 'deal_to_contact', 'category' => 'HUBSPOT_ASSOCIATION']],
                ],
            ],
        ];
        
        $result = $this->request('POST', '/crm/v3/objects/deals', $data);
        return $result['id'];
    }
}
```

## Webhook Sync

### CRM Sync Hook

```php
<?php
/**
 * Sync client to CRM on creation
 */
add_hook('ClientAdd', 1, function($vars) {
    $crm = new HubSpotCRM([
        'api_url' => 'https://api.hubapi.com',
        'api_key' => HUBSPOT_API_KEY,
    ]);
    
    $client = Capsule::table('tblclients')
        ->where('id', $vars['userid'])
        ->first();
    
    if ($client) {
        $crmId = $crm->syncClient((array)$client);
        
        // Store CRM ID
        Capsule::table('mod_crm_sync')
            ->updateOrInsert(
                ['client_id' => $client->id, 'crm_type' => 'hubspot'],
                ['crm_id' => $crmId, 'synced_at' => date('Y-m-d H:i:s')]
            );
        
        logActivity("Client #{$client->id} synced to HubSpot: {$crmId}");
    }
});

/**
 * Sync client on update
 */
add_hook('ClientEdit', 1, function($vars) {
    $crm = new HubSpotCRM([
        'api_url' => 'https://api.hubapi.com',
        'api_key' => HUBSPOT_API_KEY,
    ]);
    
    $client = Capsule::table('tblclients')
        ->where('id', $vars['userid'])
        ->first();
    
    if ($client) {
        $crm->syncClient((array)$client);
    }
});
```

### Service Sync Hook

```php
<?php
/**
 * Sync service to CRM
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    $crm = new HubSpotCRM([
        'api_url' => 'https://api.hubapi.com',
        'api_key' => HUBSPOT_API_KEY,
    ]);
    
    // Get CRM contact ID
    $crmRecord = Capsule::table('mod_crm_sync')
        ->where('client_id', $vars['params']['userid'])
        ->where('crm_type', 'hubspot')
        ->first();
    
    if ($crmRecord) {
        $service = Capsule::table('tblhosting')
            ->where('id', $vars['serviceid'])
            ->first();
        
        if ($service) {
            $dealId = $crm->createDeal((array)$service, $crmRecord->crm_id);
            
            Capsule::table('mod_crm_sync')
                ->updateOrInsert(
                    ['service_id' => $service->id, 'crm_type' => 'hubspot'],
                    ['crm_id' => $dealId, 'synced_at' => date('Y-m-d H:i:s')]
                );
            
            logActivity("Service #{$service->id} synced to HubSpot Deal: {$dealId}");
        }
    }
});
```

## Batch Sync

### Full CRM Sync

```php
<?php
/**
 * Perform full CRM sync
 */
function performFullCRMSync(string $crmType = 'hubspot'): array
{
    $results = [
        'synced' => 0,
        'failed' => 0,
        'errors' => [],
    ];
    
    $crm = $crmType === 'hubspot' 
        ? new HubSpotCRM(['api_url' => 'https://api.hubapi.com', 'api_key' => HUBSPOT_API_KEY])
        : new SalesforceCRM(['api_url' => 'https://salesforce.com', 'api_key' => '', 'instance_url' => '']);
    
    // Sync all clients
    $clients = Capsule::table('tblclients')
        ->where('status', '!=', 'Closed')
        ->get();
    
    foreach ($clients as $client) {
        try {
            $crmId = $crm->syncClient((array)$client);
            
            Capsule::table('mod_crm_sync')
                ->updateOrInsert(
                    ['client_id' => $client->id, 'crm_type' => $crmType],
                    ['crm_id' => $crmId, 'synced_at' => date('Y-m-d H:i:s')]
                );
            
            $results['synced']++;
            
        } catch (Exception $e) {
            $results['failed']++;
            $results['errors'][] = "Client {$client->id}: " . $e->getMessage();
        }
    }
    
    logActivity("CRM Sync completed: {$results['synced']} synced, {$results['failed']} failed");
    
    return $results;
}
```

## Best Practices

1. **Real-time sync** - Use hooks for immediate updates
2. **Batch sync** - Schedule periodic full syncs
3. **Handle duplicates** - Match by email/ID
4. **Log sync operations** - Track all sync activities
5. **Retry failed syncs** - Implement retry logic
6. **Rate limiting** - Respect CRM API limits

## Related Documentation

- [whmcs-integration-api.md](whmcs-integration-api.md)
- [whmcs-integration-webhooks.md](whmcs-integration-webhooks.md)
