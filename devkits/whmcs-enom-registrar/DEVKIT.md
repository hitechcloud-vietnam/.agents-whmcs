# WHMCS Enom Registrar Module - DEVKIT

## Module Information
- **Name**: eNom Registrar
- **Version**: 1.0.0
- **Type**: Registrar Module
- **Description**: eNom domain registrar integration for WHMCS

## Installation
1. Copy to `/modules/registrars/enom/`
2. Activate via WHMCS Admin > Configuration > Domain Registrars

## enom.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function enom_config()
{
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => 'eNom'],
        'username' => ['FriendlyName' => 'Username', 'Type' => 'text', 'Size' => '50'],
        'password' => ['FriendlyName' => 'Password', 'Type' => 'password', 'Size' => '50'],
        'base_url' => ['FriendlyName' => 'Base URL', 'Type' => 'text', 'Size' => '100', 'Default' => 'https://reseller.enom.com'],
        'test_mode' => ['FriendlyName' => 'Test Mode', 'Type' => 'yesno']
    ];
}

function enom_api_call($params, $command, $data = [])
{
    $baseUrl = $params['test_mode'] ? 'https://reseller.enom.com' : 'https://reseller.enom.com';
    
    $data = array_merge($data, [
        'UID' => $params['username'],
        'PW' => $params['password'],
        'ResponseType' => 'JSON',
        'command' => $command
    ]);
    
    $ch = curl_init($baseUrl . '/interface.asp?' . http_build_query($data));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    
    return json_decode(curl_exec($ch), true);
}

function enom_registerDomain($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $data = [
        'sld' => $params['sld'],
        'tld' => $params['tld'],
        'regperiod' => $params['regperiod'],
        'RegistrantFirstName' => $params['firstname'],
        'RegistrantLastName' => $params['lastname'],
        'RegistrantEmailAddress' => $params['email'],
        'RegistrantAddress1' => $params['address1'],
        'RegistrantCity' => $params['city'],
        'RegistrantStateProvince' => $params['state'],
        'RegistrantPostalCode' => $params['postcode'],
        'RegistrantCountry' => $params['country'],
        'RegistrantPhone' => $params['phonenumber']
    ];
    
    $result = enom_api_call($params, 'PP_CreateOrder', $data);
    
    if ($result['response_code'] == 0) {
        return ['success' => true, 'domain' => $domain];
    }
    
    return ['success' => false, 'error' => $result['response_msg'] ?? 'Registration failed'];
}

function enom_transferDomain($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $data = [
        'sld' => $params['sld'],
        'tld' => $params['tld'],
        'regperiod' => $params['regperiod'],
        'AuthCode' => $params['transfersecret']
    ];
    
    $result = enom_api_call($params, 'RUSTransfer', $data);
    
    return ['success' => $result['response_code'] == 0];
}

function enom_renewDomain($params)
{
    $data = ['sld' => $params['sld'], 'tld' => $params['tld'], 'regperiod' => $params['regperiod']];
    $result = enom_api_call($params, 'RenewDomain', $data);
    
    return ['success' => $result['response_code'] == 0];
}

function enom_getNameservers($params)
{
    $data = ['sld' => $params['sld'], 'tld' => $params['tld']];
    $result = enom_api_call($params, 'GetDNS', $data);
    
    if ($result['response_code'] == 0 && isset($result['DNSList'])) {
        $nameservers = explode(',', $result['DNSList']);
        return [
            'success' => true,
            'ns1' => trim($nameservers[0] ?? ''),
            'ns2' => trim($nameservers[1] ?? '')
        ];
    }
    
    return ['success' => false];
}

function enom_saveNameservers($params)
{
    $data = [
        'sld' => $params['sld'],
        'tld' => $params['tld'],
        'nameserver1' => $params['ns1'],
        'nameserver2' => $params['ns2'],
        'nameserver3' => $params['ns3'] ?? '',
        'nameserver4' => $params['ns4'] ?? ''
    ];
    
    $result = enom_api_call($params, 'SetDNS', $data);
    
    return ['success' => $result['response_code'] == 0];
}

function enom_getDomainInfo($params)
{
    $data = ['sld' => $params['sld'], 'tld' => $params['tld']];
    $result = enom_api_call($params, 'GetDomainInfo', $data);
    
    if ($result['response_code'] == 0) {
        return [
            'registrationdate' => $result['registration_date'] ?? '',
            'expirydate' => $result['exp_date'] ?? '',
            'autorenew' => ($result['auto_renew'] ?? '') == 'True'
        ];
    }
    
    return ['success' => false];
}

function enom_sync($params)
{
    return enom_getDomainInfo($params);
}

function enom_getContactDetails($params)
{
    return [
        'Registrant' => [
            'First Name' => $params['firstname'],
            'Last Name' => $params['lastname'],
            'Email' => $params['email']
        ]
    ];
}

function enom_saveContactDetails($params)
{
    return ['success' => true];
}

function enom_requestDelete($params)
{
    $data = ['sld' => $params['sld'], 'tld' => $params['tld']];
    $result = enom_api_call($params, 'RequestDelete', $data);
    
    return ['success' => $result['response_code'] == 0];
}
```