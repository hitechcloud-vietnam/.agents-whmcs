# WHMCS Registrar Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/registrar-module/
├── registrar.php          # Main registrar module
├── lib/
│   └── ApiClient.php     # API client skeleton
└── DEVKIT.md            # This file
```

## Template

```php
<?php
/**
 * WHMCS Registrar Module: {registrar}
 * Domain Registrar Template
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function {registrar}_getConfigArray(): array {
    return [
        'Username' => [
            'FriendlyName' => 'API Username',
            'Type' => 'text',
            'Size' => '30',
        ],
        'Password' => [
            'FriendlyName' => 'API Password',
            'Type' => 'password',
            'Size' => '30',
        ],
        'APIKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50',
        ],
        'TestMode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
        ],
    ];
}

function {registrar}_RegisterDomain(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->registerDomain([
            'domain' => $params['domain'],
            'years' => $params['regperiod'],
            'registrant' => buildRegistrantData($params),
        ]);

        if (!$result['success']) {
            return ['error' => $result['message'] ?? 'Registration failed'];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {registrar}_TransferDomain(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->transferDomain([
            'domain' => $params['domain'],
            'auth_code' => $params['eppcode'],
        ]);

        if (!$result['success']) {
            return ['error' => $result['message'] ?? 'Transfer failed'];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {registrar}_RenewDomain(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->renewDomain([
            'domain' => $params['domain'],
            'years' => $params['regperiod'],
        ]);

        if (!$result['success']) {
            return ['error' => $result['message'] ?? 'Renewal failed'];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {registrar}_GetNameservers(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->getNameservers($params['domain']);

        return $result;
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {registrar}_SaveNameservers(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->saveNameservers($params['domain'], [
            'ns1' => $params['ns1'],
            'ns2' => $params['ns2'],
            'ns3' => $params['ns3'] ?? '',
            'ns4' => $params['ns4'] ?? '',
        ]);

        if (!$result['success']) {
            return ['error' => $result['message'] ?? 'Failed to save nameservers'];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {registrar}_GetContactDetails(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->getContactDetails($params['domain']);

        return $result;
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {registrar}_SaveContactDetails(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->saveContactDetails($params['domain'], [
            'Registrant' => buildRegistrantData($params),
            'Admin' => buildContactData($params, 'admin'),
            'Tech' => buildContactData($params, 'tech'),
            'Billing' => buildContactData($params, 'billing'),
        ]);

        if (!$result['success']) {
            return ['error' => $result['message'] ?? 'Failed to save contacts'];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {registrar}_GetRegistrarLock(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->getRegistrarLock($params['domain']);

        return ['lockstatus' => $result['locked'] ? 'locked' : 'unlocked'];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {registrar}_SaveRegistrarLock(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->saveRegistrarLock($params['domain'], $params['lockenabled']);

        if (!$result['success']) {
            return ['error' => $result['message'] ?? 'Failed to update lock'];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {registrar}_GetDNS(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->getDNS($params['domain']);

        return $result;
    } catch (\Exception $e) {
        return [['error' => $e->getMessage()]];
    }
}

function {registrar}_SaveDNS(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->saveDNS($params['domain'], $params['dnsrecords']);

        if (!$result['success']) {
            return ['error' => $result['message'] ?? 'Failed to save DNS'];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {registrar}_GetEPPCodes(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->getEPPCode($params['domain']);

        return ['eppcode' => $result['code']];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {registrar}_RegisterNameserver(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->registerNameserver(
            $params['domain'],
            $params['ns'],
            $params['ip']
        );

        if (!$result['success']) {
            return ['error' => $result['message'] ?? 'Failed to register nameserver'];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {registrar}_ModifyNameserver(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->modifyNameserver(
            $params['domain'],
            $params['ns'],
            $params['newip']
        );

        if (!$result['success']) {
            return ['error' => $result['message'] ?? 'Failed to modify nameserver'];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {registrar}_DeleteNameserver(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->deleteNameserver($params['domain'], $params['ns']);

        if (!$result['success']) {
            return ['error' => $result['message'] ?? 'Failed to delete nameserver'];
        }

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function {registrar}_Sync(array $params): array {
    try {
        $api = new \{Registrar}\ApiClient($params);

        $result = $api->syncDomain($params['domain']);

        return [
            'status' => $result['status'],
            'expiry' => $result['expiry_date'],
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

// Helper Functions
function buildRegistrantData(array $params): array {
    return [
        'firstname' => $params['fullname'] ?? '',
        'lastname' => '',
        'organization' => $params['companyname'] ?? '',
        'email' => $params['email'] ?? '',
        'address1' => $params['address1'] ?? '',
        'address2' => $params['address2'] ?? '',
        'city' => $params['city'] ?? '',
        'state' => $params['state'] ?? '',
        'postcode' => $params['postcode'] ?? '',
        'country' => $params['country'] ?? '',
        'phonenumber' => $params['phonenumber'] ?? '',
    ];
}

function buildContactData(array $params, string $type): array {
    $prefix = $type . '_';
    return [
        'firstname' => $params[$prefix . 'firstname'] ?? '',
        'lastname' => $params[$prefix . 'lastname'] ?? '',
        'organization' => $params[$prefix . 'organization'] ?? '',
        'email' => $params[$prefix . 'email'] ?? '',
        'address1' => $params[$prefix . 'address1'] ?? '',
        'city' => $params[$prefix . 'city'] ?? '',
        'state' => $params[$prefix . 'state'] ?? '',
        'postcode' => $params[$prefix . 'postcode'] ?? '',
        'country' => $params[$prefix . 'country'] ?? '',
        'phonenumber' => $params[$prefix . 'phonenumber'] ?? '',
    ];
}
```

## API Client Skeleton

```php
<?php
namespace {Registrar};

class ApiClient {
    private string $apiUrl;
    private string $username;
    private string $password;
    private string $apiKey;

    public function __construct(array $params) {
        $this->username = $params['Username'] ?? '';
        $this->password = $params['Password'] ?? '';
        $this->apiKey = $params['APIKey'] ?? '';
        $this->apiUrl = $params['TestMode'] === 'on'
            ? 'https://test.api.registrar.com'
            : 'https://api.registrar.com';
    }

    public function request(string $command, array $data = []): array {
        $data = array_merge($data, [
            'command' => $command,
            'username' => $this->username,
            'password' => $this->password,
        ]);

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->apiUrl,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($data),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        parse_str($response, $result);

        if ($result['result'] !== 'SUCCESS') {
            throw new \Exception($result['message'] ?? 'API Error');
        }

        return $result;
    }

    public function registerDomain(array $data): array {
        return $this->request('register_domain', $data);
    }

    public function transferDomain(array $data): array {
        return $this->request('transfer_domain', $data);
    }

    public function renewDomain(array $data): array {
        return $this->request('renew_domain', $data);
    }

    public function getNameservers(string $domain): array {
        $result = $this->request('get_nameservers', ['domain' => $domain]);
        return [
            'ns1' => $result['ns1'] ?? '',
            'ns2' => $result['ns2'] ?? '',
        ];
    }

    public function saveNameservers(string $domain, array $nameservers): array {
        return $this->request('save_nameservers', array_merge(['domain' => $domain], $nameservers));
    }

    public function getContactDetails(string $domain): array {
        return $this->request('get_contact_details', ['domain' => $domain]);
    }

    public function saveContactDetails(string $domain, array $contacts): array {
        return $this->request('save_contact_details', array_merge(['domain' => $domain], $contacts));
    }

    public function getRegistrarLock(string $domain): array {
        return $this->request('get_registrar_lock', ['domain' => $domain]);
    }

    public function saveRegistrarLock(string $domain, bool $enabled): array {
        return $this->request('save_registrar_lock', ['domain' => $domain, 'lock' => $enabled]);
    }

    public function getDNS(string $domain): array {
        return $this->request('get_dns', ['domain' => $domain]);
    }

    public function saveDNS(string $domain, array $records): array {
        return $this->request('save_dns', ['domain' => $domain, 'records' => $records]);
    }

    public function getEPPCode(string $domain): array {
        return $this->request('get_eppcode', ['domain' => $domain]);
    }

    public function syncDomain(string $domain): array {
        return $this->request('sync_domain', ['domain' => $domain]);
    }
}
```

## Checklist

```
Pre-Dev:
□ Get registrar API documentation
□ Get test/sandbox credentials
□ Understand EPP or REST API
□ Identify required fields for registrant data

Development:
□ Create ApiClient class
□ Implement registration functions
□ Implement transfer functions
□ Implement renewal functions
□ Implement nameserver functions
□ Implement contact functions
□ Implement DNS management
□ Implement lock management
□ Implement sync function
□ Implement nameserver management
□ Test all functions return ['success' => true] or ['error' => '...']

Testing:
□ Test with sandbox credentials
□ Test domain registration
□ Test transfer flow
□ Test renewal
□ Test nameserver changes
□ Test contact updates
□ Test DNSSEC toggle
□ Verify sync updates WHMCS status
```
