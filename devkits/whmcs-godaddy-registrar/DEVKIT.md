# WHMCS GoDaddy Registrar Module - DEVKIT

## Module Information
- **Name**: GoDaddy Registrar
- **Version**: 1.0.0
- **Type**: Registrar Module
- **Description**: GoDaddy domain registrar integration for WHMCS

## Installation
1. Copy to `/modules/registrars/godaddy/`
2. Activate via WHMCS Admin > Configuration > Domain Registrars

## godaddy.php
```php
<?php
/**
 * WHMCS GoDaddy Registrar Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function godaddy_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'GoDaddy'
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => 'GoDaddy Domain Registrar Integration'
        ],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '80'
        ],
        'api_secret' => [
            'FriendlyName' => 'API Secret',
            'Type' => 'password',
            'Size' => '80'
        ],
        'environment' => [
            'FriendlyName' => 'Environment',
            'Type' => 'dropdown',
            'Options' => [
                'production' => 'Production',
                'sandbox' => 'Sandbox'
            ],
            'Default' => 'production'
        ],
        'default_years' => [
            'FriendlyName' => 'Default Years',
            'Type' => 'text',
            'Size' => '5',
            'Default' => '1'
        ]
    ];
}

function godaddy_getApiUrl($params)
{
    return $params['environment'] == 'sandbox'
        ? 'https://api.ote-godaddy.com'
        : 'https://api.godaddy.com';
}

function godaddy_apiRequest($params, $method, $endpoint, $data = null)
{
    $baseUrl = godaddy_getApiUrl($params);
    $url = $baseUrl . $endpoint;
    
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: sso-key ' . $params['api_key'] . ':' . $params['api_secret'],
        'Content-Type: application/json'
    ]);
    
    if ($data) {
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
    }
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    return json_decode($response, true);
}

function godaddy_registerDomain($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $contactData = [
        'nameFirst' => $params['firstname'],
        'nameLast' => $params['lastname'],
        'email' => $params['email'],
        'phone' => $params['phonenumber'],
        'address1' => $params['address1'],
        'city' => $params['city'],
        'state' => $params['state'],
        'country' => $params['country'],
        'postalCode' => $params['postcode']
    ];
    
    $data = [
        'name' => $domain,
        'period' => (int)$params['regperiod'],
        'contacts' => [
            'registrant' => $contactData,
            'admin' => $contactData,
            'tech' => $contactData,
            'billing' => $contactData
        ]
    ];
    
    $result = godaddy_apiRequest($params, 'POST', '/v1/domains', $data);
    
    if (isset($result['transferAwayEligibility'])) {
        return ['success' => true, 'domain' => $domain];
    }
    
    return ['success' => false, 'error' => $result['message'] ?? 'Registration failed'];
}

function godaddy_transferDomain($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    $eppCode = $params['transfersecret'];
    
    $contactData = [
        'nameFirst' => $params['firstname'],
        'nameLast' => $params['lastname'],
        'email' => $params['email']
    ];
    
    $data = [
        'name' => $domain,
        'period' => (int)$params['regperiod'],
        'authCode' => $eppCode,
        'contacts' => [
            'registrant' => $contactData
        ]
    ];
    
    $result = godaddy_apiRequest($params, 'POST', '/v1/domains/transfer', $data);
    
    if (isset($result['transferIn'])) {
        return [
            'success' => true,
            'transferid' => $result['transferIn']['id'] ?? ''
        ];
    }
    
    return ['success' => false, 'error' => $result['message'] ?? 'Transfer failed'];
}

function godaddy_renewDomain($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $data = [
        'duration' => (int)$params['regperiod']
    ];
    
    $result = godaddy_apiRequest($params, 'POST', '/v1/domains/' . $domain . '/renew', $data);
    
    if (isset($result['expiresAt'])) {
        return ['success' => true];
    }
    
    return ['success' => false, 'error' => $result['message'] ?? 'Renewal failed'];
}

function godaddy_getNameservers($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $result = godaddy_apiRequest($params, 'GET', '/v1/domains/' . $domain);
    
    if (isset($result['nameServers'])) {
        $nsList = $result['nameServers'];
        return [
            'success' => true,
            'ns1' => $nsList[0] ?? '',
            'ns2' => $nsList[1] ?? '',
            'ns3' => $nsList[2] ?? '',
            'ns4' => $nsList[3] ?? ''
        ];
    }
    
    return ['success' => false, 'error' => 'Could not fetch nameservers'];
}

function godaddy_saveNameservers($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $nameservers = array_filter([
        $params['ns1'],
        $params['ns2'],
        $params['ns3'],
        $params['ns4'],
        $params['ns5']
    ]);
    
    $data = ['nameServers' => $nameservers];
    
    $result = godaddy_apiRequest($params, 'PATCH', '/v1/domains/' . $domain, $data);
    
    return ['success' => !isset($result['code'])];
}

function godaddy_getDomainInfo($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $result = godaddy_apiRequest($params, 'GET', '/v1/domains/' . $domain);
    
    if (isset($result['expiresAt'])) {
        return [
            'registrationdate' => $result['createdAt'] ?? '',
            'expirydate' => $result['expiresAt'] ?? '',
            'autorenew' => $result['renewable'] ?? false,
            'locked' => $result['locked'] ?? false,
            'dnssec' => false
        ];
    }
    
    return ['success' => false];
}

function godaddy_sync($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $result = godaddy_apiRequest($params, 'GET', '/v1/domains/' . $domain);
    
    if (isset($result['expiresAt'])) {
        return [
            'expirydate' => date('Y-m-d', strtotime($result['expiresAt'])),
            'registrationdate' => date('Y-m-d', strtotime($result['createdAt'])),
            'autorenew' => $result['renewable'] ?? false,
            'locked' => $result['locked'] ?? false
        ];
    }
    
    return ['error' => 'Could not sync domain'];
}

function godaddy_requestDelete($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $result = godaddy_apiRequest($params, 'DELETE', '/v1/domains/' . $domain);
    
    return ['success' => true];
}

function godaddy_getContactDetails($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $result = godaddy_apiRequest($params, 'GET', '/v1/domains/' . $domain . '/contacts');
    
    if (isset($result['registrant'])) {
        return [
            'Registrant' => [
                'First Name' => $result['registrant']['nameFirst'] ?? '',
                'Last Name' => $result['registrant']['nameLast'] ?? '',
                'Email' => $result['registrant']['email'] ?? '',
                'Company' => $result['registrant']['organization'] ?? '',
                'Address 1' => $result['registrant']['address1'] ?? '',
                'City' => $result['registrant']['city'] ?? '',
                'State' => $result['registrant']['state'] ?? '',
                'Postcode' => $result['registrant']['postalCode'] ?? '',
                'Country' => $result['registrant']['country'] ?? '',
                'Phone' => $result['registrant']['phone'] ?? ''
            ]
        ];
    }
    
    return ['success' => false];
}

function godaddy_saveContactDetails($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $contactData = [
        'registrant' => [
            'nameFirst' => $params['firstname'],
            'nameLast' => $params['lastname'],
            'email' => $params['email'],
            'phone' => $params['phonenumber'],
            'address1' => $params['address1'],
            'city' => $params['city'],
            'state' => $params['state'],
            'country' => $params['country'],
            'postalCode' => $params['postcode']
        ]
    ];
    
    $result = godaddy_apiRequest($params, 'PATCH', '/v1/domains/' . $domain . '/contacts', $contactData);
    
    return ['success' => !isset($result['code'])];
}
```

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('DomainTransferCompleted', 1, function($vars) {
    logActivity("GoDaddy transfer completed for: " . $vars['domain']);
});

add_hook('DomainRenewed', 1, function($vars) {
    logActivity("GoDaddy renewal completed for: " . $vars['domain']);
});
```