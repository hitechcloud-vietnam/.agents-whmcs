# WHMCS Bulk Operations Workflow

## Purpose

Procedures for performing bulk operations in WHMCS, including mass service management, batch updates, and automated bulk workflows.

## Prerequisites

- WHMCS admin access
- Database access
- Bulk operation tools/scripts
- Testing environment

## Workflow Steps

### Step 1: Create Bulk Operation Framework

Build reusable bulk operation system:

```php
<?php
// modules/custom/bulk_operations.php

namespace WHMCS\Custom;

class BulkOperationManager
{
    private $batchSize = 50;
    private $errors = [];
    private $results = [];
    
    public function __construct($batchSize = 50)
    {
        $this->batchSize = $batchSize;
    }
    
    public function processServices(array $serviceIds, callable $operation, $description = 'Service')
    {
        $total = count($serviceIds);
        $processed = 0;
        
        foreach (array_chunk($serviceIds, $this->batchSize) as $batch) {
            foreach ($batch as $serviceId) {
                try {
                    $result = $operation($serviceId);
                    $this->results[$serviceId] = [
                        'success' => true,
                        'result' => $result,
                    ];
                    $processed++;
                    
                } catch (\Exception $e) {
                    $this->errors[] = [
                        'service_id' => $serviceId,
                        'error' => $e->getMessage(),
                    ];
                    $this->results[$serviceId] = [
                        'success' => false,
                        'error' => $e->getMessage(),
                    ];
                }
                
                // Log progress
                logActivity("Bulk {$description}: Processed {$processed}/{$total}");
            }
            
            // Prevent memory issues
            Capsule::connection()->getPdo()->commit();
        }
        
        return $this->generateReport($description);
    }
    
    public function processClients(array $clientIds, callable $operation, $description = 'Client')
    {
        $total = count($clientIds);
        $processed = 0;
        
        foreach (array_chunk($clientIds, $this->batchSize) as $batch) {
            foreach ($batch as $clientId) {
                try {
                    $result = $operation($clientId);
                    $this->results["client_{$clientId}"] = [
                        'success' => true,
                        'result' => $result,
                    ];
                    $processed++;
                    
                } catch (\Exception $e) {
                    $this->errors[] = [
                        'client_id' => $clientId,
                        'error' => $e->getMessage(),
                    ];
                }
            }
        }
        
        return $this->generateReport($description);
    }
    
    private function generateReport($operationType)
    {
        $successCount = count(array_filter($this->results, fn($r) => $r['success']));
        $failureCount = count($this->results) - $successCount;
        
        $report = [
            'operation' => $operationType,
            'total_processed' => count($this->results),
            'success_count' => $successCount,
            'failure_count' => $failureCount,
            'errors' => $this->errors,
            'completed_at' => date('c'),
        ];
        
        // Store report
        Capsule::table('mod_bulk_operation_logs')->insert($report);
        
        return $report;
    }
    
    public function getErrors()
    {
        return $this->errors;
    }
}
```

### Step 2: Implement Bulk Suspension

Suspend multiple services:

```php
<?php
// modules/custom/bulk_suspend.php

require_once __DIR__ . '/init.php';

class BulkSuspendService
{
    private $manager;
    
    public function __construct()
    {
        $this->manager = new \WHMCS\Custom\BulkOperationManager(25);
    }
    
    public function suspendByCriteria(array $criteria)
    {
        $serviceIds = $this->getServicesByCriteria($criteria);
        
        if (empty($serviceIds)) {
            return ['message' => 'No services found matching criteria'];
        }
        
        return $this->manager->processServices(
            $serviceIds,
            [$this, 'suspendService'],
            'Suspension'
        );
    }
    
    public function suspendByIds(array $serviceIds)
    {
        return $this->manager->processServices(
            $serviceIds,
            [$this, 'suspendService'],
            'Suspension'
        );
    }
    
    public function suspendService($serviceId)
    {
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();
        
        if (!$service) {
            throw new \Exception("Service {$serviceId} not found");
        }
        
        if ($service->domainstatus === 'Suspended') {
            return 'Already suspended';
        }
        
        // Call suspension hook/module
        $params = [
            'serviceid' => $serviceId,
            'model' => $service,
        ];
        
        $serverType = Capsule::table('tblservers')
            ->where('id', $service->server)
            ->value('type');
        
        if ($serverType) {
            $result = Provisioning\Module::call($serverType, 'SuspendAccount', $params);
            
            if ($result !== 'success') {
                throw new \Exception("Suspension failed: " . ($result['error'] ?? 'Unknown'));
            }
        }
        
        // Update status
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update([
                'domainstatus' => 'Suspended',
                'suspendreason' => 'Bulk suspension: ' . date('Y-m-d H:i:s'),
            ]);
        
        // Log activity
        logActivity("Bulk suspension: Service {$serviceId} ({$service->domain}) suspended");
        
        return 'Suspended successfully';
    }
    
    private function getServicesByCriteria(array $criteria)
    {
        $query = Capsule::table('tblhosting')
            ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
            ->where('tblhosting.domainstatus', 'Active');
        
        if (!empty($criteria['billing_cycle'])) {
            $query->where('tblhosting.billingcycle', $criteria['billing_cycle']);
        }
        
        if (!empty($criteria['product_id'])) {
            $query->where('tblhosting.packageid', $criteria['product_id']);
        }
        
        if (!empty($criteria['overdue_days'])) {
            $overdueDate = Carbon::now()->subDays($criteria['overdue_days'])->toDateString();
            $query->where('tblhosting.nextduedate', '<', $overdueDate);
        }
        
        if (!empty($criteria['client_group'])) {
            $query->where('tblclients.groupid', $criteria['client_group']);
        }
        
        if (!empty($criteria['server_id'])) {
            $query->where('tblhosting.server', $criteria['server_id']);
        }
        
        return $query->pluck('tblhosting.id')->toArray();
    }
}

// CLI execution
if (php_sapi_name() === 'cli' && basename(__FILE__) === basename($_SERVER['SCRIPT_FILENAME'])) {
    $options = getopt('', ['ids:', 'criteria:', 'dry-run']);
    
    $bulk = new BulkSuspendService();
    
    if (isset($options['ids'])) {
        $ids = array_map('intval', explode(',', $options['ids']));
        $result = $bulk->suspendByIds($ids);
    } elseif (isset($options['criteria'])) {
        $criteria = json_decode($options['criteria'], true);
        $result = $bulk->suspendByCriteria($criteria);
    }
    
    echo json_encode($result, JSON_PRETTY_PRINT) . "\n";
}
```

### Step 3: Implement Bulk Termination

Terminate multiple services:

```php
<?php
// modules/custom/bulk_terminate.php

class BulkTerminateService
{
    private $manager;
    private $dryRun = false;
    
    public function __construct($dryRun = false)
    {
        $this->manager = new \WHMSC\Custom\BulkOperationManager(10);
        $this->dryRun = $dryRun;
    }
    
    public function terminateExpired(array $criteria = [])
    {
        $criteria['status'] = 'Expired';
        $serviceIds = $this->findTerminatableServices($criteria);
        
        if (empty($serviceIds)) {
            return ['message' => 'No expired services found'];
        }
        
        return $this->processTermination($serviceIds, 'Expired Service Termination');
    }
    
    public function terminateCancelled(array $criteria = [])
    {
        $criteria['status'] = 'Cancelled';
        $serviceIds = $this->findTerminatableServices($criteria);
        
        if (empty($serviceIds)) {
            return ['message' => 'No cancelled services found'];
        }
        
        return $this->processTermination($serviceIds, 'Cancelled Service Termination');
    }
    
    private function processTermination(array $serviceIds, $description)
    {
        if ($this->dryRun) {
            return [
                'dry_run' => true,
                'would_terminate' => count($serviceIds),
                'service_ids' => $serviceIds,
            ];
        }
        
        return $this->manager->processServices(
            $serviceIds,
            [$this, 'terminateService'],
            $description
        );
    }
    
    public function terminateService($serviceId)
    {
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();
        
        if (!$service) {
            throw new \Exception("Service {$serviceId} not found");
        }
        
        // Get server info
        $server = Capsule::table('tblservers')
            ->where('id', $service->server)
            ->first();
        
        // Terminate on server
        if ($server) {
            $params = [
                'serviceid' => $serviceId,
                'model' => $service,
            ];
            
            $result = Provisioning\Module::call($server->type, 'TerminateAccount', $params);
            
            if ($result !== 'success') {
                throw new \Exception("Server termination failed");
            }
        }
        
        // Update database
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update([
                'domainstatus' => 'Terminated',
                'termination_date' => Carbon::now()->toDateTimeString(),
            ]);
        
        // Cancel pending invoices
        Capsule::table('tblinvoiceitems')
            ->where('relid', $serviceId)
            ->where('type', 'hosting')
            ->join('tblinvoices', 'tblinvoiceitems.invoiceid', '=', 'tblinvoices.id')
            ->where('tblinvoices.status', 'Unpaid')
            ->update(['tblinvoices.status' => 'Cancelled']);
        
        logActivity("Bulk termination: Service {$serviceId} terminated");
        
        return 'Terminated successfully';
    }
    
    private function findTerminatableServices(array $criteria)
    {
        $query = Capsule::table('tblhosting')
            ->where('domainstatus', $criteria['status'] ?? 'Expired');
        
        if (!empty($criteria['days_overdue'])) {
            $date = Carbon::now()->subDays($criteria['days_overdue'])->toDateString();
            $query->where('nextduedate', '<', $date);
        }
        
        if (!empty($criteria['product_id'])) {
            $query->where('packageid', $criteria['product_id']);
        }
        
        // Only terminate if no active dependent services
        $query->whereNotExists(function($q) {
            $q->select(Capsule::raw(1))
                ->from('tblorders')
                ->whereColumn('tblorders.service', 'tblhosting.id')
                ->where('tblorders.status', '!=', 'Cancelled');
        });
        
        return $query->pluck('id')->toArray();
    }
}
```

### Step 4: Implement Bulk Updates

Update multiple records:

```php
<?php
// modules/custom/bulk_update.php

class BulkUpdateService
{
    public function updatePricing(array $serviceIds, array $updates)
    {
        $results = [
            'updated' => 0,
            'errors' => [],
        ];
        
        foreach (array_chunk($serviceIds, 50) as $batch) {
            foreach ($batch as $serviceId) {
                try {
                    $this->updateServicePricing($serviceId, $updates);
                    $results['updated']++;
                } catch (\Exception $e) {
                    $results['errors'][] = [
                        'id' => $serviceId,
                        'error' => $e->getMessage(),
                    ];
                }
            }
        }
        
        logActivity("Bulk pricing update: {$results['updated']} services updated");
        
        return $results;
    }
    
    private function updateServicePricing($serviceId, array $updates)
    {
        $updateData = [];
        
        if (isset($updates['amount'])) {
            $updateData['amount'] = $updates['amount'];
        }
        
        if (isset($updates['billing_cycle'])) {
            $updateData['billingcycle'] = $updates['billing_cycle'];
        }
        
        if (!empty($updateData)) {
            Capsule::table('tblhosting')
                ->where('id', $serviceId)
                ->update($updateData);
        }
    }
    
    public function changeProduct(array $serviceIds, $newProductId, $preservePricing = true)
    {
        $newProduct = Capsule::table('tblproducts')
            ->where('id', $newProductId)
            ->first();
        
        if (!$newProduct) {
            throw new \Exception("Product {$newProductId} not found");
        }
        
        $results = ['updated' => 0, 'errors' => []];
        
        foreach (array_chunk($serviceIds, 50) as $batch) {
            foreach ($batch as $serviceId) {
                try {
                    $this->changeServiceProduct($serviceId, $newProduct, $preservePricing);
                    $results['updated']++;
                } catch (\Exception $e) {
                    $results['errors'][] = [
                        'id' => $serviceId,
                        'error' => $e->getMessage(),
                    ];
                }
            }
        }
        
        return $results;
    }
    
    private function changeServiceProduct($serviceId, $newProduct, $preservePricing)
    {
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();
        
        $updateData = [
            'packageid' => $newProduct->id,
        ];
        
        if (!$preservePricing) {
            $updateData['amount'] = $newProduct->monthly;
        }
        
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update($updateData);
        
        logActivity("Product changed for service {$serviceId}: {$newProduct->name}");
    }
    
    public function changeServer(array $serviceIds, $newServerId)
    {
        $server = Capsule::table('tblservers')
            ->where('id', $newServerId)
            ->first();
        
        if (!$server) {
            throw new \Exception("Server {$newServerId} not found");
        }
        
        $results = ['updated' => 0, 'migrated' => 0, 'errors' => []];
        
        foreach (array_chunk($serviceIds, 25) as $batch) {
            foreach ($batch as $serviceId) {
                try {
                    // Update server reference
                    Capsule::table('tblhosting')
                        ->where('id', $serviceId)
                        ->update([
                            'server' => $newServerId,
                            'servertype' => $server->type,
                        ]);
                    
                    // Optionally migrate data
                    if ($_GET['migrate_data'] ?? false) {
                        $this->migrateServiceData($serviceId, $server);
                        $results['migrated']++;
                    }
                    
                    $results['updated']++;
                    
                } catch (\Exception $e) {
                    $results['errors'][] = [
                        'id' => $serviceId,
                        'error' => $e->getMessage(),
                    ];
                }
            }
        }
        
        return $results;
    }
    
    private function migrateServiceData($serviceId, $server)
    {
        // Create account on new server
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();
        
        $params = [
            'serviceid' => $serviceId,
            'model' => $service,
        ];
        
        Provisioning\Module::call($server->type, 'CreateAccount', $params);
    }
}
```

### Step 5: Create Bulk Operations Admin Interface

Admin panel for bulk operations:

```php
<?php
// admin/bulk_operations.php

require_once __DIR__ . '/../init.php';

if (!checkPermission('Bulk Operations', true)) {
    exit('Access Denied');
}

$action = $_GET['action'] ?? 'menu';

switch ($action) {
    case 'menu':
        echo $twig->render('admin/bulk_operations/menu.html');
        break;
        
    case 'suspend':
        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            $result = processBulkSuspend($_POST);
            echo $twig->render('admin/bulk_operations/results.html', [
                'result' => $result,
            ]);
        } else {
            echo $twig->render('admin/bulk_operations/suspend_form.html', [
                'products' => getProducts(),
                'servers' => getServers(),
                'clientGroups' => getClientGroups(),
            ]);
        }
        break;
        
    case 'terminate':
        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            $result = processBulkTerminate($_POST);
            echo $twig->render('admin/bulk_operations/results.html', [
                'result' => $result,
            ]);
        } else {
            echo $twig->render('admin/bulk_operations/terminate_form.html', [
                'products' => getProducts(),
            ]);
        }
        break;
        
    case 'update_pricing':
        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            $result = processBulkPricingUpdate($_POST);
            echo $twig->render('admin/bulk_operations/results.html', [
                'result' => $result,
            ]);
        } else {
            echo $twig->render('admin/bulk_operations/pricing_form.html', [
                'products' => getProducts(),
            ]);
        }
        break;
        
    case 'history':
        echo $twig->render('admin/bulk_operations/history.html', [
            'logs' => getOperationHistory(100),
        ]);
        break;
}
```

```php
<?php
// admin/helpers/bulk_processing.php

function processBulkSuspend($postData)
{
    $criteria = [];
    
    if (!empty($postData['product_id'])) {
        $criteria['product_id'] = (int) $postData['product_id'];
    }
    
    if (!empty($postData['overdue_days'])) {
        $criteria['overdue_days'] = (int) $postData['overdue_days'];
    }
    
    if (!empty($postData['server_id'])) {
        $criteria['server_id'] = (int) $postData['server_id'];
    }
    
    if (!empty($postData['client_group'])) {
        $criteria['client_group'] = (int) $postData['client_group'];
    }
    
    $bulk = new \WHMCS\Custom\BulkSuspendService();
    
    if (isset($postData['preview']) || isset($postData['dry_run'])) {
        $serviceIds = findServicesMatchingCriteria($criteria);
        return [
            'preview' => true,
            'count' => count($serviceIds),
            'services' => array_slice($serviceIds, 0, 50),
        ];
    }
    
    return $bulk->suspendByCriteria($criteria);
}

function getOperationHistory($limit = 50)
{
    return Capsule::table('mod_bulk_operation_logs')
        ->orderBy('completed_at', 'desc')
        ->limit($limit)
        ->get();
}
```

### Step 6: Create CLI Tools

Command-line bulk operations:

```bash
#!/bin/bash
# /usr/local/bin/whmcs-bulk-operation

set -euo pipefail

COMMAND="${1:-}"
shift || true

case "$COMMAND" in
    suspend)
        php /var/www/whmcs/modules/custom/bulk_suspend.php "$@"
        ;;
    terminate)
        php /var/www/whmcs/modules/custom/bulk_terminate.php "$@"
        ;;
    update-pricing)
        php /var/www/whmcs/modules/custom/bulk_update.php pricing "$@"
        ;;
    change-server)
        php /var/www/whmcs/modules/custom/bulk_update.php server "$@"
        ;;
    *)
        echo "Usage: $0 {suspend|terminate|update-pricing|change-server} [options]"
        echo ""
        echo "Commands:"
        echo "  suspend          Suspend services"
        echo "  terminate        Terminate services"
        echo "  update-pricing   Update pricing"
        echo "  change-server   Change server"
        exit 1
        ;;
esac
```

## Verification Checklist

- [ ] Bulk operation framework created
- [ ] Suspension workflow tested
- [ ] Termination workflow tested
- [ ] Update operations verified
- [ ] Admin interface functional
- [ ] CLI tools working
- [ ] Error handling implemented
- [ ] Logging enabled
- [ ] Rollback capability available
- [ ] Performance acceptable for large datasets

## Related Skills and Documentation

- [WHMCS Provisioning Automation](whmcs-provisioning-automation-workflow.md)
- [WHMCS Module Testing](whmcs-module-testing-workflow.md)
- WHMCS CLI Documentation: https://developers.whmcs.com/advanced/cli/

## Notes

- Always test bulk operations on small samples first
- Enable dry-run mode to preview changes
- Schedule large operations during off-peak hours
- Keep detailed logs of all bulk operations
- Consider database transaction sizes
- Implement proper error handling and recovery
- Notify affected clients when required
