# WHMCS Client Functions

Complete reference for client management functions in WHMCS.

## Overview

WHMCS provides comprehensive client management including creation, retrieval, updates, and relationship handling.

## Client CRUD Operations

### createClient()

Creates a new client account.

```php
/**
 * Create a new client
 * 
 * @param array $data Client data
 * @return int Client ID
 */
function createClient(array $data): int
{
    return Capsule::table('tblclients')->insertGetId([
        'email' => $data['email'],
        'firstname' => $data['firstname'] ?? '',
        'lastname' => $data['lastname'] ?? '',
        'companyname' => $data['companyname'] ?? '',
        'phonenumber' => $data['phonenumber'] ?? '',
        'password' => $data['password'] ?? '', // Will be hashed
        'currency' => $data['currency'] ?? 0,
        'groupid' => $data['groupid'] ?? 0,
        'language' => $data['language'] ?? '',
        'address1' => $data['address1'] ?? '',
        'address2' => $data['address2'] ?? '',
        'city' => $data['city'] ?? '',
        'state' => $data['state'] ?? '',
        'postcode' => $data['postcode'] ?? '',
        'country' => $data['country'] ?? 'US',
        'taxexempt' => $data['taxexempt'] ?? 0,
        'notes' => $data['notes'] ?? '',
        'created_at' => date('Y-m-d H:i:s'),
        'lastlogin' => date('Y-m-d H:i:s'),
    ]);
}
```

**Example:**
```php
$clientId = createClient([
    'email' => 'john.doe@example.com',
    'firstname' => 'John',
    'lastname' => 'Doe',
    'companyname' => 'Acme Corp',
    'phonenumber' => '+1-555-123-4567',
    'password' => 'securePassword123', // Will be hashed automatically
    'currency' => 1,
    'country' => 'US'
]);
```

### getClient()

Retrieves a client by ID.

```php
/**
 * Get client by ID
 * 
 * @param int $clientId Client ID
 * @return array|null Client data
 */
function getClient(int $clientId): ?array
{
    $result = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();
    
    return $result ? (array) $result : null;
}
```

**Example:**
```php
$client = getClient(123);

if ($client) {
    echo "Welcome, {$client['firstname']} {$client['lastname']}";
    echo "Email: {$client['email']}";
    echo "Balance: $" . number_format($client['credit'], 2);
}
```

### getClientByEmail()

Retrieves a client by email address.

```php
/**
 * Get client by email
 * 
 * @param string $email Client email
 * @return array|null Client data
 */
function getClientByEmail(string $email): ?array
{
    $result = Capsule::table('tblclients')
        ->where('email', $email)
        ->first();
    
    return $result ? (array) $result : null;
}
```

**Example:**
```php
$client = getClientByEmail('john.doe@example.com');
```

### updateClient()

Updates an existing client.

```php
/**
 * Update a client
 * 
 * @param int $clientId Client ID
 * @param array $data Updated data
 * @return bool Success status
 */
function updateClient(int $clientId, array $data): bool
{
    $data['updated_at'] = date('Y-m-d H:i:s');
    
    return Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update($data) > 0;
}
```

**Example:**
```php
updateClient(123, [
    'firstname' => 'Jonathan',
    'phonenumber' => '+1-555-987-6543',
    'address1' => '456 New Street',
    'city' => 'New York',
    'state' => 'NY',
    'postcode' => '10001'
]);
```

### deleteClient()

Deletes a client account.

```php
/**
 * Delete a client
 * 
 * @param int $clientId Client ID
 * @param bool $deleteData Delete associated data
 * @return bool Success status
 */
function deleteClient(int $clientId, bool $deleteData = false): bool
{
    if ($deleteData) {
        // Delete orders, invoices, services, domains
        deleteClientOrders($clientId);
        deleteClientInvoices($clientId);
        deleteClientServices($clientId);
        deleteClientDomains($clientId);
    } else {
        // Archive client
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update(['status' => 'Inactive', 'email' => 'deleted_' . time() . '@archived.local']);
    }
    
    return true;
}
```

## Client Contacts

### getClientContacts()

Retrieves all contacts for a client.

```php
/**
 * Get client contacts
 * 
 * @param int $clientId Client ID
 * @return array Contacts
 */
function getClientContacts(int $clientId): array
{
    return Capsule::table('tblcontacts')
        ->where('userid', $clientId)
        ->get()
        ->toArray();
}
```

**Example:**
```php
$contacts = getClientContacts(123);

foreach ($contacts as $contact) {
    echo "{$contact->firstname} {$contact->lastname} - {$contact->email}\n";
}
```

### createClientContact()

Creates a new contact for a client.

```php
/**
 * Create client contact
 * 
 * @param int $clientId Client ID
 * @param array $data Contact data
 * @return int Contact ID
 */
function createClientContact(int $clientId, array $data): int
{
    return Capsule::table('tblcontacts')->insertGetId([
        'userid' => $clientId,
        'firstname' => $data['firstname'] ?? '',
        'lastname' => $data['lastname'] ?? '',
        'email' => $data['email'],
        'companyname' => $data['companyname'] ?? '',
        'phonenumber' => $data['phonenumber'] ?? '',
        'subaccount' => $data['subaccount'] ?? 0,
        'password' => $data['password'] ?? '',
        'permissions' => implode(',', $data['permissions'] ?? []),
    ]);
}
```

**Example:**
```php
$contactId = createClientContact(123, [
    'firstname' => 'Jane',
    'lastname' => 'Doe',
    'email' => 'jane.doe@example.com',
    'companyname' => 'Acme Corp',
    'subaccount' => 1,
    'permissions' => ['products', 'invoices', 'tickets']
]);
```

## Client Services

### getClientServices()

Gets all services for a client.

```php
/**
 * Get client services/hosting accounts
 * 
 * @param int $clientId Client ID
 * @param array $filters Additional filters
 * @return array Services
 */
function getClientServices(int $clientId, array $filters = []): array
{
    $query = Capsule::table('tblhosting')
        ->select('tblhosting.*', 'tblproducts.name as product_name')
        ->leftJoin('tblproducts', 'tblproducts.id', '=', 'tblhosting.packageid')
        ->where('tblhosting.userid', $clientId);
    
    if (!empty($filters['status'])) {
        $query->where('tblhosting.domainstatus', $filters['status']);
    }
    
    if (!empty($filters['serverId'])) {
        $query->where('tblhosting.server', $filters['serverId']);
    }
    
    return $query->get()->toArray();
}
```

**Example:**
```php
$services = getClientServices(123, ['status' => 'Active']);

foreach ($services as $service) {
    echo "{$service->product_name} - {$service->domain}\n";
}
```

### getClientDomains()

Gets all domains for a client.

```php
/**
 * Get client domains
 * 
 * @param int $clientId Client ID
 * @return array Domains
 */
function getClientDomains(int $clientId): array
{
    return Capsule::table('tbldomains')
        ->where('userid', $clientId)
        ->get()
        ->toArray();
}
```

## Client Invoices

### getClientInvoices()

Gets all invoices for a client.

```php
/**
 * Get client invoices
 * 
 * @param int $clientId Client ID
 * @param string|null $status Filter by status
 * @return array Invoices
 */
function getClientInvoices(int $clientId, ?string $status = null): array
{
    $query = Capsule::table('tblinvoices')
        ->where('userid', $clientId)
        ->orderBy('id', 'desc');
    
    if ($status) {
        $query->where('status', $status);
    }
    
    return $query->get()->toArray();
}
```

**Example:**
```php
$outstanding = getClientInvoices(123, 'Unpaid');

$totalDue = array_sum(array_column($outstanding, 'total'));
echo "Total outstanding: $" . number_format($totalDue, 2);
```

## Client Search

### searchClients()

Searches for clients by various criteria.

```php
/**
 * Search clients
 * 
 * @param string $term Search term
 * @param array $fields Fields to search
 * @param int $limit Result limit
 * @return array Matching clients
 */
function searchClients(string $term, array $fields = [], int $limit = 50): array
{
    $defaultFields = ['firstname', 'lastname', 'email', 'companyname', 'phonenumber'];
    $fields = $fields ?: $defaultFields;
    
    $query = Capsule::table('tblclients');
    
    // Build OR conditions for each field
    foreach ($fields as $field) {
        $query->orWhere($field, 'like', '%' . $term . '%');
    }
    
    return $query->limit($limit)->get()->toArray();
}
```

**Example:**
```php
// Search by email
$clients = searchClients('john@example.com', ['email']);

// Search by name or company
$clients = searchClients('Acme', ['firstname', 'lastname', 'companyname']);
```

## Client Authentication

### verifyClientPassword()

Verifies client password.

```php
/**
 * Verify client password
 * 
 * @param int $clientId Client ID
 * @param string $password Password to verify
 * @return bool True if valid
 */
function verifyClientPassword(int $clientId, string $password): bool
{
    $client = getClient($clientId);
    
    if (!$client) {
        return false;
    }
    
    return password_verify($password, $client['password']);
}
```

**Example:**
```php
if (verifyClientPassword(123, 'userPassword123')) {
    echo "Authentication successful";
}
```

### setClientPassword()

Sets a new password for a client.

```php
/**
 * Set client password
 * 
 * @param int $clientId Client ID
 * @param string $password New password
 * @return bool Success status
 */
function setClientPassword(int $clientId, string $password): bool
{
    $hashed = password_hash($password, PASSWORD_DEFAULT);
    
    return Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update(['password' => $hashed]) > 0;
}
```

## Client Balance Management

### getClientCreditBalance()

Gets client's credit balance.

```php
/**
 * Get client credit balance
 * 
 * @param int $clientId Client ID
 * @return float Credit balance
 */
function getClientCreditBalance(int $clientId): float
{
    $client = getClient($clientId);
    return (float) ($client['credit'] ?? 0);
}
```

### addClientCredit()

Adds credit to a client's account.

```php
/**
 * Add credit to client
 * 
 * @param int $clientId Client ID
 * @param float $amount Amount to add
 * @param string $description Description
 * @param string $adminId Admin performing action
 * @return bool Success status
 */
function addClientCredit(
    int $clientId,
    float $amount,
    string $description,
    string $adminId = ''
): bool {
    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->increment('credit', $amount);
    
    addTransaction([
        'clientid' => $clientId,
        'description' => $description,
        'amountin' => $amount
    ]);
    
    logAdminActivity("Credit of \${$amount} added for client {$clientId}", 'billing', 'credit_add', $adminId);
    
    return true;
}
```

### removeClientCredit()

Removes credit from a client.

```php
/**
 * Remove credit from client
 * 
 * @param int $clientId Client ID
 * @param float $amount Amount to remove
 * @param string $description Description
 * @param string $adminId Admin performing action
 * @return bool Success status
 */
function removeClientCredit(
    int $clientId,
    float $amount,
    string $description,
    string $adminId = ''
): bool {
    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->decrement('credit', $amount);
    
    addTransaction([
        'clientid' => $clientId,
        'description' => $description,
        'amountout' => $amount
    ]);
    
    logAdminActivity("Credit of \${$amount} removed from client {$clientId}", 'billing', 'credit_remove', $adminId);
    
    return true;
}
```

## Client Statistics

### getClientStatistics()

Gets comprehensive statistics for a client.

```php
/**
 * Get client statistics
 * 
 * @param int $clientId Client ID
 * @return array Statistics
 */
function getClientStatistics(int $clientId): array
{
    $client = getClient($clientId);
    
    if (!$client) {
        return [];
    }
    
    // Count services
    $activeServices = Capsule::table('tblhosting')
        ->where('userid', $clientId)
        ->where('domainstatus', 'Active')
        ->count();
    
    // Count domains
    $activeDomains = Capsule::table('tbldomains')
        ->where('userid', $clientId)
        ->where('status', 'Active')
        ->count();
    
    // Sum unpaid invoices
    $unpaidInvoices = Capsule::table('tblinvoices')
        ->where('userid', $clientId)
        ->where('status', 'Unpaid')
        ->sum('total');
    
    // Count open tickets
    $openTickets = Capsule::table('tbltickets')
        ->where('userid', $clientId)
        ->whereIn('status', ['Open', 'Answered'])
        ->count();
    
    // Total spent
    $totalSpent = Capsule::table('tblinvoices')
        ->where('userid', $clientId)
        ->where('status', 'Paid')
        ->sum('total');
    
    return [
        'client_id' => $clientId,
        'client_name' => "{$client['firstname']} {$client['lastname']}",
        'active_services' => $activeServices,
        'active_domains' => $activeDomains,
        'unpaid_balance' => $unpaidInvoices,
        'open_tickets' => $openTickets,
        'total_spent' => $totalSpent,
        'credit_balance' => $client['credit'] ?? 0,
        'member_since' => $client['created_at']
    ];
}
```

**Example:**
```php
$stats = getClientStatistics(123);

echo "Active Services: {$stats['active_services']}\n";
echo "Total Spent: $" . number_format($stats['total_spent'], 2) . "\n";
echo "Unpaid Balance: $" . number_format($stats['unpaid_balance'], 2);
```

## Client Groups

### getClientGroups()

Retrieves all client groups.

```php
/**
 * Get all client groups
 * 
 * @return array Client groups
 */
function getClientGroups(): array
{
    return Capsule::table('tblclientgroups')->get()->toArray();
}
```

### assignClientGroup()

Assigns a client to a group.

```php
/**
 * Assign client to group
 * 
 * @param int $clientId Client ID
 * @param int $groupId Group ID
 * @return bool Success status
 */
function assignClientGroup(int $clientId, int $groupId): bool
{
    return Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update(['groupid' => $groupId]) > 0;
}
```

## Client Currency

### getClientCurrency()

Gets a client's currency.

```php
/**
 * Get client currency
 * 
 * @param int $clientId Client ID
 * @return int Currency ID
 */
function getClientCurrency(int $clientId): int
{
    $client = getClient($clientId);
    return (int) ($client['currency'] ?? 0);
}
```

### setClientCurrency()

Sets a client's currency.

```php
/**
 * Set client currency
 * 
 * @param int $clientId Client ID
 * @param int $currencyId Currency ID
 * @return bool Success status
 */
function setClientCurrency(int $clientId, int $currencyId): bool
{
    return Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update(['currency' => $currencyId]) > 0;
}
```

## Best Practices

1. **Always hash passwords** - Never store plain text passwords
2. **Use transactions** - Wrap related operations in database transactions
3. **Validate email uniqueness** - Check for duplicate emails before creating
4. **Log changes** - Track all client modifications for audit
5. **Sanitize input** - Use WHMCS sanitization functions
6. **Handle cascades** - Properly handle related data on deletion

## Related Functions

- [whmcs-functions-transactions.md](whmcs-functions-transactions.md) - Financial operations
- [whmcs-functions-services.md](whmcs-functions-services.md) - Service management
- [whmcs-schema-clients.md](whmcs-schema-clients.md) - Client database schema