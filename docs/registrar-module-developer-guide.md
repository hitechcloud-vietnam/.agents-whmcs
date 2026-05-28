# WHMCS Registrar Module Developer Guide

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

Registrar modules enable WHMCS to communicate with domain registrars for domain registration, transfer, renewal, and management operations. This guide covers complete development procedures.

---

## Module Structure

```
modules/registrars/
  your_registrar/
    your_registrar.php     # Main module file
    whois.php              # WHOIS lookup functionality
    logo.png               # Registrar logo (80x80px)
```

## Main Module File

```php
<?php
/**
 * Registrar Module Definition
 * 
 * @package WHMCS
 * @subpackage RegistrarModule
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Register module
 * 
 * @return array Module configuration
 */
function your_registrar_registerModule()
{
    return [
        'name'          => 'Your Registrar',
        'description'   => 'Domain registrar integration',
        'language'      => 'your_registrar',
        'version'       => '1.0.0',
    ];
}

/**
 * Get configuration parameters
 * 
 * @return array Configuration fields
 */
function your_registrar_getConfigArray()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value'=> 'Your Registrar',
        ],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type'         => 'password',
            'Size'         => '40',
        ],
        'api_secret' => [
            'FriendlyName' => 'API Secret',
            'Type'         => 'password',
            'Size'         => '40',
        ],
        'reseller_id' => [
            'FriendlyName' => 'Reseller ID',
            'Type'         => 'text',
            'Size'         => '20',
        ],
        'sandbox_mode' => [
            'FriendlyName' => 'Sandbox Mode',
            'Type'         => 'yesno',
        ],
        'dns_template' => [
            'FriendlyName' => 'Default DNS Template',
            'Type'         => 'dropdown',
            'Options'      => [
                'default'   => 'Default',
                'parked'    => 'Parked',
                'cloudflare'=> 'Cloudflare',
            ],
        ],
    ];
}
```

## Domain Registration

```php
/**
 * Register a domain
 * 
 * @param array $params Common module parameters
 * @param array $domainParams Domain-specific parameters
 * @return array Registration result
 */
function your_registrar_RegisterDomain($params, $domainParams)
{
    $apiKey   = $params['apiKey'];
    $apiSecret = $params['apiSecret'];
    $sandbox  = $params['sandbox_mode'];
    
    $domain   = $domainParams['domainname'];
    $years    = $domainParams['registrationperiod'];
    $contacts = $domainParams['contacts'];
    
    // Prepare registration data
    $registrantData = [
        'name'     => $contacts['Registrant']['name'],
        'email'    => $contacts['Registrant']['email'],
        'address1' => $contacts['Registrant']['address1'],
        'city'     => $contacts['Registrant']['city'],
        'country'  => $contacts['Registrant']['country'],
        'phone'    => $contacts['Registrant']['phone'],
        'postcode' => $contacts['Registrant']['postcode'],
    ];
    
    // Build API request
    $endpoint = $sandbox 
        ? 'https://sandbox.yourregistrar.com/api/register' 
        : 'https://api.yourregistrar.com/v1/domains';
    
    $data = [
        'domain'     => $domain,
        'period'     => $years,
        'registrant' => $registrantData,
        'nameservers'=> $domainParams['nameservers'] ?? [],
    ];
    
    $response = your_registrar_apiCall('POST', $endpoint, $data, $apiKey, $apiSecret);
    
    if ($response['success']) {
        return [
            'success'   => true,
            'domain'    => $domain,
            'expirydate'=> $response['expiry_date'],
            'reg_period'=> $years,
        ];
    }
    
    return [
        'success' => false,
        'error'   => $response['message'] ?? 'Registration failed',
    ];
}
```

## Domain Transfer

```php
/**
 * Transfer a domain
 * 
 * @param array $params Common module parameters
 * @param array $domainParams Domain-specific parameters
 * @return array Transfer result
 */
function your_registrar_TransferDomain($params, $domainParams)
{
    $apiKey    = $params['apiKey'];
    $apiSecret = $params['apiSecret'];
    $sandbox   = $params['sandbox_mode'];
    
    $domain      = $domainParams['domainname'];
    $authCode    = $domainParams['transfersecret'];
    $registrant  = $domainParams['contacts']['Registrant'];
    
    $endpoint = $sandbox 
        ? 'https://sandbox.yourregistrar.com/api/transfer' 
        : 'https://api.yourregistrar.com/v1/domains/transfer';
    
    $data = [
        'domain'          => $domain,
        'auth_code'       => $authCode,
        'registrant_name' => $registrant['name'],
        'registrant_email'=> $registrant['email'],
    ];
    
    $response = your_registrar_apiCall('POST', $endpoint, $data, $apiKey, $apiSecret);
    
    if ($response['success']) {
        return [
            'success'    => true,
            'transferid' => $response['transfer_id'],
            'domain'     => $domain,
        ];
    }
    
    return [
        'success' => false,
        'error'   => $response['message'] ?? 'Transfer initiation failed',
    ];
}
```

## Domain Renewal

```php
/**
 * Renew a domain
 * 
 * @param array $params Common module parameters
 * @param array $domainParams Domain-specific parameters
 * @return array Renewal result
 */
function your_registrar_RenewDomain($params, $domainParams)
{
    $apiKey    = $params['apiKey'];
    $apiSecret = $params['apiSecret'];
    $sandbox   = $params['sandbox_mode'];
    
    $domain = $domainParams['domainname'];
    $years  = $domainParams['registrationperiod'];
    
    $endpoint = $sandbox 
        ? 'https://sandbox.yourregistrar.com/api/renew' 
        : 'https://api.yourregistrar.com/v1/domains/renew';
    
    $data = [
        'domain' => $domain,
        'period' => $years,
    ];
    
    $response = your_registrar_apiCall('POST', $endpoint, $data, $apiKey, $apiSecret);
    
    if ($response['success']) {
        return [
            'success'   => true,
            'expirydate'=> $response['expiry_date'],
            'domain'    => $domain,
        ];
    }
    
    return [
        'success' => false,
        'error'   => $response['message'] ?? 'Renewal failed',
    ];
}
```

## Domain Management

```php
/**
 * Get domain details
 * 
 * @param array $params Common module parameters
 * @param array $domainParams Domain-specific parameters
 * @return array Domain information
 */
function your_registrar_GetDomainDetails($params, $domainParams)
{
    $apiKey    = $params['apiKey'];
    $apiSecret = $params['apiSecret'];
    $sandbox   = $params['sandbox_mode'];
    
    $domain = $domainParams['domainname'];
    
    $endpoint = $sandbox 
        ? 'https://sandbox.yourregistrar.com/api/domain/' . $domain 
        : 'https://api.yourregistrar.com/v1/domains/' . $domain;
    
    $response = your_registrar_apiCall('GET', $endpoint, [], $apiKey, $apiSecret);
    
    if ($response['success']) {
        return [
            'success'       => true,
            'domain'        => $domain,
            'registrationdate' => $response['registration_date'],
            'expirydate'    => $response['expiry_date'],
            'dnssec'        => $response['dnssec'] ?? [],
            'nameservers'   => $response['nameservers'] ?? [],
            'status'        => $response['status'],
            'registrant'    => $response['registrant'],
            'admin'         => $response['admin'],
            'tech'          => $response['tech'],
            'billing'       => $response['billing'],
            'privacy'       => $response['privacy_protection'],
        ];
    }
    
    return ['success' => false, 'error' => $response['message']];
}

/**
 * Save domain contacts
 * 
 * @param array $params Common module parameters
 * @param array $domainParams Domain-specific parameters
 * @return array Operation result
 */
function your_registrar_SaveContactDetails($params, $domainParams)
{
    $apiKey    = $params['apiKey'];
    $apiSecret = $params['apiSecret'];
    $sandbox   = $params['sandbox_mode'];
    
    $domain   = $domainParams['domainname'];
    $contacts = $domainParams['contacts'];
    
    $endpoint = $sandbox 
        ? 'https://sandbox.yourregistrar.com/api/domain/' . $domain . '/contacts' 
        : 'https://api.yourregistrar.com/v1/domains/' . $domain . '/contacts';
    
    $data = [
        'registrant' => $contacts['Registrant'],
        'admin'      => $contacts['Admin'],
        'tech'       => $contacts['Tech'],
        'billing'    => $contacts['Billing'],
    ];
    
    $response = your_registrar_apiCall('PUT', $endpoint, $data, $apiKey, $apiSecret);
    
    return [
        'success' => $response['success'],
        'error'   => $response['success'] ? null : ($response['message'] ?? 'Save failed'),
    ];
}

/**
 * Update nameservers
 * 
 * @param array $params Common module parameters
 * @param array $domainParams Domain-specific parameters
 * @return array Operation result
 */
function your_registrar_UpdateNameservers($params, $domainParams)
{
    $apiKey    = $params['apiKey'];
    $apiSecret = $params['apiSecret'];
    $sandbox   = $params['sandbox_mode'];
    
    $domain      = $domainParams['domainname'];
    $nameservers = $domainParams['nslist'];
    
    $endpoint = $sandbox 
        ? 'https://sandbox.yourregistrar.com/api/domain/' . $domain . '/nameservers' 
        : 'https://api.yourregistrar.com/v1/domains/' . $domain . '/nameservers';
    
    $data = [
        'nameservers' => $nameservers,
    ];
    
    $response = your_registrar_apiCall('PUT', $endpoint, $data, $apiKey, $apiSecret);
    
    return [
        'success' => $response['success'],
        'error'   => $response['success'] ? null : ($response['message'] ?? 'Update failed'),
    ];
}
```

## DNSSEC Management

```php
/**
 * Get DNSSEC information
 * 
 * @param array $params Common module parameters
 * @param array $domainParams Domain-specific parameters
 * @return array DNSSEC records
 */
function your_registrar_GetDNSSEC($params, $domainParams)
{
    $apiKey    = $params['apiKey'];
    $apiSecret = $params['apiSecret'];
    $sandbox   = $params['sandbox_mode'];
    
    $domain = $domainParams['domainname'];
    
    $endpoint = $sandbox 
        ? 'https://sandbox.yourregistrar.com/api/domain/' . $domain . '/dnssec' 
        : 'https://api.yourregistrar.com/v1/domains/' . $domain . '/dnssec';
    
    $response = your_registrar_apiCall('GET', $endpoint, [], $apiKey, $apiSecret);
    
    if ($response['success']) {
        $records = [];
        foreach ($response['records'] as $record) {
            $records[] = [
                'flags'     => $record['flags'],
                'protocol'  => $record['protocol'],
                'algorithm' => $record['algorithm'],
                'digest'    => $record['digest'],
            ];
        }
        return ['success' => true, 'dnssec' => $records];
    }
    
    return ['success' => false, 'dnssec' => []];
}

/**
 * Save/update DNSSEC records
 * 
 * @param array $params Common module parameters
 * @param array $domainParams Domain-specific parameters
 * @return array Operation result
 */
function your_registrar_SaveDNSSEC($params, $domainParams)
{
    $apiKey    = $params['apiKey'];
    $apiSecret = $params['apiSecret'];
    $sandbox   = $params['sandbox_mode'];
    
    $domain  = $domainParams['domainname'];
    $records = $domainParams['dnssec'];
    
    $endpoint = $sandbox 
        ? 'https://sandbox.yourregistrar.com/api/domain/' . $domain . '/dnssec' 
        : 'https://api.yourregistrar.com/v1/domains/' . $domain . '/dnssec';
    
    $data = ['records' => $records];
    
    $response = your_registrar_apiCall('PUT', $endpoint, $data, $apiKey, $apiSecret);
    
    return [
        'success' => $response['success'],
        'error'   => $response['success'] ? null : ($response['message'] ?? 'Save failed'),
    ];
}
```

## WHOIS Lookup

```php
<?php
/**
 * WHOIS Lookup
 * 
 * @param string $domain Domain name
 * @return array WHOIS result
 */
function your_registrar_whois($domain)
{
    $response = your_registrar_apiCall('GET', 'https://api.yourregistrar.com/v1/whois/' . $domain);
    
    if ($response['success']) {
        return [
            'success'  => true,
            'whois'    => $response['whois_data'],
            'available'=> $response['available'],
        ];
    }
    
    return [
        'success'  => false,
        'whois'    => '',
        'available'=> null,
        'error'    => $response['message'],
    ];
}
```

## Availability Check

```php
/**
 * Check domain availability
 * 
 * @param array $params Common module parameters
 * @param array $domains Array of domain names to check
 * @return array Availability results
 */
function your_registrar_CheckAvailability($params, $domains)
{
    $apiKey    = $params['apiKey'];
    $apiSecret = $params['apiSecret'];
    $sandbox   = $params['sandbox_mode'];
    
    $endpoint = $sandbox 
        ? 'https://sandbox.yourregistrar.com/api/check' 
        : 'https://api.yourregistrar.com/v1/domains/check';
    
    $data = ['domains' => $domains];
    
    $response = your_registrar_apiCall('POST', $endpoint, $data, $apiKey, $apiSecret);
    
    // Format results for WHMCS
    $results = [];
    foreach ($response['results'] as $domain => $info) {
        $results[$domain] = [
            'status'     => $info['status'],
            'price'      => $info['price'] ?? 0,
            'currency'   => $info['currency'] ?? 'USD',
            'duration'   => $info['duration'] ?? [],
            'transfer_price' => $info['transfer_price'] ?? 0,
            'renew_price'    => $info['renew_price'] ?? 0,
        ];
    }
    
    return $results;
}
```

## Helper Function

```php
/**
 * Make API call to registrar
 * 
 * @param string $method HTTP method
 * @param string $endpoint API endpoint
 * @param array $data Request data
 * @param string $apiKey API key
 * @param string $apiSecret API secret
 * @return array Response
 */
function your_registrar_apiCall($method, $endpoint, $data = [], $apiKey = '', $apiSecret = '')
{
    $ch = curl_init();
    
    $headers = [
        'Content-Type: application/json',
        'Accept: application/json',
    ];
    
    if ($apiKey && $apiSecret) {
        $timestamp = time();
        $signature = hash_hmac('sha256', $timestamp . $apiKey, $apiSecret);
        $headers[] = 'X-API-Key: ' . $apiKey;
        $headers[] = 'X-Timestamp: ' . $timestamp;
        $headers[] = 'X-Signature: ' . $signature;
    }
    
    curl_setopt_array($ch, [
        CURLOPT_URL            => $endpoint,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT        => 30,
        CURLOPT_HTTPHEADER     => $headers,
    ]);
    
    if (in_array($method, ['POST', 'PUT', 'DELETE'])) {
        curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
        if (!empty($data)) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }
    }
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    $result = json_decode($response, true) ?? [];
    $result['http_code'] = $httpCode;
    
    return $result;
}
```

## Installation Checklist

1. Create registrar directory in `/modules/registrars/`
2. Implement all required functions
3. Add WHOIS lookup functionality
4. Include availability checking
5. Implement all contact management functions
6. Add DNSSEC support if registrar supports it
7. Test in sandbox environment
8. Document all required TLDs

## Required Functions

| Function | Required | Description |
|----------|----------|-------------|
| `registerModule` | Yes | Module registration |
| `getConfigArray` | Yes | Configuration fields |
| `RegisterDomain` | Yes | Domain registration |
| `TransferDomain` | Yes | Domain transfer |
| `RenewDomain` | Yes | Domain renewal |
| `GetDomainDetails` | Yes | Retrieve domain info |
| `SaveContactDetails` | Yes | Update contacts |
| `UpdateNameservers` | Yes | Update DNS servers |
| `CheckAvailability` | Yes | Check availability |
| `GetDNSSEC` | No | DNSSEC retrieval |
| `SaveDNSSEC` | No | DNSSEC update |

---

## Related Skills and Workflows

- `module-security-standards` - API credential security
- `module-testing-strategies` - Testing registrar operations
- `module-error-handling-guide` - Error handling patterns
- `registrar-api-reference` - Registrar API documentation
- `webhook-events-reference` - Domain lifecycle webhooks
