# WHMCS Registrar Module Creation

## Overview

This workflow guides you through creating a domain registrar module for WHMCS. Registrar modules enable domain registration, transfer, renewal, and management through various domain providers.

## Prerequisites

- WHMCS v8.0+
- PHP 7.4+ with required extensions
- Domain provider API documentation
- Registry credentials (API key, reseller ID)
- Test account for sandbox testing

## Step-by-Step Instructions

### Step 1: Create Module Directory Structure

```
/modules/registrars/YourRegistrar/
    ├── yourregistrar.php        # Main module file
    ├── yourregistrar_api.php     # API communication class
    ├── whois.php                # WHOIS server config (optional)
    ├── dnszones.php             # DNS zone management (optional)
    └── lang/
        └── english.php
```

### Step 2: Create the Main Registrar Module

Create `yourregistrar.php`:

```php
<?php
/**
 * WHMCS Registrar Module - YourRegistrar
 *
 * @copyright Copyright (c) 2024 Your Name
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Registrar module configuration.
 *
 * @return array
 */
function YourRegistrar_MetaData()
{
    return [
        'DisplayName' => 'Your Registrar Name',
        'APIVersion' => '1.1',
        'RequiresServer' => false,
        'DefaultPorts' => [],
        'ListOfTlds' => '.com,.net,.org,.info,.biz,.io,.co',
    ];
}

/**
 * Configuration fields.
 *
 * @return array
 */
function YourRegistrar_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Your Registrar Name',
        ],
        'apiKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Description' => 'Your registrar API key',
        ],
        'apiUsername' => [
            'FriendlyName' => 'API Username',
            'Type' => 'text',
            'Description' => 'Reseller/Username for API',
        ],
        'sandbox' => [
            'FriendlyName' => 'Sandbox Mode',
            'Type' => 'yesno',
            'Description' => 'Enable sandbox for testing',
            'Default' => 'yes',
        ],
        'registrarLock' => [
            'FriendlyName' => 'Registrar Lock',
            'Type' => 'yesno',
            'Description' => 'Enable registrar lock by default',
            'Default' => 'yes',
        ],
        'emailTransfer' => [
            'FriendlyName' => 'Email Transfer',
            'Type' => 'yesno',
            'Description' => 'Allow email-based transfer approval',
            'Default' => 'yes',
        ],
    ];
}

/**
 * Register domain.
 *
 * @param array $params
 * @return array
 */
function YourRegistrar_RegisterDomain($params)
{
    try {
        $api = new YourRegistrar\Api($params);

        $result = $api->registerDomain([
            'domain' => $params['sld'] . '.' . $params['tld'],
            ' registrant' => [
                'name' => $params['fullname'],
                'organization' => $params['companyname'],
                'email' => $params['email'],
                'address1' => $params['address1'],
                'address2' => $params['address2'],
                'city' => $params['city'],
                'state' => $params['state'],
                'postcode' => $params['postcode'],
                'country' => $params['country'],
                'phone' => $params['phone'],
                'phonecc' => $params['phonecc'],
            ],
            'years' => $params['regperiod'],
            'nameservers' => [
                ['nameserver' => $params['ns1']],
                ['nameserver' => $params['ns2']],
                ['nameserver' => $params['ns3'] ?? null],
                ['nameserver' => $params['ns4'] ?? null],
            ],
        ]);

        if ($result['success']) {
            return [
                'success' => true,
                'domainid' => $result['domain_id'],
            ];
        }

        return [
            'success' => false,
            'error' => $result['message'],
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Transfer domain.
 *
 * @param array $params
 * @return array
 */
function YourRegistrar_TransferDomain($params)
{
    try {
        $api = new YourRegistrar\Api($params);

        $result = $api->transferDomain([
            'domain' => $params['sld'] . '.' . $params['tld'],
            'auth_code' => $params['transfersecret'],
            'registrant' => [
                'email' => $params['email'],
            ],
        ]);

        return [
            'success' => true,
            'transferid' => $result['transfer_id'],
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Renew domain.
 *
 * @param array $params
 * @return array
 */
function YourRegistrar_RenewDomain($params)
{
    try {
        $api = new YourRegistrar\Api($params);

        $result = $api->renewDomain([
            'domain' => $params['sld'] . '.' . $params['tld'],
            'years' => $params['regperiod'],
        ]);

        return [
            'success' => true,
            'expirydate' => $result['expiry_date'],
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Release domain.
 *
 * @param array $params
 * @return array
 */
function YourRegistrar_ReleaseDomain($params)
{
    try {
        $api = new YourRegistrar\Api($params);

        $result = $api->releaseDomain([
            'domain' => $params['sld'] . '.' . $params['tld'],
            'new_tag' => $params['newRegistrar'],
        ]);

        return ['success' => true];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Get nameservers.
 *
 * @param array $params
 * @return array
 */
function YourRegistrar_GetNameservers($params)
{
    try {
        $api = new YourRegistrar\Api($params);

        $result = $api->getNameservers($params['sld'] . '.' . $params['tld']);

        return [
            'success' => true,
            'ns1' => $result['nameservers'][0] ?? '',
            'ns2' => $result['nameservers'][1] ?? '',
            'ns3' => $result['nameservers'][2] ?? '',
            'ns4' => $result['nameservers'][3] ?? '',
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Save nameservers.
 *
 * @param array $params
 * @return array
 */
function YourRegistrar_SaveNameservers($params)
{
    try {
        $api = new YourRegistrar\Api($params);

        $result = $api->setNameservers($params['sld'] . '.' . $params['tld'], [
            $params['ns1'],
            $params['ns2'],
            $params['ns3'] ?? null,
            $params['ns4'] ?? null,
        ]);

        return ['success' => true];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Get domain contact details.
 *
 * @param array $params
 * @return array
 */
function YourRegistrar_GetContactDetails($params)
{
    try {
        $api = new YourRegistrar\Api($params);

        $result = $api->getContactDetails($params['sld'] . '.' . $params['tld']);

        return [
            'success' => true,
            'ContactDetails' => $result['contacts'],
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->Message(),
        ];
    }
}

/**
 * Save domain contact details.
 *
 * @param array $params
 * @return array
 */
function YourRegistrar_SaveContactDetails($params)
{
    try {
        $api = new YourRegistrar\Api($params);

        $result = $api->setContactDetails($params['sld'] . '.' . $params['tld'], $params['ContactDetails']);

        return ['success' => true];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Enable/disable registrar lock.
 *
 * @param array $params
 * @return array
 */
function YourRegistrar_RegistrarLock($params)
{
    try {
        $api = new YourRegistrar\Api($params);

        $result = $api->setRegistrarLock($params['sld'] . '.' . $params['tld'], $params['lockenabled']);

        return ['success' => true];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Get domain EPP code / transfer key.
 *
 * @param array $params
 * @return array
 */
function YourRegistrar_GetEPPCode($params)
{
    try {
        $api = new YourRegistrar\Api($params);

        $result = $api->getTransferKey($params['sld'] . '.' . $params['tld']);

        return [
            'success' => true,
            'eppcode' => $result['epp_code'],
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Domain synchronization.
 *
 * @param array $params
 * @return array
 */
function YourRegistrar_Sync($params)
{
    try {
        $api = new YourRegistrar\Api($params);

        $result = $api->getDomainStatus($params['sld'] . '.' . $params['tld']);

        return [
            'success' => true,
            'active' => $result['status'] === 'active',
            'expirydate' => $result['expiry_date'],
            'expired' => $result['expired'],
            'transferredAway' => $result['transferred'] ?? false,
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Transfer sync.
 *
 * @param array $params
 * @return array
 */
function YourRegistrar_TransferSync($params)
{
    try {
        $api = new YourRegistrar\Api($params);

        $result = $api->getTransferStatus($params['sld'] . '.' . $params['tld']);

        if ($result['status'] === 'completed') {
            return [
                'success' => true,
                'completed' => true,
            ];
        }

        if ($result['status'] === 'cancelled' || $result['status'] === 'rejected') {
            return [
                'success' => true,
                'completed' => false,
                'reason' => $result['reason'] ?? 'Transfer rejected',
            ];
        }

        return [
            'success' => true,
            'completed' => false,
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * WHOIS lookup.
 *
 * @param array $params
 * @return array
 */
function YourRegistrar_WHOISLookup($params)
{
    try {
        $api = new YourRegistrar\Api($params);

        $result = $api->whoisLookup($params['sld'] . '.' . $params['tld']);

        return [
            'success' => true,
            'whois' => $result['whois_data'],
            'available' => $result['available'],
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Get domain pricing.
 *
 * @param array $params
 * @return array
 */
function YourRegistrar_GetPricing($params)
{
    try {
        $api = new YourRegistrar\Api($params);

        $result = $api->getPricing();

        return [
            'success' => true,
            'pricing' => $result,
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}
```

### Step 3: Create API Class

Create `yourregistrar_api.php`:

```php
<?php

namespace YourRegistrar;

class Api
{
    private $apiKey;
    private $apiUsername;
    private $sandbox;
    private $apiEndpoint;

    public function __construct(array $params)
    {
        $this->apiKey = $params['apiKey'];
        $this->apiUsername = $params['apiUsername'];
        $this->sandbox = ($params['sandbox'] ?? false) === 'on';

        $this->apiEndpoint = $this->sandbox
            ? 'https://sandbox.yourregistrar.com/api/v1'
            : 'https://api.yourregistrar.com/v1';
    }

    private function request(string $endpoint, string $method = 'GET', array $data = []): array
    {
        $url = $this->apiEndpoint . '/' . ltrim($endpoint, '/');

        $ch = curl_init();

        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'X-Username: ' . $this->apiUsername,
                'Content-Type: application/json',
            ],
        ]);

        if ($method !== 'GET') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
            if (!empty($data)) {
                curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
            }
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $result = json_decode($response, true) ?? [];

        if ($httpCode >= 400) {
            throw new \Exception($result['message'] ?? 'API Error');
        }

        return $result;
    }

    public function registerDomain(array $data): array
    {
        return $this->request('domains/register', 'POST', $data);
    }

    public function transferDomain(array $data): array
    {
        return $this->request('domains/transfer', 'POST', $data);
    }

    public function renewDomain(array $data): array
    {
        return $this->request('domains/renew', 'POST', $data);
    }

    public function releaseDomain(array $data): array
    {
        return $this->request('domains/release', 'POST', $data);
    }

    public function getNameservers(string $domain): array
    {
        return $this->request("domains/{$domain}/nameservers");
    }

    public function setNameservers(string $domain, array $nameservers): array
    {
        return $this->request("domains/{$domain}/nameservers", 'PUT', [
            'nameservers' => array_filter($nameservers),
        ]);
    }

    public function getContactDetails(string $domain): array
    {
        return $this->request("domains/{$domain}/contacts");
    }

    public function setContactDetails(string $domain, array $contacts): array
    {
        return $this->request("domains/{$domain}/contacts", 'PUT', $contacts);
    }

    public function setRegistrarLock(string $domain, bool $lock): array
    {
        return $this->request("domains/{$domain}/lock", $lock ? 'POST' : 'DELETE');
    }

    public function getTransferKey(string $domain): array
    {
        return $this->request("domains/{$domain}/auth-code");
    }

    public function getDomainStatus(string $domain): array
    {
        return $this->request("domains/{$domain}/status");
    }

    public function getTransferStatus(string $domain): array
    {
        return $this->request("domains/{$domain}/transfer");
    }

    public function whoisLookup(string $domain): array
    {
        return $this->request("domains/{$domain}/whois");
    }

    public function getPricing(): array
    {
        return $this->request('pricing');
    }
}
```

### Step 4: Install and Configure

1. Upload to `/modules/registrars/YourRegistrar/`
2. Go to Configuration > System > Domain Registrars
3. Activate and configure module
4. Test connectivity

## Expected Outcomes

- Registrar appears in domain registrar list
- Domain registration creates account at provider
- Transfers initiate with auth code
- Renewals extend registration period
- DNS changes sync to registry

## Testing Checklist

- [ ] Module activates without errors
- [ ] Domain registration works
- [ ] Transfer initiates correctly
- [ ] Renewal extends expiry date
- [ ] Nameserver changes update
- [ ] Registrar lock toggles
- [ ] EPP code retrieval works
- [ ] WHOIS lookup returns data
- [ ] Sync updates domain status
- [ ] Error handling works properly
