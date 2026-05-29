# WHMCS DNSSEC Manager Module - DEVKIT

## Module Information
- **Name**: DNSSEC Manager
- **Version**: 1.0.0
- **Type**: Registrar Module
- **Description**: DNSSEC management and monitoring for WHMCS

## Installation
1. Copy to `/modules/registrars/dnssec_manager/`
2. Activate via WHMCS Admin > Configuration > Domain Registrars

## dnssec_manager.php
```php
<?php
/**
 * WHMCS DNSSEC Manager Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function dnssec_manager_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'DNSSEC Manager'
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => 'DNSSEC management and monitoring'
        ],
        'dns_provider' => [
            'FriendlyName' => 'DNS Provider',
            'Type' => 'dropdown',
            'Options' => [
                'cloudflare' => 'Cloudflare',
                'dnsimple' => 'DNSimple',
                'route53' => 'Route 53',
                'generic' => 'Generic API'
            ],
            'Default' => 'generic'
        ],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '80'
        ],
        'auto_enable' => [
            'FriendlyName' => 'Auto Enable DNSSEC',
            'Type' => 'yesno',
            'Description' => 'Automatically enable DNSSEC for new domains'
        ],
        'monitor_expiry' => [
            'FriendlyName' => 'Monitor Expiry',
            'Type' => 'yesno',
            'Description' => 'Monitor DNSSEC key expiration'
        ],
        'expiry_warning_days' => [
            'FriendlyName' => 'Warning Days Before Expiry',
            'Type' => 'text',
            'Size' => '10',
            'Default' => '30',
            'Description' => 'Days before expiry to send warning'
        ]
    ];
}

function dnssec_manager_getDNSSEC($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $result = dnssec_manager_queryDnssec($params, $domain);
    
    if ($result['exists']) {
        return [
            'exists' => true,
            'keys' => $result['keys']
        ];
    }
    
    return ['exists' => false];
}

function dnssec_manager_setDNSSEC($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    $enable = $params['enable'];
    
    if ($enable) {
        return dnssec_manager_enableDnssec($params, $domain);
    } else {
        return dnssec_manager_disableDnssec($params, $domain);
    }
}

function dnssec_manager_enableDnssec($params, $domain)
{
    $provider = $params['dns_provider'];
    
    switch ($provider) {
        case 'cloudflare':
            return dnssec_cloudflare_enable($params, $domain);
            
        case 'dnsimple':
            return dnssec_dnsimple_enable($params, $domain);
            
        case 'route53':
            return dnssec_route53_enable($params, $domain);
            
        default:
            return dnssec_generic_enable($params, $domain);
    }
}

function dnssec_manager_disableDnssec($params, $domain)
{
    $provider = $params['dns_provider'];
    
    switch ($provider) {
        case 'cloudflare':
            return dnssec_cloudflare_disable($params, $domain);
            
        case 'dnsimple':
            return dnssec_dnsimple_disable($params, $domain);
            
        case 'route53':
            return dnssec_route53_disable($params, $domain);
            
        default:
            return dnssec_generic_disable($params, $domain);
    }
}

function dnssec_manager_queryDnssec($params, $domain)
{
    $provider = $params['dns_provider'];
    
    switch ($provider) {
        case 'cloudflare':
            return dnssec_cloudflare_query($params, $domain);
            
        case 'dnsimple':
            return dnssec_dnsimple_query($params, $domain);
            
        case 'route53':
            return dnssec_route53_query($params, $domain);
            
        default:
            return dnssec_generic_query($params, $domain);
    }
}

// Cloudflare implementation
function dnssec_cloudflare_enable($params, $domain)
{
    $apiToken = $params['api_key'];
    
    // Get zone ID
    $ch = curl_init('https://api.cloudflare.com/client/v4/zones?name=' . urlencode($domain));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $apiToken,
        'Content-Type: application/json'
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $data = json_decode($response, true);
    
    if (empty($data['result'][0])) {
        return ['success' => false, 'error' => 'Zone not found'];
    }
    
    $zoneId = $data['result'][0]['id'];
    
    // Enable DNSSEC
    $ch = curl_init('https://api.cloudflare.com/client/v4/zones/' . $zoneId . '/dnssec');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $apiToken,
        'Content-Type: application/json'
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['result'])) {
        // Store DS record info
        dnssec_manager_storeDsRecord($domain, $result['result']);
        
        return ['success' => true];
    }
    
    return ['success' => false, 'error' => $result['errors'][0]['message'] ?? 'Failed'];
}

function dnssec_cloudflare_query($params, $domain)
{
    $apiToken = $params['api_key'];
    
    $ch = curl_init('https://api.cloudflare.com/client/v4/zones?name=' . urlencode($domain));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $apiToken
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $data = json_decode($response, true);
    
    if (empty($data['result'][0])) {
        return ['exists' => false];
    }
    
    $zoneId = $data['result'][0]['id'];
    
    $ch = curl_init('https://api.cloudflare.com/client/v4/zones/' . $zoneId . '/dnssec');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $apiToken
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['result'])) {
        return [
            'exists' => true,
            'keys' => [
                [
                    'algorithm' => '13', // ECDSAP256SHA256
                    'digest_type' => 2, // SHA-256
                    'digest' => $result['result']['digest'] ?? '',
                    'key_tag' => $result['result']['key_tag'] ?? 0
                ]
            ],
            'ds_record' => $result['result']['ds'] ?? ''
        ];
    }
    
    return ['exists' => false];
}

function dnssec_cloudflare_disable($params, $domain)
{
    $apiToken = $params['api_key'];
    
    $ch = curl_init('https://api.cloudflare.com/client/v4/zones?name=' . urlencode($domain));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $apiToken
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $data = json_decode($response, true);
    
    if (empty($data['result'][0])) {
        return ['success' => false];
    }
    
    $zoneId = $data['result'][0]['id'];
    
    $ch = curl_init('https://api.cloudflare.com/client/v4/zones/' . $zoneId . '/dnssec');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'DELETE');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $apiToken
    ]);
    
    curl_exec($ch);
    curl_close($ch);
    
    return ['success' => true];
}

// DNSimple implementation
function dnssec_dnsimple_enable($params, $domain)
{
    $apiToken = $params['api_key'];
    
    // Get domain ID
    $ch = curl_init('https://api.dnsimple.com/v2/domains/' . urlencode($domain));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $apiToken
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $data = json_decode($response, true);
    
    if (empty($data['data']['id'])) {
        return ['success' => false, 'error' => 'Domain not found'];
    }
    
    $domainId = $data['data']['id'];
    
    // Enable DNSSEC
    $ch = curl_init('https://api.dnsimple.com/v2/domains/' . $domainId . '/dnssec');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $apiToken,
        'Content-Type: application/json'
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    return ['success' => isset($result['data'])];
}

function dnssec_dnsimple_query($params, $domain)
{
    $apiToken = $params['api_key'];
    
    $ch = curl_init('https://api.dnsimple.com/v2/domains/' . urlencode($domain) . '/dnssec');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $apiToken
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['data'])) {
        return [
            'exists' => true,
            'keys' => [[
                'algorithm' => $result['data']['algorithm'] ?? 13,
                'key_tag' => $result['data']['key_tag'] ?? 0
            ]]
        ];
    }
    
    return ['exists' => false];
}

function dnssec_dnsimple_disable($params, $domain)
{
    $apiToken = $params['api_key'];
    
    $ch = curl_init('https://api.dnsimple.com/v2/domains/' . urlencode($domain) . '/dnssec');
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'DELETE');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $apiToken
    ]);
    
    curl_exec($ch);
    curl_close($ch);
    
    return ['success' => true];
}

// Route53 implementation
function dnssec_route53_enable($params, $domain)
{
    // Route 53 DNSSEC implementation
    return ['success' => false, 'error' => 'Route53 implementation needed'];
}

function dnssec_route53_query($params, $domain)
{
    return ['exists' => false];
}

function dnssec_route53_disable($params, $domain)
{
    return ['success' => false];
}

// Generic implementation
function dnssec_generic_enable($params, $domain)
{
    return ['success' => false, 'error' => 'Generic API not configured'];
}

function dnssec_generic_query($params, $domain)
{
    return ['exists' => false];
}

function dnssec_generic_disable($params, $domain)
{
    return ['success' => false];
}

// Helper function to store DS records
function dnssec_manager_storeDsRecord($domain, $dnssecData)
{
    full_query("
        INSERT INTO " . TABLE_PREFIX . "mod_dnssec_records 
        (domain, ds_record, key_tag, algorithm, created_at)
        VALUES (
            '" . db_escape_string($domain) . "',
            '" . db_escape_string($dnssecData['ds'] ?? '') . "',
            " . (int)($dnssecData['key_tag'] ?? 0) . ",
            '" . db_escape_string($dnssecData['algorithm'] ?? '') . "',
            NOW()
        )
    ");
}
```

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('DailyCronJob', 1, function($vars) {
    DNSSECHelper::monitorExpiringKeys();
});

add_hook('DomainTransferCompleted', 1, function($vars) {
    $params = [
        'sld' => $vars['sld'],
        'tld' => $vars['tld'],
        'dns_provider' => 'cloudflare',
        'api_key' => 'your_api_key'
    ];
    
    // Auto-enable DNSSEC for transferred domains if configured
    // dnssec_manager_enableDnssec($params, $vars['domain']);
});

class DNSSECHelper
{
    public static function monitorExpiringKeys()
    {
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . "mod_dnssec_records
            WHERE expires_at IS NOT NULL
            AND expires_at <= DATE_ADD(CURDATE(), INTERVAL 30 DAY)
            AND notified = 0
        ");
        
        while ($record = mysql_fetch_array($result)) {
            self::sendExpiryNotification($record);
            
            full_query("
                UPDATE " . TABLE_PREFIX . "mod_dnssec_records
                SET notified = 1
                WHERE id = " . (int)$record['id']
            );
        }
    }
    
    private static function sendExpiryNotification($record)
    {
        // Send admin notification about expiring DNSSEC keys
        logActivity("DNSSEC key expiring for domain: " . $record['domain']);
    }
    
    public static function validateDNSSEC($domain)
    {
        $dsRecords = dnssec_fetchDsRecords($domain);
        
        if (empty($dsRecords)) {
            return ['valid' => false, 'error' => 'No DS records found'];
        }
        
        $dnskeys = dnssec_fetchDnskeys($domain);
        
        foreach ($dsRecords as $ds) {
            $matchingKey = false;
            
            foreach ($dnskeys as $key) {
                if ($key['key_tag'] == $ds['key_tag'] && $key['algorithm'] == $ds['algorithm']) {
                    $matchingKey = true;
                    break;
                }
            }
            
            if (!$matchingKey) {
                return ['valid' => false, 'error' => 'DS record validation failed'];
            }
        }
        
        return ['valid' => true];
    }
}
```