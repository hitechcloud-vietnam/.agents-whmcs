# WHMCS Cloudflare Registrar Module - DEVKIT

## Module Information
- **Name**: Cloudflare Registrar
- **Version**: 1.0.0
- **Type**: Registrar Module
- **Description**: Cloudflare Registrar integration for WHMCS

## Installation
1. Copy to `/modules/registrars/cloudflare_registrar/`
2. Activate via WHMCS Admin > Configuration > Domain Registrars

## cloudflare_registrar.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function cloudflare_registrar_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Cloudflare Registrar'
        ],
        'api_token' => [
            'FriendlyName' => 'API Token',
            'Type' => 'password',
            'Size' => '80'
        ],
        'account_id' => [
            'FriendlyName' => 'Account ID',
            'Type' => 'text',
            'Size' => '50'
        ]
    ];
}

function cloudflare_registrar_registerDomain($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $data = [
        'name' => $domain,
        'period' => (int)$params['regperiod'],
        'registrant_contact' => [
            'first_name' => $params['firstname'],
            'last_name' => $params['lastname'],
            'email' => $params['email'],
            'phone' => $params['phonenumber'],
            'address1' => $params['address1'],
            'city' => $params['city'],
            'state' => $params['state'],
            'country' => $params['country'],
            'postal_code' => $params['postcode']
        ]
    ];
    
    $result = cloudflare_api($params, 'POST', '/accounts/' . $params['account_id'] . '/registrar/domains', $data);
    
    if (isset($result['result'])) {
        return ['success' => true, 'domain' => $domain];
    }
    
    return ['success' => false, 'error' => $result['errors'][0]['message'] ?? 'Failed'];
}

function cloudflare_registrar_transferDomain($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $data = [
        'name' => $domain,
        'period' => (int)$params['regperiod'],
        'auth_code' => $params['transfersecret']
    ];
    
    $result = cloudflare_api($params, 'POST', '/accounts/' . $params['account_id'] . '/registrar/domains/' . $domain . '/transfer', $data);
    
    return ['success' => isset($result['result'])];
}

function cloudflare_registrar_renewDomain($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    $data = ['period' => (int)$params['regperiod']];
    
    $result = cloudflare_api($params, 'POST', '/accounts/' . $params['account_id'] . '/registrar/domains/' . $domain . '/renew', $data);
    
    return ['success' => isset($result['result'])];
}

function cloudflare_registrar_getNameservers($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    $result = cloudflare_api($params, 'GET', '/accounts/' . $params['account_id'] . '/registrar/domains/' . $domain . '/dns');
    
    if (isset($result['result'])) {
        return ['success' => true, 'ns1' => $result['result'][0]['dns'] ?? '', 'ns2' => $result['result'][1]['dns'] ?? ''];
    }
    
    return ['success' => false];
}

function cloudflare_registrar_saveNameservers($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    $data = ['dns' => array_filter([$params['ns1'], $params['ns2']])];
    
    $result = cloudflare_api($params, 'PUT', '/accounts/' . $params['account_id'] . '/registrar/domains/' . $domain . '/dns', $data);
    
    return ['success' => isset($result['result'])];
}

function cloudflare_registrar_getDomainInfo($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    $result = cloudflare_api($params, 'GET', '/accounts/' . $params['account_id'] . '/registrar/domains/' . $domain);
    
    if (isset($result['result'])) {
        return ['registrationdate' => $result['result']['created_at'], 'expirydate' => $result['result']['expires_at'], 'autorenew' => $result['result']['auto_renew']];
    }
    
    return ['success' => false];
}

function cloudflare_registrar_sync($params)
{
    return cloudflare_registrar_getDomainInfo($params);
}

function cloudflare_api($params, $method, $endpoint, $data = null)
{
    $ch = curl_init('https://api.cloudflare.com/client/v4' . $endpoint);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Authorization: Bearer ' . $params['api_token'], 'Content-Type: application/json']);
    
    if ($data) {
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
    }
    
    return json_decode(curl_exec($ch), true);
}
```