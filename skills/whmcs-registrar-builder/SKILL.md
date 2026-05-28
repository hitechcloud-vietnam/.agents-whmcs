# WHMCS Registrar Module Builder Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building WHMCS domain registrar modules for domain registration, transfer, and management.

## When to Use

- Creating new domain registrar integration
- Adding support for new TLD extensions
- Building EPP-based or REST API registrar

## Registrar Structure

```
modules/registrars/{module}/
├── {module}.php              ← Main registrar file
├── lib/
│   ├── ApiClient.php        ← EPP/REST client
│   └── EppConnection.php     ← EPP connection handler
├── templates/
│   └── .gitkeep
└── logo.png
```

## Building Steps

### Step 1: Create API Client

```php
<?php
// lib/ApiClient.php
namespace Registrar;

class ApiClient {
    private string $apiUrl;
    private string $username;
    private string $password;
    private string $registrarName;
    private bool $testMode;

    public function __construct(array $params) {
        $this->apiUrl = $params['eppUrl'];
        $this->username = $params['Username'];
        $this->password = $params['Password'];
        $this->registrarName = $params['RegistrarName'];
        $this->testMode = $params['TestMode'] ?? false;
    }

    public function request(string $command, array $data = []): array {
        $xml = $this->buildXml($command, $data);

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->apiUrl,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $xml,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_HTTPHEADER => ['Content-Type: application/xml'],
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return $this->parseResponse($response);
    }

    private function buildXml(string $command, array $data): string {
        $xml = '<?xml version="1.0" encoding="UTF-8"?>';
        $xml .= '<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">';
        $xml .= '<command>';
        $xml .= '<' . $command . ' xmlns="urn:ietf:params:xml:ns:domain-1.0">';
        foreach ($data as $key => $value) {
            $xml .= '<' . $key . '>' . htmlspecialchars($value) . '</' . $key . '>';
        }
        $xml .= '</' . $command . '>';
        $xml .= '<clTRID>WHMc-' . time() . '</clTRID>';
        $xml .= '</command></epp>';
        return $xml;
    }

    private function parseResponse(string $xml): array {
        $doc = simplexml_load_string($xml);
        if (!$doc) return ['error' => 'Parse error'];

        $result = $doc->xpath('//result');
        $code = (int) ($result[0]['code'] ?? 500);

        if ($code >= 1000 && $code < 2000) {
            return ['success' => true];
        }

        $msg = $doc->xpath('//result/msg');
        return ['error' => (string) ($msg[0] ?? 'Unknown error')];
    }

    public function createDomain(array $params): array {
        return $this->request('create', [
            'name' => $params['domain'],
            'period' => $params['period'],
            'registrant' => $params['registrant'],
        ]);
    }

    public function transferDomain(array $params): array {
        return $this->request('transfer', [
            'name' => $params['domain'],
            'authCode' => $params['authCode'],
        ]);
    }

    public function renewDomain(array $params): array {
        return $this->request('renew', [
            'name' => $params['domain'],
            'period' => $params['period'],
        ]);
    }

    public function infoDomain(string $domain): array {
        $result = $this->request('info', ['name' => $domain]);
        return $result;
    }

    public function updateDomain(string $domain, array $changes): array {
        return $this->request('update', array_merge(['name' => $domain], $changes));
    }

    public function deleteDomain(string $domain): array {
        return $this->request('delete', ['name' => $domain]);
    }

    public function getNameservers(string $domain): array {
        return ['ns1' => '', 'ns2' => ''];
    }

    public function setNameservers(string $domain, array $nameservers): array {
        return ['success' => true];
    }

    public function getContacts(string $domain): array {
        return ['Registrant' => [], 'Admin' => [], 'Tech' => []];
    }

    public function setContacts(string $domain, array $contacts): array {
        return ['success' => true];
    }

    public function getLockStatus(string $domain): bool {
        return false;
    }

    public function setLockStatus(string $domain, bool $locked): array {
        return ['success' => true];
    }

    public function getAuthCode(string $domain): string {
        return '';
    }
}
```

### Step 2: Create Registrar Module

```php
<?php
// modules/registrars/{module}/{module}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {module}_getConfigArray(): array {
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => '{Registrar Name}'],
        'Description' => ['Type' => 'System', 'Value' => '{Description}'],
        'eppUrl' => [
            'FriendlyName' => 'EPP/REST URL',
            'Type' => 'text',
            'Size' => '50',
            'Description' => 'Server endpoint',
        ],
        'RegistrarName' => [
            'FriendlyName' => 'Registrar Name (ROID)',
            'Type' => 'text',
            'Size' => '30',
        ],
        'Username' => ['FriendlyName' => 'Username', 'Type' => 'text', 'Size' => '20'],
        'Password' => ['FriendlyName' => 'Password', 'Type' => 'password', 'Size' => '30'],
        'TestMode' => ['FriendlyName' => 'Test Mode', 'Type' => 'yesno'],
    ];
}

function {module}_RegisterDomain(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->createDomain([
            'domain' => $params['domain'],
            'period' => $params['regperiod'],
            'registrant' => formatRegistrant($params['contactdetails']['Registrant']),
        ]);

        return isset($result['error']) ? ['error' => $result['error']] : ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {module}_TransferDomain(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->transferDomain([
            'domain' => $params['domain'],
            'authCode' => $params['transfersecret'],
        ]);

        return isset($result['error']) ? ['error' => $result['error']] : ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {module}_RenewDomain(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->renewDomain([
            'domain' => $params['domain'],
            'period' => $params['regperiod'],
        ]);

        return isset($result['error']) ? ['error' => $result['error']] : ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

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

function {module}_SaveNameservers(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->setNameservers($params['domain'], [
            'ns1' => $params['ns1'],
            'ns2' => $params['ns2'],
            'ns3' => $params['ns3'] ?? '',
            'ns4' => $params['ns4'] ?? '',
        ]);

        return isset($result['error']) ? ['error' => $result['error']] : ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {module}_GetContactDetails(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        return $api->getContacts($params['domain']);
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {module}_SaveContactDetails(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->setContacts($params['domain'], $params['contactdetails']);

        return isset($result['error']) ? ['error' => $result['error']] : ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {module}_GetRegistrarLock(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        return ['lockstatus' => $api->getLockStatus($params['domain']) ? 'locked' : 'unlocked'];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {module}_SaveRegistrarLock(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->setLockStatus($params['domain'], $params['lockenabled'] === 'locked');

        return isset($result['error']) ? ['error' => $result['error']] : ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {module}_GetEPPCodes(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        return ['eppcode' => $api->getAuthCode($params['domain'])];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {module}_Sync(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->infoDomain($params['domain']);

        return [
            'status' => mapStatus($result['status'] ?? ''),
            'expiry' => $result['expiry'] ?? '',
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {module}_RequestDelete(array $params): array {
    try {
        $api = new \Registrar\ApiClient($params);
        $result = $api->deleteDomain($params['domain']);

        return isset($result['error']) ? ['error' => $result['error']] : ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function formatRegistrant(array $contact): array {
    return [
        'name' => $contact['fullname'] ?? '',
        'org' => $contact['companyname'] ?? '',
        'email' => $contact['email'] ?? '',
        'voice' => formatPhone($contact['phonenumber'] ?? ''),
        'address1' => $contact['address1'] ?? '',
        'city' => $contact['city'] ?? '',
        'country' => $contact['country'] ?? '',
    ];
}

function formatPhone(string $phone): string {
    $phone = preg_replace('/[^0-9]/', '', $phone);
    if (substr($phone, 0, 1) === '0') $phone = substr($phone, 1);
    return '+' . $phone;
}

function mapStatus(string $status): string {
    $status = strtolower($status);
    if (in_array($status, ['ok', 'active'])) return 'Active';
    if (in_array($status, ['pending transfer'])) return 'Pending Transfer';
    return 'Active';
}
```

## Checklist

- [ ] getConfigArray() with all settings
- [ ] RegisterDomain() / TransferDomain() / RenewDomain()
- [ ] GetNameservers() / SaveNameservers()
- [ ] GetContactDetails() / SaveContactDetails()
- [ ] GetRegistrarLock() / SaveRegistrarLock()
- [ ] GetEPPCodes() returns ['eppcode' => '...']
- [ ] Sync() returns ['status' => '', 'expiry' => '']
- [ ] RequestDelete()
- [ ] ApiClient with EPP/REST support

---

**Related Skills:**
- whmcs-epp-protocol
- whmcs-domain-sync
- whmcs-registrar-testing