# WHMCS Rollback Automation Workflow

## Overview
This workflow implements automated rollback mechanisms for WHMCS when operations fail.

## Prerequisites
- WHMCS with database access
- PHP 7.4+ for modern syntax
- Comprehensive logging setup

## Step-by-Step Process

### Step 1: Create Rollback Manager
```php
<?php
// /includes/rollback/RollbackManager.php

namespace WHMCS\Rollback;

class RollbackManager
{
    private $actions = [];
    private $executed = false;

    /**
     * Begin rollback-capable operation
     */
    public function begin(): self
    {
        $this->actions = [];
        $this->executed = false;
        return $this;
    }

    /**
     * Register an action with its rollback
     */
    public function register(string $name, callable $action, callable $rollback, $context = null): self
    {
        $this->actions[] = [
            'name' => $name,
            'action' => $action,
            'rollback' => $rollback,
            'context' => $context,
            'executed' => false,
            'result' => null
        ];

        return $this;
    }

    /**
     * Execute all actions
     */
    public function execute(): array
    {
        $results = [];
        $this->executed = true;

        foreach ($this->actions as $index => &$action) {
            try {
                $action['result'] = $action['action']($action['context']);
                $action['executed'] = true;
                $results[$action['name']] = ['success' => true, 'result' => $action['result']];
            } catch (Exception $e) {
                $results[$action['name']] = ['success' => false, 'error' => $e->getMessage()];

                // Stop on first failure
                break;
            }
        }

        return $results;
    }

    /**
     * Rollback all executed actions
     */
    public function rollback(): array
    {
        if (!$this->executed) {
            return ['message' => 'Nothing to rollback'];
        }

        $results = [];

        // Rollback in reverse order
        for ($i = count($this->actions) - 1; $i >= 0; $i--) {
            $action = $this->actions[$i];

            if ($action['executed']) {
                try {
                    $rollbackResult = $action['rollback']($action['context'], $action['result']);
                    $results[$action['name']] = ['success' => true, 'rollback_result' => $rollbackResult];

                    logActivity("Rollback successful for: {$action['name']}");
                } catch (Exception $e) {
                    $results[$action['name']] = ['success' => false, 'error' => $e->getMessage()];

                    logActivity("Rollback failed for: {$action['name']} - " . $e->getMessage());
                }
            }
        }

        return $results;
    }

    /**
     * Execute with automatic rollback on failure
     */
    public function executeOrRollback(callable $callback): array
    {
        try {
            return $this->execute();
        } catch (Exception $e) {
            logActivity("Operation failed, initiating rollback: " . $e->getMessage());
            return $this->rollback();
        }
    }
}
```

### Step 2: Create Rollback-Aware Service Provisioning
```php
<?php
// Service provisioning with rollback

class RollbackServiceProvisioning
{
    private $rollbackManager;

    public function __construct()
    {
        $this->rollbackManager = new RollbackManager();
    }

    public function provision(array $orderData): array
    {
        $this->rollbackManager->begin();

        try {
            // Step 1: Create or get client
            $clientId = $this->createOrGetClient($orderData);

            $this->rollbackManager->register(
                'create_client',
                function($data) { return $this->doCreateClient($data); },
                function($data, $result) { $this->undoCreateClient($result); },
                $orderData
            );

            // Step 2: Create service record
            $serviceId = $this->createServiceRecord($clientId, $orderData);

            $this->rollbackManager->register(
                'create_service',
                function($data) { return $this->doCreateService($data); },
                function($data, $result) { $this->undoCreateService($result); },
                ['client_id' => $clientId, 'data' => $orderData]
            );

            // Step 3: Create invoice
            $invoiceId = $this->createInvoice($clientId, $serviceId, $orderData);

            $this->rollbackManager->register(
                'create_invoice',
                function($data) { return $this->doCreateInvoice($data); },
                function($data, $result) { $this->undoCreateInvoice($result); },
                ['service_id' => $serviceId, 'data' => $orderData]
            );

            // Step 4: Provision on server
            $provisionResult = $this->provisionOnServer($serviceId);

            $this->rollbackManager->register(
                'provision_server',
                function($data) { return $this->doProvisionServer($data); },
                function($data, $result) { $this->undoProvisionServer($result); },
                ['service_id' => $serviceId, 'result' => $provisionResult]
            );

            // Step 5: Update service status
            $this->activateService($serviceId);

            $this->rollbackManager->register(
                'activate_service',
                function($data) { return $this->doActivateService($data); },
                function($data, $result) { $this->undoActivateService($data); },
                ['service_id' => $serviceId, 'previous_status' => 'Pending']
            );

            // All successful
            $this->rollbackManager->execute();

            return [
                'success' => true,
                'client_id' => $clientId,
                'service_id' => $serviceId,
                'invoice_id' => $invoiceId
            ];
        } catch (Exception $e) {
            logActivity("Service provisioning failed: " . $e->getMessage());

            // Rollback all changes
            $rollbackResults = $this->rollbackManager->rollback();

            return [
                'success' => false,
                'error' => $e->getMessage(),
                'rollback_results' => $rollbackResults
            ];
        }
    }

    private function doCreateClient($data) { /* create client */ }
    private function undoCreateClient($result) { Capsule::table('tblclients')->where('id', $result)->delete(); }
    private function doCreateService($data) { /* create service */ }
    private function undoCreateService($result) { Capsule::table('tblhosting')->where('id', $result)->delete(); }
    private function doCreateInvoice($data) { /* create invoice */ }
    private function undoCreateInvoice($result) { Capsule::table('tblinvoices')->where('id', $result)->delete(); }
    private function doProvisionServer($data) { /* provision */ }
    private function undoProvisionServer($result) { ServerAPI::terminateAccount($result['server_id']); }
    private function doActivateService($data) { /* activate */ }
    private function undoActivateService($data) { Capsule::table('tblhosting')->where('id', $data['service_id'])->update(['domainstatus' => 'Pending']); }
}
```

### Step 3: Create State Snapshots
```php
<?php
// State snapshot for rollback

class StateSnapshot
{
    /**
     * Create snapshot before operation
     */
    public static function create(string $entityType, int $entityId): int
    {
        $data = self::captureState($entityType, $entityId);

        return Capsule::table('mod_state_snapshots')->insertGetId([
            'entity_type' => $entityType,
            'entity_id' => $entityId,
            'state_data' => json_encode($data),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    /**
     * Capture current state
     */
    private static function captureState(string $entityType, int $entityId): array
    {
        return match ($entityType) {
            'service' => self::captureServiceState($entityId),
            'client' => self::captureClientState($entityId),
            'invoice' => self::captureInvoiceState($entityId),
            default => []
        };
    }

    private static function captureServiceState(int $serviceId): array
    {
        return Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();
    }

    private static function captureClientState(int $clientId): array
    {
        $client = Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();

        $services = Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->get();

        $invoices = Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->get();

        return [
            'client' => $client,
            'services' => $services,
            'invoices' => $invoices
        ];
    }

    private static function captureInvoiceState(int $invoiceId): array
    {
        return Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->first();
    }

    /**
     * Restore from snapshot
     */
    public static function restore(int $snapshotId): bool
    {
        $snapshot = Capsule::table('mod_state_snapshots')
            ->where('id', $snapshotId)
            ->first();

        if (!$snapshot) {
            return false;
        }

        $stateData = json_decode($snapshot->state_data, true);

        return match ($snapshot->entity_type) {
            'service' => self::restoreServiceState($snapshot->entity_id, $stateData),
            'client' => self::restoreClientState($snapshot->entity_id, $stateData),
            'invoice' => self::restoreInvoiceState($snapshot->entity_id, $stateData),
            default => false
        };
    }

    private static function restoreServiceState(int $serviceId, array $state): bool
    {
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update([
                'domainstatus' => $state->domainstatus,
                'username' => $state->username,
                'password' => $state->password,
                'notes' => $state->notes
            ]);

        return true;
    }
}
```

### Step 4: Implement Automatic Rollback Hooks
```php
<?php
// /includes/hooks/rollback_hooks.php

add_hook('PreModuleCreate', 1, function($vars) {
    // Create snapshot before provisioning
    $snapshotId = StateSnapshot::create('service', $vars['serviceid']);

    // Store in session or request context
    $_SESSION['rollback_snapshot'] = $snapshotId;
});

add_hook('AfterModuleCreate', 1, function($vars) {
    // Clear snapshot on success
    unset($_SESSION['rollback_snapshot']);
});

add_hook('AfterModuleCreateFailed', 1, function($vars) {
    // Restore snapshot on failure
    if (isset($_SESSION['rollback_snapshot'])) {
        StateSnapshot::restore($_SESSION['rollback_snapshot']);
        unset($_SESSION['rollback_snapshot']);

        logActivity("Service state restored after provisioning failure");
    }
});
```

### Step 5: Create Rollback for Invoice Operations
```php
<?php
// Invoice operations with rollback

class RollbackInvoiceOperations
{
    public static function processPaymentWithRollback(int $invoiceId, array $paymentData): array
    {
        $rollbackManager = new RollbackManager();
        $rollbackManager->begin();

        try {
            // Get current invoice state
            $snapshotId = StateSnapshot::create('invoice', $invoiceId);
            $rollbackManager->register(
                'create_snapshot',
                fn() => $snapshotId,
                fn($id) => true, // Snapshot cleanup is automatic
                $snapshotId
            );

            // 1. Record payment
            $transactionId = self::recordPayment($invoiceId, $paymentData);
            $rollbackManager->register(
                'record_payment',
                fn($data) => self::doRecordPayment($data),
                fn($data, $result) => self::undoRecordPayment($result),
                ['invoice_id' => $invoiceId, 'payment' => $paymentData]
            );

            // 2. Update invoice status
            self::updateInvoiceStatus($invoiceId, 'Paid');
            $rollbackManager->register(
                'update_status',
                fn($id) => self::doUpdateStatus($id),
                fn($id) => self::undoStatus($id, $snapshotId),
                $invoiceId
            );

            // 3. Activate services
            $services = self::activateServices($invoiceId);
            $rollbackManager->register(
                'activate_services',
                fn($id) => self::doActivateServices($id),
                fn($id, $results) => self::undoActivateServices($results),
                $invoiceId
            );

            $result = $rollbackManager->execute();

            return [
                'success' => true,
                'transaction_id' => $transactionId,
                'services_activated' => $services
            ];
        } catch (Exception $e) {
            $rollbackResults = $rollbackManager->rollback();

            return [
                'success' => false,
                'error' => $e->getMessage(),
                'rollback_results' => $rollbackResults
            ];
        }
    }
}
```

### Step 6: Monitor Rollback Operations
```php
<?php
// Rollback monitoring

add_hook('DailyCronJob', 1, function($vars) {
    // Check for recent rollbacks
    $recentRollbacks = Capsule::table('mod_rollback_log')
        ->where('created_at', '>', date('Y-m-d H:i:s', strtotime('-24 hours')))
        ->count();

    if ($recentRollbacks > 5) {
        sendAdminEmail('High Rollback Rate Detected', [
            'rollback_count' => $recentRollbacks,
            'period' => 'last 24 hours'
        ]);
    }
});

/**
 * Log rollback operation
 */
function logRollback(string $operation, array $results, string $reason)
{
    Capsule::table('mod_rollback_log')->insert([
        'operation' => $operation,
        'results' => json_encode($results),
        'reason' => $reason,
        'created_at' => date('Y-m-d H:i:s')
    ]);
}
```

## Rollback Best Practices

1. **Always rollback on failure** - Don't leave partial data
2. **Test rollback operations** - Ensure they work correctly
3. **Keep rollback simple** - Complex rollbacks can fail too
4. **Log everything** - For debugging and auditing
5. **Create snapshots** - For complex state restoration
6. **Notify on failures** - Alert admin when rollbacks occur

## Related Workflows
- [WHMCS Transaction Management](./whmcs-transaction-management.md)
- [WHMCS Error Recovery](./whmcs-error-recovery.md)
- [WHMCS Backup Automation](./whmcs-backup-automation.md)