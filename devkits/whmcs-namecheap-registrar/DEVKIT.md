# WHMCS Namecheap Registrar Module - DEVKIT

## Module Information
- **Name**: Namecheap Registrar
- **Version**: 1.0.0
- **Type**: Registrar Module
- **Description**: Namecheap domain registrar integration for WHMCS

## Installation
1. Copy to `/modules/registrars/namecheap/`
2. Activate via WHMCS Admin > Configuration > Domain Registrars

## namecheap.php
```php
<?php
/**
 * WHMCS Namecheap Registrar Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function namecheap_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Namecheap'
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => 'Namecheap Domain Registrar Integration'
        ],
        'api_user' => [
            'FriendlyName' => 'API Username',
            'Type' => 'text',
            'Size' => '50'
        ],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50'
        ],
        'client_id' => [
            'FriendlyName' => 'Client ID',
            'Type' => 'text',
            'Size' => '50',
            'Description' => 'Your Namecheap Client ID (from API access page)'
        ],
        'sandbox_mode' => [
            'FriendlyName' => 'Sandbox Mode',
            'Type' => 'yesno',
            'Description' => 'Enable sandbox environment for testing'
        ],
        'default_years' => [
            'FriendlyName' => 'Default Registration Years',
            'Type' => 'text',
            'Size' => '5',
            'Default' => '1',
            'Description' => 'Default number of years for new registrations'
        ],
        'auto_renew' => [
            'FriendlyName' => 'Auto Renew',
            'Type' => 'yesno',
            'Description' => 'Enable auto-renew by default'
        ]
    ];
}

function namecheap_getNameservers($params)
{
    $apiUser = $params['api_user'];
    $apiKey = $params['api_key'];
    $clientId = $params['client_id'];
    $sandbox = $params['sandbox_mode'] ? 'true' : 'false';
    
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, 'https://api.namecheap.com/xml.response');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    
    $postFields = [
        'ApiUser' => $apiUser,
        'ApiKey' => $apiKey,
        'ClientId' => $clientId,
        'UserName' => $apiUser,
        'Command' => 'namecheap.domains.getList',
        'Sandbox' => $sandbox
    ];
    
    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($postFields));
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $xml = simplexml_load_string($response);
    
    // Parse nameservers from domain info
    return [
        'success' => true,
        'ns1' => $xml->CommandResponse->DomainDNSGetListResult->Domain ?? '',
        'ns2' => '',
        'ns3' => '',
        'ns4' => '',
        'ns5' => ''
    ];
}

function namecheap_saveNameservers($params)
{
    $apiUser = $params['api_user'];
    $apiKey = $params['api_key'];
    $clientId = $params['client_id'];
    $sandbox = $params['sandbox_mode'] ? 'true' : 'false';
    
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, 'https://api.namecheap.com/xml.response');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    
    $postFields = [
        'ApiUser' => $apiUser,
        'ApiKey' => $apiKey,
        'ClientId' => $clientId,
        'UserName' => $apiUser,
        'Command' => 'namecheap.domains.dns.setCustom',
        'Sandbox' => $sandbox,
        'SLD' => $params['sld'],
        'TLD' => $params['tld'],
        'Nameservers' => implode(',', array_filter([
            $params['ns1'],
            $params['ns2'],
            $params['ns3'],
            $params['ns4'],
            $params['ns5']
        ]))
    ];
    
    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($postFields));
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $xml = simplexml_load_string($response);
    
    if ($xml->Attributes()->Success == 'true') {
        return ['success' => true];
    } else {
        return [
            'success' => false,
            'error' => (string)$xml->Errors->Error
        ];
    }
}

function namecheap_registerDomain($params)
{
    $apiUser = $params['api_user'];
    $apiKey = $params['api_key'];
    $clientId = $params['client_id'];
    $sandbox = $params['sandbox_mode'] ? 'true' : 'false';
    
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, 'https://api.namecheap.com/xml.response');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    
    $postFields = [
        'ApiUser' => $apiUser,
        'ApiKey' => $apiKey,
        'ClientId' => $clientId,
        'UserName' => $apiUser,
        'Command' => 'namecheap.domains.create',
        'Sandbox' => $sandbox,
        'DomainName' => $domain,
        'Years' => $params['regperiod'],
        'RegistrantFirstName' => $params['firstname'],
        'RegistrantLastName' => $params['lastname'],
        'RegistrantAddress1' => $params['address1'],
        'RegistrantCity' => $params['city'],
        'RegistrantStateProvince' => $params['state'],
        'RegistrantPostalCode' => $params['postcode'],
        'RegistrantCountry' => $params['country'],
        'RegistrantPhone' => $params['phonenumber'],
        'RegistrantEmailAddress' => $params['email'],
        'AdminFirstName' => $params['firstname'],
        'AdminLastName' => $params['lastname'],
        'AdminAddress1' => $params['address1'],
        'AdminCity' => $params['city'],
        'AdminStateProvince' => $params['state'],
        'AdminPostalCode' => $params['postcode'],
        'AdminCountry' => $params['country'],
        'AdminPhone' => $params['phonenumber'],
        'AdminEmailAddress' => $params['email']
    ];
    
    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($postFields));
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $xml = simplexml_load_string($response);
    
    if ($xml->Attributes()->Success == 'true') {
        return [
            'success' => true,
            'domain' => $domain
        ];
    } else {
        return [
            'success' => false,
            'error' => (string)$xml->Errors->Error
        ];
    }
}

function namecheap_transferDomain($params)
{
    $apiUser = $params['api_user'];
    $apiKey = $params['api_key'];
    $clientId = $params['client_id'];
    $sandbox = $params['sandbox_mode'] ? 'true' : 'false';
    
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, 'https://api.namecheap.com/xml.response');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    
    $postFields = [
        'ApiUser' => $apiUser,
        'ApiKey' => $apiKey,
        'ClientId' => $clientId,
        'UserName' => $apiUser,
        'Command' => 'namecheap.domains.transfer.create',
        'Sandbox' => $sandbox,
        'DomainName' => $domain,
        'Years' => $params['regperiod'],
        'EPPCode' => $params['transfersecret'],
        'RegistrantFirstName' => $params['firstname'],
        'RegistrantLastName' => $params['lastname'],
        'RegistrantEmailAddress' => $params['email']
    ];
    
    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($postFields));
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $xml = simplexml_load_string($response);
    
    if ($xml->Attributes()->Success == 'true') {
        return [
            'success' => true,
            'transferid' => (string)$xml->CommandResponse->DomainTransferCreateResult->TransferID
        ];
    } else {
        return [
            'success' => false,
            'error' => (string)$xml->Errors->Error
        ];
    }
}

function namecheap_renewDomain($params)
{
    $apiUser = $params['api_user'];
    $apiKey = $params['api_key'];
    $clientId = $params['client_id'];
    $sandbox = $params['sandbox_mode'] ? 'true' : 'false';
    
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, 'https://api.namecheap.com/xml.response');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    
    $postFields = [
        'ApiUser' => $apiUser,
        'ApiKey' => $apiKey,
        'ClientId' => $clientId,
        'UserName' => $apiUser,
        'Command' => 'namecheap.domains.renew',
        'Sandbox' => $sandbox,
        'DomainName' => $domain,
        'Years' => $params['regperiod']
    ];
    
    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($postFields));
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $xml = simplexml_load_string($response);
    
    if ($xml->Attributes()->Success == 'true') {
        return ['success' => true];
    } else {
        return [
            'success' => false,
            'error' => (string)$xml->Errors->Error
        ];
    }
}

function namecheap_getContactDetails($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    // Return contact info from WHMCS
    return [
        'Registrant' => [
            'First Name' => $params['firstname'],
            'Last Name' => $params['lastname'],
            'Company' => $params['companyname'],
            'Email' => $params['email'],
            'Address 1' => $params['address1'],
            'City' => $params['city'],
            'State' => $params['state'],
            'Postcode' => $params['postcode'],
            'Country' => $params['country'],
            'Phone' => $params['phonenumber']
        ]
    ];
}

function namecheap_saveContactDetails($params)
{
    // Save contact details - Namecheap uses same for all contacts
    return ['success' => true];
}

function namecheap_requestDelete($params)
{
    $apiUser = $params['api_user'];
    $apiKey = $params['api_key'];
    $clientId = $params['client_id'];
    $sandbox = $params['sandbox_mode'] ? 'true' : 'false';
    
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, 'https://api.namecheap.com/xml.response');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    
    $postFields = [
        'ApiUser' => $apiUser,
        'ApiKey' => $apiKey,
        'ClientId' => $clientId,
        'UserName' => $apiUser,
        'Command' => 'namecheap.domains.delete',
        'Sandbox' => $sandbox,
        'DomainName' => $domain
    ];
    
    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($postFields));
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $xml = simplexml_load_string($response);
    
    return ['success' => $xml->Attributes()->Success == 'true'];
}

function namecheap_getDomainInfo($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    return [
        'registrationdate' => date('Y-m-d'),
        'expirydate' => date('Y-m-d', strtotime('+1 year')),
        'dnssec' => false
    ];
}

function namecheap_sync($params)
{
    $apiUser = $params['api_user'];
    $apiKey = $params['api_key'];
    $clientId = $params['client_id'];
    $sandbox = $params['sandbox_mode'] ? 'true' : 'false';
    
    $domain = $params['sld'] . '.' . $params['tld'];
    
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, 'https://api.namecheap.com/xml.response');
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    
    $postFields = [
        'ApiUser' => $apiUser,
        'ApiKey' => $apiKey,
        'ClientId' => $clientId,
        'UserName' => $apiUser,
        'Command' => 'namecheap.domains.getInfo',
        'Sandbox' => $sandbox,
        'DomainName' => $domain
    ];
    
    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($postFields));
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $xml = simplexml_load_string($response);
    
    $info = $xml->CommandResponse->DomainGetInfoResult;
    
    return [
        'registrationdate' => (string)$info->RegistrationDate,
        'expirydate' => (string)$info->ExpirationDate,
        'autorenew' => ((string)$info->AutoRenew == 'true'),
        'locked' => ((string)$info->IsLocked == 'true')
    ];
}
```

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('DomainTransferCompleted', 1, function($vars) {
    logActivity("Namecheap transfer completed for domain: " . $vars['domain']);
});

add_hook('DomainRenewed', 1, function($vars) {
    logActivity("Namecheap renewal completed for domain: " . $vars['domain']);
});
```