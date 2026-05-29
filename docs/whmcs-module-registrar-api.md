# WHMCS Registrar Module API

Complete reference for domain registrar module development in WHMCS.

## Overview

Registrar modules control domain registration, transfer, renewal, and management operations.

## Module Structure

### Basic Registrar Module Template

```php
<?php
/**
 * WHMCS Registrar Module
 * 
 * Registrar: Your Registrar
 * Version: 1.0
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function yourregistrar_MetaData()
{
    return [
        'DisplayName' => 'Your Registrar',
        'APIVersion' => '1.0',
        'RequiresServer' => false,
        'DefaultRegistrar' => true,
        'Parameters' => [
            '/domain/sld/tld',
        ],
    ];
}

function yourregistrar_ConfigArray()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Your Registrar',
        ],
        'APIKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'text',
            'Size' => '50',
        ],
        'APIUser' => [
            'FriendlyName' => 'API Username',
            'Type' => 'text',
            'Size' => '30',
        ],
        'TestMode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
        ],
    ];
}
```

## Domain Registration

### RegisterDomain()

```php
/**
 * Register domain
 */
function yourregistrar_RegisterDomain(array $params)
{
    try {
        $api = new RegistrarAPI($params['config']);
        
        $result = $api->register([
            'sld' => $params['sld'],
            'tld' => $params['tld'],
            'registration_period' => $params['regperiod'],
            'registrant' => [
                'first_name' => $params['firstname'],
                'last_name' => $params['lastname'],
                'email' => $params['email'],
                'company' => $params['companyname'],
                'address1' => $params['address1'],
                'city' => $params['city'],
                'state' => $params['state'],
                'postcode' => $params['postcode'],
                'country' => $params['country'],
                'phone' => $params['phonenumber'],
            ],
            'nameservers' => [
                $params['ns1'],
                $params['ns2'],
                $params['ns3'] ?? null,
                $params['ns4'] ?? null,
            ],
        ]);
        
        return [
            'success' => true,
            'domainid' => $result['domain_id'],
        ];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### TransferDomain()

```php
/**
 * Transfer domain
 */
function yourregistrar_TransferDomain(array $params)
{
    try {
        $api = new RegistrarAPI($params['config']);
        
        $result = $api->transfer([
            'sld' => $params['sld'],
            'tld' => $params['tld'],
            'auth_code' => $params['transfersecret'],
            'registrant' => [
                'email' => $params['email'],
            ],
        ]);
        
        return [
            'success' => true,
            'transfer_id' => $result['transfer_id'],
        ];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### RenewDomain()

```php
/**
 * Renew domain
 */
function yourregistrar_RenewDomain(array $params)
{
    try {
        $api = new RegistrarAPI($params['config']);
        
        $result = $api->renew([
            'sld' => $params['sld'],
            'tld' => $params['tld'],
            'period' => $params['regperiod'],
        ]);
        
        return ['success' => true];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

## Domain Management

### GetDNS()

```php
/**
 * Get DNS records
 */
function yourregistrar_GetDNS(array $params)
{
    try {
        $api = new RegistrarAPI($params['config']);
        
        $result = $api->getDNS([
            'sld' => $params['sld'],
            'tld' => $params['tld'],
        ]);
        
        return [
            'success' => true,
            'dnsrecords' => $result['records'],
        ];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### SaveDNS()

```php
/**
 * Save DNS records
 */
function yourregistrar_SaveDNS(array $params)
{
    try {
        $api = new RegistrarAPI($params['config']);
        
        $api->saveDNS([
            'sld' => $params['sld'],
            'tld' => $params['tld'],
            'records' => $params['dnsrecords'],
        ]);
        
        return ['success' => true];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### GetNameservers()

```php
/**
 * Get nameservers
 */
function yourregistrar_GetNameservers(array $params)
{
    try {
        $api = new RegistrarAPI($params['config']);
        
        $result = $api->getNameservers([
            'sld' => $params['sld'],
            'tld' => $params['tld'],
        ]);
        
        return [
            'success' => true,
            'ns1' => $result['ns1'],
            'ns2' => $result['ns2'],
            'ns3' => $result['ns3'] ?? null,
            'ns4' => $result['ns4'] ?? null,
        ];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### SaveNameservers()

```php
/**
 * Save nameservers
 */
function yourregistrar_SaveNameservers(array $params)
{
    try {
        $api = new RegistrarAPI($params['config']);
        
        $api->setNameservers([
            'sld' => $params['sld'],
            'tld' => $params['tld'],
            'nameservers' => [
                $params['ns1'],
                $params['ns2'],
                $params['ns3'] ?? null,
                $params['ns4'] ?? null,
            ],
        ]);
        
        return ['success' => true];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

## Contact Management

### GetContactDetails()

```php
/**
 * Get contact details
 */
function yourregistrar_GetContactDetails(array $params)
{
    try {
        $api = new RegistrarAPI($params['config']);
        
        $result = $api->getContacts([
            'sld' => $params['sld'],
            'tld' => $params['tld'],
        ]);
        
        return [
            'success' => true,
            'Registrant' => $result['registrant'],
            'Admin' => $result['admin'],
            'Tech' => $result['tech'],
            'Billing' => $result['billing'],
        ];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### SaveContactDetails()

```php
/**
 * Save contact details
 */
function yourregistrar_SaveContactDetails(array $params)
{
    try {
        $api = new RegistrarAPI($params['config']);
        
        $api->setContacts([
            'sld' => $params['sld'],
            'tld' => $params['tld'],
            'contacts' => $params['contacts'],
        ]);
        
        return ['success' => true];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

## Domain Status

### GetDomainInformation()

```php
/**
 * Get domain information
 */
function yourregistrar_GetDomainInformation(array $params)
{
    try {
        $api = new RegistrarAPI($params['config']);
        
        $result = $api->getDomainInfo([
            'sld' => $params['sld'],
            'tld' => $params['tld'],
        ]);
        
        return [
            'success' => true,
            'registrationdate' => $result['created_date'],
            'expirydate' => $result['expiry_date'],
            'domainstatus' => $result['status'],
            'dnsmanage' => $result['dns_enabled'],
            'emailforward' => $result['email_forwarding_enabled'],
            'idprotect' => $result['privacy_enabled'],
        ];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### Sync()

```php
/**
 * Sync domain status
 */
function yourregistrar_Sync(array $params)
{
    try {
        $api = new RegistrarAPI($params['config']);
        
        $result = $api->getDomainInfo([
            'sld' => $params['sld'],
            'tld' => $params['tld'],
        ]);
        
        return [
            'success' => true,
            'domainstatus' => $result['status'],
            'expirydate' => $result['expiry_date'],
            'nextduedate' => $result['next_due_date'],
        ];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

## Additional Features

### CheckAvailability()

```php
/**
 * Check domain availability
 */
function yourregistrar_CheckAvailability(array $params)
{
    try {
        $api = new RegistrarAPI($params['config']);
        
        $result = $api->checkAvailability([
            'sld' => $params['sld'],
            'tld' => $params['tld'],
        ]);
        
        return [
            'success' => true,
            'available' => $result['available'],
            'price' => $result['price'],
        ];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### GetRegistrarLock()

```php
/**
 * Get transfer lock status
 */
function yourregistrar_GetRegistrarLock(array $params)
{
    try {
        $api = new RegistrarAPI($params['config']);
        
        $result = $api->getLockStatus([
            'sld' => $params['sld'],
            'tld' => $params['tld'],
        ]);
        
        return [
            'success' => true,
            'lockstatus' => $result['locked'] ? 'locked' : 'unlocked',
        ];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### SaveRegistrarLock()

```php
/**
 * Set transfer lock
 */
function yourregistrar_SaveRegistrarLock(array $params)
{
    try {
        $api = new RegistrarAPI($params['config']);
        
        $api->setLock([
            'sld' => $params['sld'],
            'tld' => $params['tld'],
            'lock' => $params['lockenabled'] === 'locked',
        ]);
        
        return ['success' => true];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

## Related Documentation

- [whmcs-functions-domains.md](../functions/whmcs-functions-domains.md)
- [whmcs-schema-registrars.md](../schema/whmcs-schema-registrars.md)