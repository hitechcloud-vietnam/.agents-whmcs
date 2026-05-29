# WHMCS Registrar Commands Documentation

## Overview

Registrar commands in WHMCS provide the interface between WHMCS and domain registrars. This documentation covers all standard registrar commands and their implementation.

## Registrar Module Structure

### Required Files

```
modules/registrars/
├── ModuleName/
│   ├── ModuleName.php          # Main module class
│   ├── includes/
│   │   └── EppClient.php       # EPP client (if applicable)
│   └── composer.json
```

### Module Class Structure

```php
namespace WHMCS\Module\Registrar;

class ModuleName implements RegistrarModuleInterface
{
    protected $params;

    public function __construct(array $params = [])
    {
        $this->params = $params;
    }

    // All required methods...
}
```

## Required Commands

### getConfiguration

Returns module configuration options.

```php
public function getConfiguration()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Module Name'
        ],
        'APIUsername' => [
            'Type' => 'text',
            'Size' => '20',
            'Description' => 'Your API username'
        ],
        'APIPassword' => [
            'Type' => 'password',
            'Size' => '40',
            'Description' => 'Your API password'
        ],
        'APIKey' => [
            'Type' => 'password',
            'Size' => '80',
            'Description' => 'API key (if applicable)'
        ],
        'TestMode' => [
            'Type' => 'yesno',
            'Description' => 'Enable test mode'
        ],
        'Logging' => [
            'Type' => 'yesno',
            'Description' => 'Enable API logging'
        ]
    ];
}
```

### getDomainInformation

Retrieves domain information from registrar.

```php
public function getDomainInformation($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $response = $api->getDomainInfo([
        'domain' => $params['domain'],
        'includeContacts' => true,
        'includeNameservers' => true,
        'includeStatus' => true
    ]);

    if (!$response['success']) {
        return [
            'error' => $response['error_message']
        ];
    }

    return [
        'domain' => $params['domain'],
        'status' => $this->mapStatus($response['status']),
        'registrationdate' => $response['created_date'],
        'expirydate' => $response['expiry_date'],
        'dnsmanagement' => !empty($response['dnssec']),
        'emailforwarding' => true,
        'idprotection' => $response['privacy_enabled'],
        'nameservers' => $response['nameservers'],
        'registrant' => $response['contacts']['registrant'],
        'admin' => $response['contacts']['admin'],
        'tech' => $response['contacts']['tech'],
        'billing' => $response['contacts']['billing']
    ];
}

private function mapStatus($registrarStatus)
{
    $statusMap = [
        'active' => 'active',
        'ok' => 'active',
        'serverTransferProhibited' => 'transfer_lock',
        'clientTransferProhibited' => 'client_lock',
        'pendingDelete' => 'pending_delete',
        'inactive' => 'inactive',
        'expired' => 'expired'
    ];

    return $statusMap[$registrarStatus] ?? 'unknown';
}
```

### registerDomain

Registers a new domain.

```php
public function registerDomain($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    // Validate registrant data
    $validation = $this->validateRegistrantData($params['registrant']);
    if (!$validation['valid']) {
        return [
            'error' => 'Invalid registrant data: ' . $validation['errors'][0]
        ];
    }

    // Prepare registration request
    $requestData = [
        'domain' => $params['domain'],
        'period' => $params['regperiod'],
        'nameservers' => $params['nslist'],
        'registrant' => $this->formatRegistrant($params['registrant']),
        'admin' => $this->formatContact($params['admin']),
        'tech' => $this->formatContact($params['tech']),
        'billing' => $this->formatContact($params['billing']),
        'privacy' => $params['idprotection'] ?? false,
        'dnssec' => $params['dnsmanagement'] ?? false
    ];

    $response = $api->registerDomain($requestData);

    if ($response['success']) {
        return [
            'success' => true,
            'domain' => $params['domain'],
            'registrationdate' => date('Y-m-d'),
            'expirydate' => $response['expiry_date']
        ];
    }

    return [
        'error' => $response['error_message']
    ];
}
```

### transferDomain

Initiates or completes domain transfer.

```php
public function transferDomain($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $requestData = [
        'domain' => $params['domain'],
        'authcode' => $params['transfersecret'],
        'nameservers' => $params['nslist'],
        'registrant' => $this->formatRegistrant($params['registrant'])
    ];

    $response = $api->transferDomain($requestData);

    if ($response['success']) {
        return [
            'success' => true,
            'transferid' => $response['transfer_id'],
            'status' => $response['transfer_status'],
            'expirydate' => $response['expiry_date']
        ];
    }

    return [
        'error' => $response['error_message']
    ];
}
```

### renewDomain

Renews domain registration.

```php
public function renewDomain($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $response = $api->renewDomain([
        'domain' => $params['domain'],
        'period' => $params['regperiod'],
        'current_expiry' => $params['expirydate']
    ]);

    if ($response['success']) {
        return [
            'success' => true,
            'expirydate' => $response['new_expiry_date']
        ];
    }

    return [
        'error' => $response['error_message']
    ];
}
```

### releaseDomain

Releases domain to another registrar (PUSH).

```php
public function releaseDomain($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $response = $api->releaseDomain([
        'domain' => $params['domain'],
        'tag' => $params['new_tag'] // New registrar's EPF tag
    ]);

    return [
        'success' => $response['success'],
        'error' => $response['error_message'] ?? null
    ];
}
```

### modifyNS

Updates domain nameservers.

```php
public function modifyNS($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $response = $api->updateNameservers([
        'domain' => $params['domain'],
        'nameservers' => $params['nslist']
    ]);

    if ($response['success']) {
        return [
            'success' => true
        ];
    }

    return [
        'error' => $response['error_message']
    ];
}
```

### modifyContact

Updates domain contact information.

```php
public function modifyContact($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $contacts = [];

    if (isset($params['fullname'])) {
        $contacts['registrant'] = $this->formatContact($params);
    }
    if (isset($params['adminfullname'])) {
        $contacts['admin'] = $this->formatContact($params, 'admin');
    }
    if (isset($params['techfullname'])) {
        $contacts['tech'] = $this->formatContact($params, 'tech');
    }
    if (isset($params['billingfullname'])) {
        $contacts['billing'] = $this->formatContact($params, 'billing');
    }

    $response = $api->updateContacts([
        'domain' => $params['domain'],
        'contacts' => $contacts
    ]);

    return [
        'success' => $response['success'],
        'error' => $response['error_message'] ?? null
    ];
}
```

### getContactDetails

Retrieves domain contact information.

```php
public function getContactDetails($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $response = $api->getContacts([
        'domain' => $params['domain']
    ]);

    if (!$response['success']) {
        return [
            'error' => $response['error_message']
        ];
    }

    return [
        'Registrant' => $response['registrant'],
        'Admin' => $response['admin'],
        'Tech' => $response['tech'],
        'Billing' => $response['billing']
    ];
}
```

### sync

Synchronizes domain status with registrar.

```php
public function sync($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $response = $api->getDomainStatus([
        'domain' => $params['domain']
    ]);

    return [
        'active' => ($response['status'] === 'active'),
        'expired' => ($response['status'] === 'expired'),
        'expirydate' => $response['expiry_date'],
        'error' => $response['error_message'] ?? null
    ];
}
```

## Optional Commands

### transferSync

Synchronizes transfer status.

```php
public function transferSync($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $response = $api->getTransferStatus([
        'domain' => $params['domain']
    ]);

    $statusMap = [
        'pending' => 'pending',
        'approved' => 'completed',
        'rejected' => 'rejected',
        'cancelled' => 'cancelled'
    ];

    return [
        'status' => $statusMap[$response['status']] ?? 'unknown',
        'completed' => ($response['status'] === 'approved'),
        'error' => $response['error_message'] ?? null
    ];
}
```

### requestDelete

Requests domain deletion.

```php
public function requestDelete($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $response = $api->requestDeletion([
        'domain' => $params['domain']
    ]);

    return [
        'success' => $response['success'],
        'error' => $response['error_message'] ?? null
    ];
}
```

### enablePrivacy

Enables WHOIS privacy.

```php
public function enablePrivacy($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $response = $api->enablePrivacy([
        'domain' => $params['domain']
    ]);

    return [
        'success' => $response['success'],
        'privacyid' => $response['privacy_id'] ?? null,
        'error' => $response['error_message'] ?? null
    ];
}
```

### disablePrivacy

Disables WHOIS privacy.

```php
public function disablePrivacy($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $response = $api->disablePrivacy([
        'domain' => $params['domain']
    ]);

    return [
        'success' => $response['success'],
        'error' => $response['error_message'] ?? null
    ];
}
```

### getEPPCodes

Retrieves transfer auth code (EPP key).

```php
public function getEPPCodes($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $response = $api->getAuthCode([
        'domain' => $params['domain']
    ]);

    return [
        'success' => $response['success'],
        'eppcode' => $response['auth_code'] ?? null,
        'error' => $response['error_message'] ?? null
    ];
}
```

### saveRegistrarLock

Enables or disables domain lock.

```php
public function saveRegistrarLock($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $lockStatus = ($params['lockenabled'] === 'locked');

    $response = $api->setLock([
        'domain' => $params['domain'],
        'lock' => $lockStatus
    ]);

    return [
        'success' => $response['success'],
        'error' => $response['error_message'] ?? null
    ];
}
```

### getDNSZoneRecords

Gets DNS zone records.

```php
public function getDNSZoneRecords($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $response = $api->getDnsRecords([
        'domain' => $params['domain']
    ]);

    if (!$response['success']) {
        return [
            'error' => $response['error_message']
        ];
    }

    $records = [];
    foreach ($response['records'] as $record) {
        $records[] = [
            'id' => $record['id'],
            'name' => $record['name'],
            'type' => $record['type'],
            'value' => $record['value'],
            'priority' => $record['priority'] ?? 0,
            'ttl' => $record['ttl'] ?? 3600
        ];
    }

    return [
        'zone_id' => $response['zone_id'],
        'records' => $records
    ];
}
```

### saveDNSZoneRecords

Saves DNS zone records.

```php
public function saveDNSZoneRecords($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $records = [];
    foreach ($params['records'] as $record) {
        $records[] = [
            'name' => $record['name'],
            'type' => $record['type'],
            'value' => $record['value'],
            'priority' => $record['priority'] ?? 0,
            'ttl' => $record['ttl'] ?? 3600
        ];
    }

    $response = $api->setDnsRecords([
        'domain' => $params['domain'],
        'records' => $records
    ]);

    return [
        'success' => $response['success'],
        'error' => $response['error_message'] ?? null
    ];
}
```

### checkDnsAvailability

Checks if DNS management is available.

```php
public function checkDnsAvailability($params)
{
    return [
        'available' => true,
        'premium' => false
    ];
}
```

### checkTransferAvailability

Checks if transfer is available for domain.

```php
public function checkTransferAvailability($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $response = $api->checkTransferEligibility([
        'domain' => $params['domain']
    ]);

    return [
        'transferable' => $response['eligible'],
        'reason' => $response['reason'] ?? null
    ];
}
```

### resendTransferEmail

Resends transfer authorization email.

```php
public function resendTransferEmail($params)
{
    $api = new RegistrarApiClient($this->getApiCredentials());

    $response = $api->resendTransferEmail([
        'domain' => $params['domain']
    ]);

    return [
        'success' => $response['success'],
        'error' => $response['error_message'] ?? null
    ];
}
```

## Helper Methods

### API Client Base Class

```php
abstract class RegistrarApiClient
{
    protected $credentials;
    protected $testMode;
    protected $logger;

    public function __construct(array $credentials)
    {
        $this->credentials = $credentials;
        $this->testMode = $credentials['test_mode'] ?? false;
    }

    protected function request(string $endpoint, array $data, string $method = 'POST')
    {
        $url = $this->getBaseUrl() . $endpoint;

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'Authorization: Bearer ' . $this->credentials['api_key']
            ]
        ]);

        if ($method !== 'GET') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return $this->parseResponse($response, $httpCode);
    }

    abstract protected function getBaseUrl(): string;
    abstract protected function parseResponse(string $response, int $httpCode): array;
}
```

### Data Validation

```php
protected function validateRegistrantData(array $data): array
{
    $errors = [];

    if (empty($data['firstname']) || empty($data['lastname'])) {
        $errors[] = 'First and last name are required';
    }

    if (empty($data['email']) || !filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
        $errors[] = 'Valid email is required';
    }

    if (empty($data['address1'])) {
        $errors[] = 'Address is required';
    }

    if (empty($data['city'])) {
        $errors[] = 'City is required';
    }

    if (empty($data['country'])) {
        $errors[] = 'Country is required';
    }

    if (empty($data['postcode'])) {
        $errors[] = 'Postal code is required';
    }

    if (empty($data['phone'])) {
        $errors[] = 'Phone number is required';
    }

    return [
        'valid' => empty($errors),
        'errors' => $errors
    ];
}
```

## See Also

- [EPP Interface](./whmcs-epp-interface.md)
- [Sync Daemon](./whmcs-sync-daemon.md)
- [Module Development Guide](../developer/registrar-modules.md)
