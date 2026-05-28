# WHMCS Registrar Module Workflow
# Version: 1.0 | Created: 2026-05-28

---

## Overview

This workflow guides the creation of WHMCS domain registrar modules for domain registration, transfer, and management.

## Prerequisites

1. Read `.agents-whmcs/CLAUDE.md` (Technical Reference)
2. Read `Core_exapm_whmcs/sample-registrar-module/`
3. Identify EPP (Extensible Provisioning Protocol) or REST API documentation

---

## Module Structure

```
modules/registrars/{module}/
├── {module}.php              ← Main registrar file
├── lib/
│   └── ApiClient.php        ← EPP/REST API client
├── templates/
│   └── .gitkeep             ← (if needed)
├── hooks.php                ← Hooks (optional)
└── logo.png                 ← 80x80px logo
```

---

## Step-by-Step Development

### Step 1: Create Main Registrar File

Create `{module}.php`:

```php
<?php
/**
 * {Module} Domain Registrar Module for WHMCS
 * Version: 1.0.0
 * Author: HiTechCloud
 */

use WHMCS\Database\Capsule;

if (!defined("WHMCS")) {
    die("Direct access denied");
}

/**
 * Get Registrar Configuration
 */
function {module}_getConfigArray(): array {
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => '{Registrar Name}',
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => '{Description of registrar}',
        ],
        'eppUrl' => [
            'FriendlyName' => 'EPP/REST URL',
            'Type' => 'text',
            'Size' => '50',
            'Description' => 'EPP server URL or REST API endpoint',
        ],
        'RegistrarName' => [
            'FriendlyName' => 'Registrar Name (ROID)',
            'Type' => 'text',
            'Size' => '30',
        ],
        'Username' => [
            'FriendlyName' => 'Username',
            'Type' => 'text',
            'Size' => '20',
        ],
        'Password' => [
            'FriendlyName' => 'Password',
            'Type' => 'password',
            'Size' => '30',
        ],
        'TestMode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
            'Description' => 'Enable for testing',
        ],
    ];
}

/**
 * Register Domain
 */
function {module}_RegisterDomain(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->registerDomain([
            'domain' => $params['domain'],
            ' registrant' => formatRegistrant($params['contactdetails']['Registrant']),
            'period' => $params['regperiod'],
        ]);

        if (isset($result['error'])) {
            return ['error' => $result['error']];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Transfer Domain
 */
function {module}_TransferDomain(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->requestTransfer([
            'domain' => $params['domain'],
            'authCode' => $params['transfersecret'],
            'registrant' => formatRegistrant($params['contactdetails']['Registrant']),
        ]);

        if (isset($result['error'])) {
            return ['error' => $result['error']];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Renew Domain
 */
function {module}_RenewDomain(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->renewDomain([
            'domain' => $params['domain'],
            'period' => $params['regperiod'],
        ]);

        if (isset($result['error'])) {
            return ['error' => $result['error']];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Get Nameservers
 */
function {module}_GetNameservers(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->getNameservers($params['domain']);

        return [
            'ns1' => $result['ns1'] ?? '',
            'ns2' => $result['ns2'] ?? '',
            'ns3' => $result['ns3'] ?? '',
            'ns4' => $result['ns4'] ?? '',
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Save Nameservers
 */
function {module}_SaveNameservers(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->setNameservers($params['domain'], [
            'ns1' => $params['ns1'],
            'ns2' => $params['ns2'],
            'ns3' => $params['ns3'] ?? '',
            'ns4' => $params['ns4'] ?? '',
        ]);

        if (isset($result['error'])) {
            return ['error' => $result['error']];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Get Contact Details
 */
function {module}_GetContactDetails(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->getContacts($params['domain']);

        return [
            'Registrant' => $result['Registrant'] ?? [],
            'Admin' => $result['Admin'] ?? [],
            'Tech' => $result['Tech'] ?? [],
            'Billing' => $result['Billing'] ?? [],
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Save Contact Details
 */
function {module}_SaveContactDetails(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->setContacts($params['domain'], $params['contactdetails']);

        if (isset($result['error'])) {
            return ['error' => $result['error']];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Get Registrar Lock Status
 */
function {module}_GetRegistrarLock(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->getLockStatus($params['domain']);

        return [
            'lockstatus' => ($result['locked'] ?? false) ? 'locked' : 'unlocked',
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Save Registrar Lock
 */
function {module}_SaveRegistrarLock(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->setLockStatus($params['domain'], $params['lockenabled'] === 'locked');

        if (isset($result['error'])) {
            return ['error' => $result['error']];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Get EPP Code
 */
function {module}_GetEPPCodes(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->getAuthCode($params['domain']);

        return [
            'eppcode' => $result['eppcode'] ?? '',
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Sync Domain Status
 */
function {module}_Sync(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->getDomainInfo($params['domain']);

        return [
            'status' => mapDomainStatus($result['status']),
            'expiry' => $result['expiry'] ?? '',
            'registrationstatus' => $result['active'] ?? true,
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Request Delete
 */
function {module}_RequestDelete(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->requestDelete($params['domain']);

        if (isset($result['error'])) {
            return ['error' => $result['error']];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

// Helper functions

function formatRegistrant(array $contact): array {
    return [
        'name' => $contact['fullname'] ?? '',
        'org' => $contact['companyname'] ?? '',
        'email' => $contact['email'] ?? '',
        'voice' => formatPhone($contact['phonenumber'] ?? ''),
        'fax' => formatPhone($contact['faxnumber'] ?? ''),
        'address1' => $contact['address1'] ?? '',
        'address2' => $contact['address2'] ?? '',
        'city' => $contact['city'] ?? '',
        'state' => $contact['state'] ?? '',
        'postcode' => $contact['postcode'] ?? '',
        'country' => $contact['country'] ?? '',
    ];
}

function formatPhone(string $phone): string {
    // Convert to international format: +84.XXX.XXXX.XXX
    $phone = preg_replace('/[^0-9]/', '', $phone);
    if (substr($phone, 0, 1) === '0') {
        $phone = substr($phone, 1);
    }
    return '+' . $phone;
}

function mapDomainStatus(string $status): string {
    $status = strtolower($status);
    if (in_array($status, ['ok', 'active'])) {
        return 'Active';
    }
    if (in_array($status, ['expired', 'pending delete'])) {
        return 'Expired';
    }
    if (in_array($status, ['pending transfer', 'transfer pending'])) {
        return 'Pending Transfer';
    }
    return 'Active';
}
```

### Step 2: Create API Client

Create `lib/ApiClient.php`:

```php
<?php
namespace Registrar;

class ApiClient {
    private $apiUrl;
    private $username;
    private $password;
    private $registrarName;
    private $testMode;

    public function __construct(array $params) {
        $this->apiUrl = $params['eppUrl'];
        $this->username = $params['Username'];
        $this->password = $params['Password'];
        $this->registrarName = $params['RegistrarName'];
        $this->testMode = $params['TestMode'] ?? false;
    }

    public function request(string $method, string $command, array $data = []): array {
        $ch = curl_init();

        $xml = $this->buildEppXml($command, $data);

        curl_setopt_array($ch, [
            CURLOPT_URL => $this->apiUrl,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $xml,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/xml',
            ],
        ]);

        $response = curl_exec($ch);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('EPP Error: ' . $error);
        }

        return $this->parseEppResponse($response);
    }

    private function buildEppXml(string $command, array $data): string {
        $xml = '<?xml version="1.0" encoding="UTF-8"?>';
        $xml .= '<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">';
        $xml .= '<command>';
        $xml .= '<' . $command . '>';

        foreach ($data as $key => $value) {
            $xml .= '<' . $key . '>' . htmlspecialchars($value) . '</' . $key . '>';
        }

        $xml .= '</' . $command . '>';
        $xml .= '<clTRID>WHMc-' . time() . '</clTRID>';
        $xml .= '</command>';
        $xml .= '</epp>';

        return $xml;
    }

    private function parseEppResponse(string $xml): array {
        libxml_use_internal_errors(true);
        $doc = simplexml_load_string($xml);

        if ($doc === false) {
            throw new \Exception('Failed to parse EPP response');
        }

        // Parse response code
        $result = $doc->xpath('//epp/response/result');
        $code = (int) ($result[0]['code'] ?? 500);

        if ($code >= 200 && $code < 300) {
            return ['success' => true];
        }

        $msg = $doc->xpath('//epp/response/result/msg');
        return ['error' => (string) ($msg[0] ?? 'Unknown error')];
    }

    // Domain operations

    public function registerDomain(array $data): array {
        return $this->request('create', 'domain:create', $data);
    }

    public function requestTransfer(array $data): array {
        return $this->request('transfer', 'domain:transfer', $data);
    }

    public function renewDomain(array $data): array {
        return $this->request('renew', 'domain:renew', $data);
    }

    public function getNameservers(string $domain): array {
        $result = $this->request('info', 'domain:info', ['name' => $domain]);
        // Parse and return nameservers
        return ['ns1' => '', 'ns2' => ''];
    }

    public function setNameservers(string $domain, array $nameservers): array {
        return $this->request('update', 'domain:update', [
            'name' => $domain,
            'ns' => implode(' ', $nameservers),
        ]);
    }

    public function getContacts(string $domain): array {
        // Return contact details
        return [];
    }

    public function setContacts(string $domain, array $contacts): array {
        return ['success' => true];
    }

    public function getLockStatus(string $domain): array {
        return ['locked' => false];
    }

    public function setLockStatus(string $domain, bool $locked): array {
        return ['success' => true];
    }

    public function getAuthCode(string $domain): array {
        return ['eppcode' => ''];
    }

    public function getDomainInfo(string $domain): array {
        return ['status' => 'ok', 'expiry' => ''];
    }

    public function requestDelete(string $domain): array {
        return $this->request('delete', 'domain:delete', ['name' => $domain]);
    }
}
```

---

## Checklist

- [ ] getConfigArray() with all settings
- [ ] RegisterDomain() - Return success or error array
- [ ] TransferDomain() - With auth code
- [ ] RenewDomain() - With period
- [ ] GetNameservers() / SaveNameservers()
- [ ] GetContactDetails() / SaveContactDetails()
- [ ] GetRegistrarLock() / SaveRegistrarLock()
- [ ] GetEPPCodes() - Return ['eppcode' => '...']
- [ ] Sync() - Return ['status' => '', 'expiry' => '']
- [ ] RequestDelete()
- [ ] API Client with EPP or REST support

---

## Common Issues

| Issue | Solution |
|-------|----------|
| Transfer failed | Check auth code / EPP secret |
| Status not syncing | Check Sync() return format |
| Contact save failed | Verify all required fields |
| EPP connection timeout | Check firewall and SSL |

---

## WHOIS Contact Format

Registrant contact array structure:
```php
[
    'fullname' => 'John Doe',
    'companyname' => 'Company Ltd',
    'email' => 'john@example.com',
    'phonenumber' => '84.123.456.789',
    'address1' => '123 Main St',
    'address2' => '',
    'city' => 'Ho Chi Minh City',
    'state' => 'Ho Chi Minh',
    'postcode' => '70000',
    'country' => 'VN',
]
```

---

Last updated: 2026-05-28