# WHMCS Service Functions

Complete reference for service/hosting account management functions in WHMCS.

## Overview

WHMCS provides comprehensive service management including provisioning, suspension, termination, and lifecycle management.

## Service CRUD Operations

### createService()

Creates a new service/hosting account.

```php
/**
 * Create a new service
 * 
 * @param array $data Service data
 * @return int Service ID
 */
function createService(array $data): int
{
    return Capsule::table('tblhosting')->insertGetId([
        'userid' => $data['clientid'],
        'orderid' => $data['orderid'] ?? 0,
        'packageid' => $data['productid'],
        'server' => $data['serverid'] ?? 0,
        'regdate' => date('Y-m-d H:i:s'),
        'domainstatus' => 'Pending',
        'username' => $data['username'] ?? '',
        'password' => $data['password'] ?? '',
        'subscriptionid' => $data['subscriptionid'] ?? '',
        'firstpaymentamount' => $data['first_payment'] ?? 0,
        'amount' => $data['recurring_amount'] ?? 0,
        ' billingcycle' => $data['billingcycle'] ?? 'Monthly',
        'nextinvoicedate' => $data['next_invoice_date'] ?? null,
        'nextduedate' => $data['next_due_date'] ?? null,
        'termination_date' => null,
        'diskusage' => 0,
        'disklimit' => $data['disk_limit'] ?? 0,
        'bwusage' => 0,
        'bwlimit' => $data['bandwidth_limit'] ?? 0,
        'module' => $data['module'] ?? '',
    ]);
}
```

**Example:**
```php
$serviceId = createService([
    'clientid' => 123,
    'orderid' => 1001,
    'productid' => 1,
    'serverid' => 5,
    'username' => 'user_example',
    'password' => encryptPassword('SecurePass123'),
    'first_payment' => 9.99,
    'recurring_amount' => 9.99,
    'billingcycle' => 'Monthly',
    'next_invoice_date' => date('Y-m-d', strtotime('+30 days')),
    'next_due_date' => date('Y-m-d', strtotime('+30 days'))
]);
```

### getService()

Retrieves a service by ID.

```php
/**
 * Get service by ID
 * 
 * @param int $serviceId Service ID
 * @return array|null Service data
 */
function getService(int $serviceId): ?array
{
    $result = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();
    
    return $result ? (array) $result : null;
}
```

### getServiceByDomain()

Retrieves a service by domain.

```php
/**
 * Get service by domain
 * 
 * @param string $domain Domain name
 * @return array|null Service data
 */
function getServiceByDomain(string $domain): ?array
{
    $result = Capsule::table('tblhosting')
        ->where('domain', $domain)
        ->first();
    
    return $result ? (array) $result : null;
}
```

**Example:**
```php
$service = getServiceByDomain('example.com');

if ($service) {
    echo "Service ID: {$service['id']} - Status: {$service['domainstatus']}";
}
```

### getServices()

Retrieves services with filtering.

```php
/**
 * Get services with filters
 * 
 * @param array $filters Filter options
 * @param int $limit Number of records
 * @param int $offset Starting offset
 * @return array Services
 */
function getServices(array $filters = [], int $limit = 50, int $offset = 0): array
{
    $query = Capsule::table('tblhosting')
        ->select('tblhosting.*', 'tblclients.firstname', 'tblclients.lastname', 'tblproducts.name as product_name')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tblhosting.userid')
        ->leftJoin('tblproducts', 'tblproducts.id', '=', 'tblhosting.packageid')
        ->orderBy('tblhosting.id', 'desc');
    
    if (!empty($filters['clientId'])) {
        $query->where('tblhosting.userid', $filters['clientId']);
    }
    
    if (!empty($filters['status'])) {
        $query->where('tblhosting.domainstatus', $filters['status']);
    }
    
    if (!empty($filters['productId'])) {
        $query->where('tblhosting.packageid', $filters['productId']);
    }
    
    if (!empty($filters['serverId'])) {
        $query->where('tblhosting.server', $filters['serverId']);
    }
    
    return $query->limit($limit)->offset($offset)->get()->toArray();
}
```

**Example:**
```php
// Get all suspended services on server 5
$suspended = getServices([
    'status' => 'Suspended',
    'serverId' => 5
]);

// Get client's active services
$clientServices = getServices([
    'clientId' => 123,
    'status' => 'Active'
]);
```

### updateService()

Updates an existing service.

```php
/**
 * Update a service
 * 
 * @param int $serviceId Service ID
 * @param array $data Updated data
 * @return bool Success status
 */
function updateService(int $serviceId, array $data): bool
{
    $data['updated_at'] = date('Y-m-d H:i:s');
    
    return Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->update($data) > 0;
}
```

**Example:**
```php
updateService(456, [
    'disklimit' => 50000,
    'bwlimit' => 500000,
    'username' => 'new_username'
]);
```

## Service Lifecycle

### provisionService()

Provisions a service using the module.

```php
/**
 * Provision a service
 * 
 * @param int $serviceId Service ID
 * @param bool $sendEmail Send welcome email
 * @return array Result
 */
function provisionService(int $serviceId, bool $sendEmail = true): array
{
    $service = getService($serviceId);
    
    if (!$service) {
        return ['success' => false, 'error' => 'Service not found'];
    }
    
    $product = getProduct($service['packageid']);
    
    if (empty($product['module'])) {
        return ['success' => false, 'error' => 'No provisioning module configured'];
    }
    
    // Load module
    $moduleInterface = new WHMCS\Module\Server();
    $moduleInterface->load($product['module']);
    
    // Prepare params
    $params = [
        'domain' => $service['domain'],
        'username' => $service['username'],
        'password' => decryptPassword($service['password']),
        'producttype' => $product['type'],
        ' module' => $product['module'],
        ' pid' => $service['packageid'],
        ' serviceid' => $serviceId,
        ' clientid' => $service['userid'],
        ' serverid' => $service['server'],
    ];
    
    // Call create account
    $result = $moduleInterface->createAccount($params);
    
    if ($result === true) {
        updateService($serviceId, ['domainstatus' => 'Active']);
        
        if ($sendEmail) {
            sendServiceWelcomeEmail($serviceId);
        }
        
        return ['success' => true, 'service_id' => $serviceId];
    }
    
    return ['success' => false, 'error' => $moduleInterface->getOutput()];
}
```

### suspendService()

Suspends a service.

```php
/**
 * Suspend a service
 * 
 * @param int $serviceId Service ID
 * @param string $reason Suspension reason
 * @return array Result
 */
function suspendService(int $serviceId, string $reason = ''): array
{
    $service = getService($serviceId);
    
    if ($service['domainstatus'] === 'Suspended') {
        return ['success' => false, 'error' => 'Already suspended'];
    }
    
    $product = getProduct($service['packageid']);
    
    if (!empty($product['module'])) {
        $moduleInterface = new WHMCS\Module\Server();
        $moduleInterface->load($product['module']);
        
        $params = buildModuleParams($service);
        $result = $moduleInterface->suspendAccount($params);
        
        if ($result !== true) {
            return ['success' => false, 'error' => $moduleInterface->getOutput()];
        }
    }
    
    updateService($serviceId, [
        'domainstatus' => 'Suspended',
        'suspendreason' => $reason
    ]);
    
    logActivity("Service #{$serviceId} suspended: {$reason}", 0);
    
    return ['success' => true];
}
```

### unsuspendService()

Unsuspends a service.

```php
/**
 * Unsuspend a service
 * 
 * @param int $serviceId Service ID
 * @return array Result
 */
function unsuspendService(int $serviceId): array
{
    $service = getService($serviceId);
    
    if ($service['domainstatus'] !== 'Suspended') {
        return ['success' => false, 'error' => 'Not suspended'];
    }
    
    $product = getProduct($service['packageid']);
    
    if (!empty($product['module'])) {
        $moduleInterface = new WHMCS\Module\Server();
        $moduleInterface->load($product['module']);
        
        $params = buildModuleParams($service);
        $result = $moduleInterface->unsuspendAccount($params);
        
        if ($result !== true) {
            return ['success' => false, 'error' => $moduleInterface->getOutput()];
        }
    }
    
    updateService($serviceId, [
        'domainstatus' => 'Active',
        'suspendreason' => ''
    ]);
    
    logActivity("Service #{$serviceId} unsuspended", 0);
    
    return ['success' => true];
}
```

### terminateService()

Terminates a service.

```php
/**
 * Terminate a service
 * 
 * @param int $serviceId Service ID
 * @param bool $terminateModule Terminate on module
 * @param bool $deleteFiles Delete associated files
 * @return array Result
 */
function terminateService(
    int $serviceId,
    bool $terminateModule = true,
    bool $deleteFiles = false
): array {
    $service = getService($serviceId);
    
    if (!$service) {
        return ['success' => false, 'error' => 'Service not found'];
    }
    
    // Terminate on module
    if ($terminateModule && !empty($service['module'])) {
        $moduleInterface = new WHMCS\Module\Server();
        $moduleInterface->load($service['module']);
        
        $params = buildModuleParams($service);
        $result = $moduleInterface->terminateAccount($params);
        
        if ($result !== true) {
            logActivity("Module termination failed for service #{$serviceId}", 0);
        }
    }
    
    // Update status
    updateService($serviceId, [
        'domainstatus' => 'Terminated',
        'termination_date' => date('Y-m-d H:i:s')
    ]);
    
    logActivity("Service #{$serviceId} terminated", 0);
    
    return ['success' => true];
}
```

### changeServicePackage()

Changes the package for a service.

```php
/**
 * Change service package
 * 
 * @param int $serviceId Service ID
 * @param int $newProductId New product ID
 * @param bool $prorate Prorate the difference
 * @return array Result
 */
function changeServicePackage(int $serviceId, int $newProductId, bool $prorate = true): array
{
    $service = getService($serviceId);
    $newProduct = getProduct($newProductId);
    
    if (!$service || !$newProduct) {
        return ['success' => false, 'error' => 'Invalid service or product'];
    }
    
    $oldProduct = getProduct($service['packageid']);
    $pricing = getProductPricing($newProductId, getClientCurrency($service['userid']));
    
    // Calculate prorated amount if needed
    $newAmount = $pricing[$service['billingcycle']] ?? 0;
    
    if ($prorate && $newAmount != $service['amount']) {
        $difference = $newAmount - $service['amount'];
        // Create prorate invoice...
    }
    
    // Update service
    updateService($serviceId, [
        'packageid' => $newProductId,
        'amount' => $newAmount,
    ]);
    
    // Call module change function if exists
    if (!empty($service['module'])) {
        $moduleInterface = new WHMCS\Module\Server();
        $moduleInterface->load($service['module']);
        
        $params = buildModuleParams($service);
        $moduleInterface->changePackage($params);
    }
    
    return ['success' => true, 'new_amount' => $newAmount];
}
```

## Service Password Management

```php
/**
 * Update service password
 * 
 * @param int $serviceId Service ID
 * @param string $newPassword New password
 * @return array Result
 */
function updateServicePassword(int $serviceId, string $newPassword): array
{
    $service = getService($serviceId);
    
    if (!$service) {
        return ['success' => false, 'error' => 'Service not found'];
    }
    
    $encrypted = encryptPassword($newPassword);
    
    updateService($serviceId, ['password' => $encrypted]);
    
    // Update on module
    if (!empty($service['module'])) {
        $moduleInterface = new WHMCS\Module\Server();
        $moduleInterface->load($service['module']);
        
        $params = buildModuleParams($service);
        $params['password'] = $newPassword;
        
        $moduleInterface->changePassword($params);
    }
    
    logActivity("Password changed for service #{$serviceId}", 0);
    
    return ['success' => true];
}
```

## Service Usage

```php
/**
 * Update service usage stats
 * 
 * @param int $serviceId Service ID
 * @param int $diskUsage Disk usage in MB
 * @param int $bandwidthUsage Bandwidth usage in MB
 * @return bool Success status
 */
function updateServiceUsage(int $serviceId, int $diskUsage, int $bandwidthUsage): bool
{
    return updateService($serviceId, [
        'diskusage' => $diskUsage,
        'bwusage' => $bandwidthUsage
    ]);
}

/**
 * Check service usage limits
 * 
 * @param int $serviceId Service ID
 * @return array Usage status
 */
function checkServiceUsage(int $serviceId): array
{
    $service = getService($serviceId);
    
    $diskPercent = $service['disklimit'] > 0 
        ? ($service['diskusage'] / $service['disklimit']) * 100 
        : 0;
    
    $bwPercent = $service['bwlimit'] > 0 
        ? ($service['bwusage'] / $service['bwlimit']) * 100 
        : 0;
    
    return [
        'service_id' => $serviceId,
        'disk_used' => $service['diskusage'],
        'disk_limit' => $service['disklimit'],
        'disk_percent' => round($diskPercent, 2),
        'disk_over' => $service['diskusage'] > $service['disklimit'],
        'bw_used' => $service['bwusage'],
        'bw_limit' => $service['bwlimit'],
        'bw_percent' => round($bwPercent, 2),
        'bw_over' => $service['bwusage'] > $service['bwlimit'],
    ];
}
```

## Service Renewals

```php
/**
 * Process service renewal
 * 
 * @param int $serviceId Service ID
 * @return bool Success status
 */
function processServiceRenewal(int $serviceId): bool
{
    $service = getService($serviceId);
    
    if (!$service || $service['domainstatus'] !== 'Active') {
        return false;
    }
    
    // Calculate next due date based on billing cycle
    $nextDueDate = calculateNextBillingDate($service['nextduedate'], $service['billingcycle']);
    
    updateService($serviceId, [
        'nextduedate' => $nextDueDate,
        'nextinvoicedate' => $nextDueDate
    ]);
    
    // Create renewal invoice
    $invoiceId = createInvoice([
        'clientid' => $service['userid'],
        'date' => date('Y-m-d'),
        'duedate' => $nextDueDate,
        'subtotal' => $service['amount'],
        'total' => $service['amount'],
        'status' => 'Unpaid'
    ]);
    
    addInvoiceItem([
        'invoiceid' => $invoiceId,
        'clientid' => $service['userid'],
        'type' => 'Hosting',
        'relid' => $serviceId,
        'description' => "{$service['domain']} - {$service['billingcycle']} Renewal",
        'amount' => $service['amount']
    ]);
    
    return true;
}

/**
 * Calculate next billing date
 * 
 * @param string $currentDate Current date
 * @param string $billingCycle Billing cycle
 * @return string Next date
 */
function calculateNextBillingDate(string $currentDate, string $billingCycle): string
{
    $intervals = [
        'Monthly' => '+1 month',
        'Quarterly' => '+3 months',
        'SemiAnnually' => '+6 months',
        'Annually' => '+1 year',
        'Biennially' => '+2 years',
        'Triennially' => '+3 years',
    ];
    
    $interval = $intervals[$billingCycle] ?? '+1 month';
    
    return date('Y-m-d', strtotime($interval, strtotime($currentDate)));
}
```

## Service Search

```php
/**
 * Search services
 * 
 * @param string $term Search term
 * @param array $fields Fields to search
 * @return array Matching services
 */
function searchServices(string $term, array $fields = []): array
{
    $defaultFields = ['domain', 'username'];
    $fields = $fields ?: $defaultFields;
    
    $query = Capsule::table('tblhosting')
        ->select('tblhosting.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tblhosting.userid');
    
    foreach ($fields as $field) {
        $query->orWhere('tblhosting.' . $field, 'like', '%' . $term . '%');
    }
    
    return $query->limit(50)->get()->toArray();
}
```

## Service Module Actions

```php
/**
 * Execute module action on service
 * 
 * @param int $serviceId Service ID
 * @param string $action Module action
 * @param array $params Additional parameters
 * @return array Result
 */
function executeServiceModuleAction(int $serviceId, string $action, array $params = []): array
{
    $service = getService($serviceId);
    
    if (!$service || empty($service['module'])) {
        return ['success' => false, 'error' => 'Module not configured'];
    }
    
    $moduleInterface = new WHMCS\Module\Server();
    $moduleInterface->load($service['module']);
    
    $moduleParams = buildModuleParams($service);
    $moduleParams = array_merge($moduleParams, $params);
    
    $moduleActions = [
        'create' => 'createAccount',
        'terminate' => 'terminateAccount',
        'suspend' => 'suspendAccount',
        'unsuspend' => 'unsuspendAccount',
        'change_password' => 'changePassword',
        'change_package' => 'changePackage',
    ];
    
    if (!isset($moduleActions[$action])) {
        return ['success' => false, 'error' => 'Invalid action'];
    }
    
    $method = $moduleActions[$action];
    $result = $moduleInterface->$method($moduleParams);
    
    return [
        'success' => $result === true,
        'output' => $moduleInterface->getOutput(),
        'errors' => $moduleInterface->getErrors(),
    ];
}
```

## Related Functions

- [whmcs-functions-orders.md](whmcs-functions-orders.md) - Order processing
- [whmcs-functions-products.md](whmcs-functions-products.md) - Product management
- [whmcs-schema-services.md](whmcs-schema-services.md) - Service database schema