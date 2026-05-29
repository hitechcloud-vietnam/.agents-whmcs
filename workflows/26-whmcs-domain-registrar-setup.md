# WHMCS Domain Registrar Setup Workflow

## Overview
This workflow covers creating a custom domain registrar module for WHMCS.

## Step 1: Registrar Module

```php
<?php
// modules/registrars/your_registrar/your_registrar.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Registrar module configuration
 */
function your_registrar_MetaData()
{
    return [
        'DisplayName' => 'Your Registrar',
        'APIVersion' => '1.1',
        'RequiresRegSync' => true,
        'Parameters' => ['username', 'password', 'api_key']
    ];
}

function your_registrar_getConfigArray()
{
    return [
        'Username' => [
            'Type' => 'text',
            'Size' => '20',
            'Description' => 'Your registrar username'
        ],
        'Password' => [
            'Type' => 'password',
            'Size' => '20',
            'Description' => 'Your registrar password'
        ],
        'APIKey' => [
            'Type' => 'password',
            'Size' => '40',
            'Description' => 'API Key for API access'
        ],
        'TestMode' => [
            'Type' => 'yesno',
            'Description' => 'Enable test mode'
        ]
    ];
}

/**
 * Register domain
 */
function your_registrar_RegisterDomain($params)
{
    $username = $params['username'];
    $password = $params['password'];
    $apiKey = $params['api_key'];
    $domain = $params['domain'];
    $years = $params['regperiod'];
    $registrant = $params['contactdetails'];

    try {
        $response = callRegistrarApi($params, 'POST', '/domains/register', [
            'domain' => $domain,
            'period' => $years,
            'registrant' => $registrant
        ]);

        if ($response['success']) {
            return ['success' => true, 'transid' => $response['order_id']];
        }

        return ['error' => $response['message'] ?? 'Registration failed'];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Transfer domain
 */
function your_registrar_TransferDomain($params)
{
    $domain = $params['domain'];
    $authCode = $params['transfersecret'];

    try {
        $response = callRegistrarApi($params, 'POST', '/domains/transfer', [
            'domain' => $domain,
            'auth_code' => $authCode
        ]);

        return ['success' => true, 'transid' => $response['transfer_id']];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Renew domain
 */
function your_registrar_RenewDomain($params)
{
    $domain = $params['domain'];
    $years = $params['regperiod'];

    try {
        $response = callRegistrarApi($params, 'POST', '/domains/renew', [
            'domain' => $domain,
            'period' => $years
        ]);

        return ['success' => true, 'transid' => $response['renewal_id']];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Get nameservers
 */
function your_registrar_GetNameservers($params)
{
    $domain = $params['domain'];

    try {
        $response = callRegistrarApi($params, 'GET', "/domains/$domain/nameservers");

        return [
            'success' => true,
            'ns1' => $response['ns1'],
            'ns2' => $response['ns2'],
            'ns3' => $response['ns3'] ?? '',
            'ns4' => $response['ns4'] ?? ''
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Save nameservers
 */
function your_registrar_SaveNameservers($params)
{
    $domain = $params['domain'];
    $nameservers = [
        $params['ns1'],
        $params['ns2'],
        $params['ns3'],
        $params['ns4']
    ];

    try {
        callRegistrarApi($params, 'PUT', "/domains/$domain/nameservers", [
            'nameservers' => array_filter($nameservers)
        ]);

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Get domain info
 */
function your_registrar_GetDomainInfo($params)
{
    $domain = $params['domain'];

    try {
        $response = callRegistrarApi($params, 'GET', "/domains/$domain");

        return [
            'success' => true,
            'status' => $response['status'],
            'registration_date' => $response['created_date'],
            'expiry_date' => $response['expiry_date'],
            'dnssec' => $response['dnssec'] ?? false
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Release domain
 */
function your_registrar_ReleaseDomain($params)
{
    $domain = $params['domain'];
    $newRegistrar = $params['new registrant'];

    try {
        callRegistrarApi($params, 'POST', "/domains/$domain/release", [
            'new_registrar' => $newRegistrar
        ]);

        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Sync domain
 */
function your_registrar_Sync($params)
{
    $domain = $params['domain'];

    try {
        $response = callRegistrarApi($params, 'GET', "/domains/$domain");

        return [
            'success' => true,
            'expirydate' => $response['expiry_date'],
            'active' => $response['status'] === 'active'
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function callRegistrarApi($params, string $method, string $endpoint, array $data = []): array
{
    $baseUrl = ($params['TestMode'] ?? false)
        ? 'https://api.test.registrar.com'
        : 'https://api.registrar.com';

    $ch = curl_init($baseUrl . $endpoint);

    $headers = [
        'Authorization: Bearer ' . $params['apiKey'],
        'Content-Type: application/json'
    ];

    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => $headers
    ]);

    if ($method === 'POST' || $method === 'PUT') {
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
    }

    $response = curl_exec($ch);
    curl_close($ch);

    $result = json_decode($response, true);

    if (!$result['success']) {
        throw new \Exception($result['message'] ?? 'API error');
    }

    return $result;
}
```

## Verification Checklist

- [ ] Registrar module created
- [ ] Registration working
- [ ] Transfer working
- [ ] Renewal working
- [ ] Nameserver management working
- [ ] Domain sync working
- [ ] Test domain registered successfully
