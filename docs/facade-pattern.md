# Facade Pattern in WHMCS

The Facade Pattern provides a simplified interface to a complex subsystem. In WHMCS, this pattern is ideal for creating easy-to-use APIs over complex module internals, hiding complexity while exposing necessary functionality.

## Overview

A facade acts as a unified entry point to a collection of subsystems, providing:
- Simplified API for common operations
- Hide implementation details from consumers
- Coordinate complex workflows through simple methods
- Reduce dependencies between client code and subsystem

## Core Structure

### Simple Facade Implementation

```php
<?php
// includes/Facades/ServiceFacade.php

namespace CustomModule\Facades;

class ServiceFacade
{
    protected static ServiceContainer $container;

    public static function setContainer(ServiceContainer $container): void
    {
        self::$container = $container;
    }

    public static function __callStatic(string $method, array $args)
    {
        return self::$container->make($method, $args);
    }
}
```

### Service Container Integration

```php
<?php
// includes/ServiceContainer.php

namespace CustomModule;

class ServiceContainer
{
    protected array $bindings = [];
    protected array $instances = [];

    public function bind(string $abstract, callable $factory): void
    {
        $this->bindings[$abstract] = $factory;
    }

    public function singleton(string $abstract, $instance): void
    {
        $this->instances[$abstract] = $instance;
    }

    public function make(string $abstract, array $args = []): mixed
    {
        if (isset($this->instances[$abstract])) {
            return $this->instances[$abstract];
        }

        if (isset($this->bindings[$abstract])) {
            return call_user_func($this->bindings[$abstract], $this, ...$args);
        }

        throw new \Exception("Service '{$abstract}' not bound");
    }
}
```

## Real-World WHMCS Examples

### Client Management Facade

```php
<?php
// includes/Facades/ClientManager.php

namespace CustomModule\Facades;

use WHMCS\Database\Capsule;
use CustomModule\Services\EmailService;
use CustomModule\Services\InvoiceService;
use CustomModule\Services\PermissionService;

class ClientManager
{
    protected EmailService $emailService;
    protected InvoiceService $invoiceService;
    protected PermissionService $permissionService;

    public function __construct(
        EmailService $emailService,
        InvoiceService $invoiceService,
        PermissionService $permissionService
    ) {
        $this->emailService = $emailService;
        $this->invoiceService = $invoiceService;
        $this->permissionService = $permissionService;
    }

    /**
     * Create a new client with all required setup
     */
    public function createClient(array $data): array
    {
        // Validate input
        $validated = $this->validateClientData($data);

        // Create client record
        $clientId = Capsule::table('tblclients')->insertGetId([
            'firstname' => $validated['firstname'],
            'lastname' => $validated['lastname'],
            'email' => $validated['email'],
            'companyname' => $validated['company'] ?? '',
            'phonenumber' => $validated['phone'] ?? '',
            'address1' => $validated['address'] ?? '',
            'city' => $validated['city'] ?? '',
            'state' => $validated['state'] ?? '',
            'country' => $validated['country'] ?? 'US',
            'postcode' => $validated['postcode'] ?? '',
            'password' => $this->hashPassword($validated['password'] ?? ''),
            'created_at' => date('Y-m-d H:i:s'),
            'status' => 'Active'
        ]);

        // Create client group if specified
        if (!empty($validated['group_id'])) {
            Capsule::table('tblclientgroups')->insert([
                'client_id' => $clientId,
                'group_id' => $validated['group_id']
            ]);
        }

        // Set custom fields
        $this->setCustomFields($clientId, $validated['custom_fields'] ?? []);

        // Send welcome email
        $this->emailService->sendWelcomeEmail($clientId);

        // Create welcome invoice if amount set
        if (!empty($validated['setup_amount'])) {
            $this->invoiceService->createSetupInvoice($clientId, $validated['setup_amount']);
        }

        return [
            'success' => true,
            'client_id' => $clientId,
            'email' => $validated['email']
        ];
    }

    /**
     * Suspend a client and all related services
     */
    public function suspendClient(int $clientId, string $reason): array
    {
        // Verify client exists
        $client = Capsule::table('tblclients')->find($clientId);
        if (!$client) {
            return ['success' => false, 'error' => 'Client not found'];
        }

        // Suspend all active services
        Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->where('domainstatus', 'Active')
            ->update(['domainstatus' => 'Suspended']);

        // Suspend addons
        Capsule::table('tblhostingaddons')
            ->whereIn('hostingid', function($query) use ($clientId) {
                $query->select('id')
                    ->from('tblhosting')
                    ->where('userid', $clientId);
            })
            ->where('status', 'Active')
            ->update(['status' => 'Suspended']);

        // Update client status
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update(['status' => 'Suspended']);

        // Log the suspension
        Capsule::table('tblactivitylog')->insert([
            'date' => date('Y-m-d H:i:s'),
            'description' => "Client suspended: {$reason}",
            'userid' => $_SESSION['adminid'] ?? 0,
            'clientid' => $clientId
        ]);

        // Send suspension notification
        $this->emailService->sendSuspensionNotice($clientId, $reason);

        return ['success' => true, 'services_suspended' => true];
    }

    /**
     * Terminate client and cleanup all data
     */
    public function terminateClient(int $clientId, bool $deleteData = false): array
    {
        $client = Capsule::table('tblclients')->find($clientId);
        if (!$client) {
            return ['success' => false, 'error' => 'Client not found'];
        }

        // Terminate all services
        $services = Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->whereIn('domainstatus', ['Active', 'Suspended'])
            ->get();

        foreach ($services as $service) {
            $this->terminateService($service->id);
        }

        // Cancel pending invoices
        Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->where('status', 'Unpaid')
            ->update(['status' => 'Cancelled']);

        // Delete client data if requested
        if ($deleteData) {
            $this->deleteClientData($clientId);
        }

        // Mark client as terminated
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update(['status' => 'Closed']);

        return [
            'success' => true,
            'client_id' => $clientId,
            'services_terminated' => count($services)
        ];
    }

    /**
     * Get client summary with all related data
     */
    public function getClientSummary(int $clientId): array
    {
        $client = Capsule::table('tblclients')->find($clientId);
        if (!$client) {
            return ['success' => false, 'error' => 'Client not found'];
        }

        $services = Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->get();

        $invoices = Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->orderBy('date', 'desc')
            ->limit(10)
            ->get();

        $transactions = Capsule::table('tblaccounts')
            ->where('userid', $clientId)
            ->orderBy('date', 'desc')
            ->limit(10)
            ->get();

        $totalBilled = Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->where('status', 'Paid')
            ->sum('total');

        $openTickets = Capsule::table('tbltickets')
            ->where('userid', $clientId)
            ->whereIn('status', ['Open', 'Answered'])
            ->count();

        return [
            'success' => true,
            'client' => [
                'id' => $client->id,
                'name' => $client->firstname . ' ' . $client->lastname,
                'email' => $client->email,
                'status' => $client->status,
                'created' => $client->created_at
            ],
            'services' => [
                'active' => count(array_filter($services, fn($s) => $s->domainstatus === 'Active')),
                'total' => count($services)
            ],
            'financials' => [
                'total_billed' => $totalBilled,
                'open_invoices' => count(array_filter($invoices, fn($i) => $i->status === 'Unpaid')),
                'recent_transactions' => count($transactions)
            ],
            'support' => [
                'open_tickets' => $openTickets
            ]
        ];
    }

    protected function validateClientData(array $data): array
    {
        // Validation logic...
        return $data;
    }

    protected function hashPassword(string $password): string
    {
        return password_hash($password, PASSWORD_DEFAULT);
    }

    protected function setCustomFields(int $clientId, array $fields): void
    {
        foreach ($fields as $fieldId => $value) {
            Capsule::table('tblcustomfieldsvalues')->updateOrInsert(
                ['relid' => $clientId, 'fieldid' => $fieldId],
                ['value' => $value]
            );
        }
    }

    protected function terminateService(int $serviceId): void
    {
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update(['domainstatus' => 'Terminated', 'termination_date' => date('Y-m-d')]);
    }

    protected function deleteClientData(int $clientId): void
    {
        // Delete custom field values
        Capsule::table('tblcustomfieldsvalues')
            ->where('relid', $clientId)
            ->delete();

        // Delete activity logs
        Capsule::table('tblactivitylog')
            ->where('clientid', $clientId)
            ->delete();
    }
}
```

### Domain Management Facade

```php
<?php
// includes/Facades/DomainManager.php

namespace CustomModule\Facades;

use WHMCS\Database\Capsule;
use CustomModule\Adapters\DomainRegistrarInterface;
use CustomModule\Services\DnsService;
use CustomModule\Services\EmailForwardingService;

class DomainManager
{
    protected array $registrars = [];

    public function registerRegistrar(string $tld, string $adapterClass, array $config): void
    {
        $this->registrars[$tld] = [
            'class' => $adapterClass,
            'config' => $config
        ];
    }

    /**
     * Register a new domain through appropriate registrar
     */
    public function registerDomain(string $domain, array $ registrantData): array
    {
        $tld = $this->extractTld($domain);
        $registrar = $this->getRegistrarFor($tld);

        if (!$registrar) {
            return ['success' => false, 'error' => "No registrar for .{$tld} domains"];
        }

        // Check availability
        $availability = $registrar->checkAvailability($domain);
        if (!$availability['available']) {
            return ['success' => false, 'error' => 'Domain not available'];
        }

        // Register domain
        $result = $registrar->registerDomain(new RegisterDomainRequest(
            domain: $domain,
            registrant: $registrantData,
            years: $registrantData['years'] ?? 1
        ));

        if ($result->isSuccess()) {
            $this->saveDomainRecord($domain, $result);
        }

        return [
            'success' => $result->isSuccess(),
            'domain_id' => $result->getDomainId(),
            'expiration' => $result->getExpirationDate()
        ];
    }

    /**
     * Transfer domain with authorization
     */
    public function transferDomain(string $domain, string $authCode, int $clientId): array
    {
        $tld = $this->extractTld($domain);
        $registrar = $this->getRegistrarFor($tld);

        $result = $registrar->transferDomain(new TransferDomainRequest(
            domain: $domain,
            authCode: $authCode
        ));

        if ($result->isSuccess()) {
            $this->saveTransferRecord($domain, $clientId, $result);
            $this->sendTransferNotification($clientId, $domain);
        }

        return [
            'success' => $result->isSuccess(),
            'transfer_id' => $result->getTransferId(),
            'status' => $result->getStatus()
        ];
    }

    /**
     * Configure DNS settings for domain
     */
    public function configureDns(string $domain, array $records): array
    {
        $dnsService = new DnsService();
        $dnsService->setRecords($domain, $records);

        // Update nameservers if specified
        if (!empty($records['nameservers'])) {
            $this->setNameservers($domain, $records['nameservers']);
        }

        return [
            'success' => true,
            'records_updated' => count($records)
        ];
    }

    /**
     * Enable email forwarding for domain
     */
    public function enableEmailForwarding(string $domain, array $forwardingRules): array
    {
        $emailService = new EmailForwardingService();
        return $emailService->setupForwarding($domain, $forwardingRules);
    }

    protected function extractTld(string $domain): string
    {
        $parts = explode('.', $domain);
        return count($parts) > 1 ? array_pop($parts) : '';
    }

    protected function getRegistrarFor(string $tld): ?DomainRegistrarInterface
    {
        if (!isset($this->registrars[$tld])) {
            return null;
        }

        $config = $this->registrars[$tld];
        return new $config['class'](...array_values($config['config']));
    }

    protected function saveDomainRecord(string $domain, $result): void
    {
        Capsule::table('tbldomains')->insert([
            'domain' => $domain,
            'userid' => $result->getClientId() ?? 0,
            'registrationdate' => date('Y-m-d'),
            'expirydate' => $result->getExpirationDate(),
            'registrar' => 'custom',
            'status' => 'Active'
        ]);
    }

    protected function saveTransferRecord(string $domain, int $clientId, $result): void
    {
        Capsule::table('mod_domain_transfers')->insert([
            'domain' => $domain,
            'client_id' => $clientId,
            'transfer_id' => $result->getTransferId(),
            'status' => $result->getStatus(),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    protected function setNameservers(string $domain, array $nameservers): void
    {
        Capsule::table('tbldomains')
            ->where('domain', $domain)
            ->update([
                'ns1' => $nameservers[0] ?? '',
                'ns2' => $nameservers[1] ?? '',
                'ns3' => $nameservers[2] ?? '',
                'ns4' => $nameservers[3] ?? ''
            ]);
    }

    protected function sendTransferNotification(int $clientId, string $domain): void
    {
        // Send notification email
    }
}
```

### Usage Example

```php
<?php
// hooks.php

use CustomModule\Facades\ClientManager;
use CustomModule\Facades\DomainManager;

// Hook into client creation
add_hook('ClientAreaPage', 1, function($vars) {
    // Use the facade - simple API for complex operations
    $summary = ClientManager::getClientSummary($vars['userid']);

    return [
        'clientSummary' => $summary
    ];
});

// Hook into domain registration
add_hook('DomainTransferCompleted', 1, function($vars) {
    $manager = new DomainManager();
    $manager->registerRegistrar('com', CloudflareRegistrarAdapter::class, [
        'apiKey' => 'xxx'
    ]);

    return $manager->configureDns($vars['domain'], [
        'A' => ['@' => '192.0.2.1'],
        'CNAME' => ['www' => '@']
    ]);
});
```

## Pros

- **Simplicity**: Provides easy-to-use API for complex operations
- **Decoupling**: Clients only depend on facade, not subsystem classes
- **Organization**: Centralizes related functionality
- **Testing**: Easy to mock facade for testing

## Cons

- **Limited Control**: May not expose all functionality of subsystem
- **Performance**: Additional layer adds slight overhead
- **Abstraction Leak**: Complex operations may still leak through
- **Maintenance**: Must update facade when subsystem changes

## Best Practices

1. Don't create facades for simple systems
2. Keep facade methods focused and cohesive
3. Document which subsystem operations are available
4. Provide access to underlying services when needed
5. Use method chaining for fluent interfaces where appropriate