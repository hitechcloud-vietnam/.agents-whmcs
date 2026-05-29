# WHMCS Domain Functions

Complete reference for domain management functions in WHMCS.

## Overview

WHMCS provides comprehensive domain management including registration, transfer, renewal, and DNS management through registrar modules.

## Domain CRUD Operations

### createDomain()

Creates a new domain entry.

```php
/**
 * Create a new domain
 * 
 * @param array $data Domain data
 * @return int Domain ID
 */
function createDomain(array $data): int
{
    return Capsule::table('tbldomains')->insertGetId([
        'userid' => $data['clientid'],
        'domain' => $data['domain'],
        'registrationdate' => $data['registration_date'] ?? date('Y-m-d'),
        'nextduedate' => $data['next_due_date'] ?? null,
        'nextinvoicedate' => $data['next_invoice_date'] ?? null,
        'expirydate' => $data['expiry_date'] ?? null,
        'domainstatus' => $data['status'] ?? 'Pending',
        'registrationperiod' => $data['registration_period'] ?? 1,
        'dns管理' => $data['dnsmanagement'] ?? 0,
        'emailforwarding' => $data['emailforwarding'] ?? 0,
        'idprotection' => $data['idprotection'] ?? 0,
        'registrar' => $data['registrar'] ?? '',
        'registrationdata' => $data['registration_data'] ?? '',
        'recurringamount' => $data['recurring_amount'] ?? 0,
        'paymentmethod' => $data['paymentmethod'] ?? '',
        'orderid' => $data['orderid'] ?? 0,
        'subscriptionid' => $data['subscriptionid'] ?? '',
        'type' => $data['type'] ?? 'Register',
    ]);
}
```

**Example:**
```php
$domainId = createDomain([
    'clientid' => 123,
    'domain' => 'example.com',
    'registration_date' => date('Y-m-d'),
    'next_due_date' => date('Y-m-d', strtotime('+1 year')),
    'expiry_date' => date('Y-m-d', strtotime('+1 year')),
    'status' => 'Active',
    'registration_period' => 1,
    'registrar' => 'enom',
    'recurring_amount' => 14.99,
    'type' => 'Register'
]);
```

### getDomain()

Retrieves a domain by ID.

```php
/**
 * Get domain by ID
 * 
 * @param int $domainId Domain ID
 * @return array|null Domain data
 */
function getDomain(int $domainId): ?array
{
    $result = Capsule::table('tbldomains')
        ->where('id', $domainId)
        ->first();
    
    return $result ? (array) $result : null;
}
```

### getDomainByName()

Retrieves a domain by name.

```php
/**
 * Get domain by name
 * 
 * @param string $domain Domain name
 * @return array|null Domain data
 */
function getDomainByName(string $domain): ?array
{
    $result = Capsule::table('tbldomains')
        ->where('domain', $domain)
        ->first();
    
    return $result ? (array) $result : null;
}
```

**Example:**
```php
$domain = getDomainByName('example.com');

if ($domain) {
    echo "Domain: {$domain['domain']}\n";
    echo "Status: {$domain['domainstatus']}\n";
    echo "Expires: {$domain['expirydate']}";
}
```

### getDomains()

Retrieves domains with filtering.

```php
/**
 * Get domains with filters
 * 
 * @param array $filters Filter options
 * @param int $limit Number of records
 * @param int $offset Starting offset
 * @return array Domains
 */
function getDomains(array $filters = [], int $limit = 50, int $offset = 0): array
{
    $query = Capsule::table('tbldomains')
        ->select('tbldomains.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tbldomains.userid')
        ->orderBy('tbldomains.id', 'desc');
    
    if (!empty($filters['clientId'])) {
        $query->where('tbldomains.userid', $filters['clientId']);
    }
    
    if (!empty($filters['status'])) {
        $query->where('tbldomains.domainstatus', $filters['status']);
    }
    
    if (!empty($filters['registrar'])) {
        $query->where('tbldomains.registrar', $filters['registrar']);
    }
    
    if (!empty($filters['expiringBefore'])) {
        $query->where('tbldomains.expirydate', '<=', $filters['expiringBefore']);
    }
    
    return $query->limit($limit)->offset($offset)->get()->toArray();
}
```

**Example:**
```php
// Get all domains expiring within 30 days
$expiring = getDomains([
    'expiringBefore' => date('Y-m-d', strtotime('+30 days')),
    'status' => 'Active'
]);

// Get client's domains
$clientDomains = getDomains(['clientId' => 123]);
```

### updateDomain()

Updates an existing domain.

```php
/**
 * Update a domain
 * 
 * @param int $domainId Domain ID
 * @param array $data Updated data
 * @return bool Success status
 */
function updateDomain(int $domainId, array $data): bool
{
    $data['updated_at'] = date('Y-m-d H:i:s');
    
    return Capsule::table('tbldomains')
        ->where('id', $domainId)
        ->update($data) > 0;
}
```

**Example:**
```php
updateDomain(456, [
    'dnsmanagement' => 1,
    'idprotection' => 1,
    'nextduedate' => date('Y-m-d', strtotime('+2 years'))
]);
```

## Domain Registrar Operations

### registerDomain()

Registers a domain through the registrar.

```php
/**
 * Register a domain
 * 
 * @param int $domainId Domain ID
 * @param array $contactInfo Contact information
 * @return array Result
 */
function registerDomain(int $domainId, array $contactInfo = []): array
{
    $domain = getDomain($domainId);
    
    if (!$domain) {
        return ['success' => false, 'error' => 'Domain not found'];
    }
    
    $registrarModule = new WHMCS\Module\Registrar();
    $registrarModule->load($domain['registrar']);
    
    $params = [
        'domainid' => $domainId,
        'sld' => getSld($domain['domain']),
        'tld' => getTld($domain['domain']),
        'regperiod' => $domain['registrationperiod'],
        'registrar' => $domain['registrar'],
        'dnsmanagement' => $domain['dns管理'],
        'emailforwarding' => $domain['emailforwarding'],
        'idprotection' => $domain['idprotection'],
        'contacts' => $contactInfo,
    ];
    
    $result = $registrarModule->registerDomain($params);
    
    if ($result === true) {
        updateDomain($domainId, [
            'domainstatus' => 'Active',
            'registrationdate' => date('Y-m-d'),
            'expirydate' => calculateExpiryDate(date('Y-m-d'), $domain['registrationperiod'])
        ]);
        
        logActivity("Domain {$domain['domain']} registered successfully", 0);
        
        return ['success' => true, 'domain_id' => $domainId];
    }
    
    return ['success' => false, 'error' => $registrarModule->getOutput()];
}
```

### transferDomain()

Initiates a domain transfer.

```php
/**
 * Transfer a domain
 * 
 * @param int $domainId Domain ID
 * @param string $authCode Transfer authorization code
 * @return array Result
 */
function transferDomain(int $domainId, string $authCode): array
{
    $domain = getDomain($domainId);
    
    if (!$domain) {
        return ['success' => false, 'error' => 'Domain not found'];
    }
    
    $registrarModule = new WHMCS\Module\Registrar();
    $registrarModule->load($domain['registrar']);
    
    $params = [
        'domainid' => $domainId,
        'sld' => getSld($domain['domain']),
        'tld' => getTld($domain['domain']),
        'transfersecret' => $authCode,
    ];
    
    $result = $registrarModule->transferDomain($params);
    
    if ($result === true) {
        updateDomain($domainId, [
            'domainstatus' => 'Pending Transfer',
            'transfersecret' => $authCode
        ]);
        
        return ['success' => true, 'message' => 'Transfer initiated'];
    }
    
    return ['success' => false, 'error' => $registrarModule->getOutput()];
}
```

### renewDomain()

Renews a domain.

```php
/**
 * Renew a domain
 * 
 * @param int $domainId Domain ID
 * @param int $years Number of years to renew
 * @return array Result
 */
function renewDomain(int $domainId, int $years = 1): array
{
    $domain = getDomain($domainId);
    
    if (!$domain) {
        return ['success' => false, 'error' => 'Domain not found'];
    }
    
    $registrarModule = new WHMCS\Module\Registrar();
    $registrarModule->load($domain['registrar']);
    
    $params = [
        'domainid' => $domainId,
        'sld' => getSld($domain['domain']),
        'tld' => getTld($domain['domain']),
        'regperiod' => $years,
    ];
    
    $result = $registrarModule->renewDomain($params);
    
    if ($result === true) {
        $newExpiry = calculateExpiryDate($domain['expirydate'], $years);
        
        updateDomain($domainId, [
            'expirydate' => $newExpiry,
            'nextduedate' => $newExpiry,
            'registrationperiod' => $domain['registrationperiod'] + $years
        ]);
        
        logActivity("Domain {$domain['domain']} renewed for {$years} year(s)", 0);
        
        return ['success' => true, 'new_expiry' => $newExpiry];
    }
    
    return ['success' => false, 'error' => $registrarModule->getOutput()];
}
```

### releaseDomain()

Releases a domain to another registrar.

```php
/**
 * Release a domain (push to another registrar)
 * 
 * @param int $domainId Domain ID
 * @param string $tag EPP transfer tag
 * @return array Result
 */
function releaseDomain(int $domainId, string $tag): array
{
    $domain = getDomain($domainId);
    
    if (!$domain) {
        return ['success' => false, 'error' => 'Domain not found'];
    }
    
    $registrarModule = new WHMCS\Module\Registrar();
    $registrarModule->load($domain['registrar']);
    
    $params = [
        'domainid' => $domainId,
        'sld' => getSld($domain['domain']),
        'tld' => getTld($domain['domain']),
        'transfer_tag' => $tag,
    ];
    
    $result = $registrarModule->releaseDomain($params);
    
    if ($result === true) {
        updateDomain($domainId, ['domainstatus' => 'Transferred']);
        
        return ['success' => true];
    }
    
    return ['success' => false, 'error' => $registrarModule->getOutput()];
}
```

## Domain DNS Management

```php
/**
 * Get DNS records for a domain
 * 
 * @param int $domainId Domain ID
 * @return array DNS records
 */
function getDomainDns(int $domainId): array
{
    $domain = getDomain($domainId);
    
    if (!$domain) {
        return [];
    }
    
    $registrarModule = new WHMCS\Module\Registrar();
    $registrarModule->load($domain['registrar']);
    
    $params = [
        'domainid' => $domainId,
        'sld' => getSld($domain['domain']),
        'tld' => getTld($domain['domain']),
    ];
    
    return $registrarModule->getDNS($params) ?: [];
}

/**
 * Update DNS records
 * 
 * @param int $domainId Domain ID
 * @param array $records DNS records
 * @return array Result
 */
function updateDomainDns(int $domainId, array $records): array
{
    $domain = getDomain($domainId);
    
    if (!$domain) {
        return ['success' => false, 'error' => 'Domain not found'];
    }
    
    $registrarModule = new WHMCS\Module\Registrar();
    $registrarModule->load($domain['registrar']);
    
    $params = [
        'domainid' => $domainId,
        'sld' => getSld($domain['domain']),
        'tld' => getTld($domain['domain']),
        'dnsrecords' => $records,
    ];
    
    $result = $registrarModule->saveDNS($params);
    
    if ($result === true) {
        logActivity("DNS updated for {$domain['domain']}", 0);
    }
    
    return [
        'success' => $result === true,
        'error' => $result !== true ? $registrarModule->getOutput() : null
    ];
}
```

**Example:**
```php
updateDomainDns(456, [
    ['name' => '@', 'type' => 'A', 'priority' => 0, 'address' => '192.168.1.1'],
    ['name' => 'www', 'type' => 'CNAME', 'priority' => 0, 'address' => 'example.com'],
    ['name' => '@', 'type' => 'MX', 'priority' => 10, 'address' => 'mail.example.com'],
]);
```

## Domain Contact Management

```php
/**
 * Get domain contacts
 * 
 * @param int $domainId Domain ID
 * @return array Contacts
 */
function getDomainContacts(int $domainId): array
{
    return Capsule::table('tbldomaincontacts')
        ->where('domainid', $domainId)
        ->get()
        ->toArray();
}

/**
 * Update domain contacts
 * 
 * @param int $domainId Domain ID
 * @param array $contacts Contacts by type
 * @return bool Success status
 */
function updateDomainContacts(int $domainId, array $contacts): bool
{
    foreach ($contacts as $type => $contactData) {
        Capsule::table('tbldomaincontacts')->updateOrInsert(
            ['domainid' => $domainId, 'contacttype' => $type],
            [
                'firstname' => $contactData['firstname'] ?? '',
                'lastname' => $contactData['lastname'] ?? '',
                'email' => $contactData['email'] ?? '',
                'companyname' => $contactData['companyname'] ?? '',
                'address1' => $contactData['address1'] ?? '',
                'address2' => $contactData['address2'] ?? '',
                'city' => $contactData['city'] ?? '',
                'state' => $contactData['state'] ?? '',
                'postcode' => $contactData['postcode'] ?? '',
                'country' => $contactData['country'] ?? '',
                'phonenumber' => $contactData['phonenumber'] ?? '',
            ]
        );
    }
    
    return true;
}
```

## Domain Availability Check

```php
/**
 * Check domain availability
 * 
 * @param string $domain Domain name
 * @param string $registrar Registrar module
 * @return array Availability result
 */
function checkDomainAvailability(string $domain, string $registrar = ''): array
{
    $sld = getSld($domain);
    $tld = getTld($domain);
    
    if ($registrar) {
        $module = new WHMCS\Module\Registrar();
        $module->load($registrar);
        
        $params = ['sld' => $sld, 'tld' => $tld];
        $available = $module->checkAvailability($params);
        
        return [
            'domain' => $domain,
            'available' => $available,
            'registrar' => $registrar
        ];
    }
    
    // Check multiple registrars
    $registrars = Capsule::table('tblmodules')
        ->where('type', 'registrar')
        ->where('active', 1)
        ->get();
    
    $results = [];
    foreach ($registrars as $reg) {
        $module = new WHMCS\Module\Registrar();
        $module->load($reg->name);
        
        $params = ['sld' => $sld, 'tld' => $tld];
        $available = $module->checkAvailability($params);
        
        $results[] = [
            'registrar' => $reg->name,
            'available' => $available
        ];
    }
    
    return [
        'domain' => $domain,
        'results' => $results
    ];
}

/**
 * Check multiple domains availability
 * 
 * @param array $domains Array of domain names
 * @param string $registrar Registrar module
 * @return array Bulk results
 */
function checkDomainsAvailability(array $domains, string $registrar = ''): array
{
    $results = [];
    
    foreach ($domains as $domain) {
        $results[$domain] = checkDomainAvailability($domain, $registrar);
    }
    
    return $results;
}
```

## Domain Helper Functions

```php
/**
 * Extract SLD from domain
 * 
 * @param string $domain Full domain name
 * @return string SLD (second-level domain)
 */
function getSld(string $domain): string
{
    $parts = explode('.', $domain);
    return $parts[0];
}

/**
 * Extract TLD from domain
 * 
 * @param string $domain Full domain name
 * @return string TLD (top-level domain)
 */
function getTld(string $domain): string
{
    $parts = explode('.', $domain);
    return '.' . $parts[count($parts) - 1];
}

/**
 * Calculate expiry date
 * 
 * @param string $startDate Starting date
 * @param int $years Number of years
 * @return string Expiry date
 */
function calculateExpiryDate(string $startDate, int $years): string
{
    return date('Y-m-d', strtotime("+{$years} years", strtotime($startDate)));
}
```

## Domain Status Management

```php
/**
 * Update domain status
 * 
 * @param int $domainId Domain ID
 * @param string $status New status
 * @return bool Success status
 */
function updateDomainStatus(int $domainId, string $status): bool
{
    $validStatuses = ['Pending', 'Active', 'Pending Transfer', 'Suspended', 
                      'Cancelled', 'Expired', 'Transferred'];
    
    if (!in_array($status, $validStatuses)) {
        return false;
    }
    
    return updateDomain($domainId, ['domainstatus' => $status]);
}
```

**Example:**
```php
updateDomainStatus(456, 'Pending Transfer');
updateDomainStatus(456, 'Active');
```

## Domain Search

```php
/**
 * Search domains
 * 
 * @param string $term Search term
 * @param int $limit Result limit
 * @return array Matching domains
 */
function searchDomains(string $term, int $limit = 50): array
{
    return Capsule::table('tbldomains')
        ->select('tbldomains.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tbldomains.userid')
        ->where('tbldomains.domain', 'like', '%' . $term . '%')
        ->limit($limit)
        ->get()
        ->toArray();
}
```

## Domain Renewal Processing

```php
/**
 * Process domain renewals
 * 
 * @param int $daysBefore Days before expiry to process
 * @return array Results
 */
function processDomainRenewals(int $daysBefore = 14): array
{
    $expiringDomains = getDomains([
        'expiringBefore' => date('Y-m-d', strtotime("+{$daysBefore} days")),
        'status' => 'Active'
    ], 500);
    
    $processed = 0;
    $failed = 0;
    
    foreach ($expiringDomains as $domain) {
        // Create renewal invoice
        $invoiceId = createInvoice([
            'clientid' => $domain['userid'],
            'date' => date('Y-m-d'),
            'duedate' => $domain['nextduedate'],
            'subtotal' => $domain['recurringamount'],
            'total' => $domain['recurringamount'],
            'status' => 'Unpaid'
        ]);
        
        addInvoiceItem([
            'invoiceid' => $invoiceId,
            'clientid' => $domain['userid'],
            'type' => 'Domain',
            'relid' => $domain['id'],
            'description' => "Domain Renewal - {$domain['domain']}",
            'amount' => $domain['recurringamount']
        ]);
        
        $processed++;
    }
    
    return [
        'processed' => $processed,
        'failed' => $failed
    ];
}
```

## Related Functions

- [whmcs-functions-orders.md](whmcs-functions-orders.md) - Domain order processing
- [whmcs-functions-registrars.md](whmcs-schema-registrars.md) - Registrar schema
- [whmcs-module-registrar-api.md](whmcs-module-registrar-api.md) - Registrar module API