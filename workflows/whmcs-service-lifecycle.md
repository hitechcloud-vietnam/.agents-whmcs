# WHMCS Service Lifecycle Management Workflow

## Purpose

Comprehensive guide to managing the complete lifecycle of hosting services in WHMCS, from provisioning to termination, including automated state transitions and billing integration.

## Prerequisites

- WHMCS installation with provisioning modules
- Product/service configurations
- Cron job setup
- Module development access

## Workflow Steps

### Step 1: Lifecycle State Machine

Define service lifecycle states:

```php
// includes/service_lifecycle/states.php

/**
 * Service lifecycle states
 */
class ServiceLifecycleStates
{
    const PENDING = 'Pending';
    const ACTIVE = 'Active';
    const SUSPENDED = 'Suspended';
    const TERMINATED = 'Terminated';
    const CANCELLED = 'Cancelled';
    const FRAUD = 'Fraud';
    const PENDING_TRANSFER = 'Pending Transfer';
    const TRANSFERRED = 'Transferred';
    
    /**
     * Valid state transitions
     */
    public static function getValidTransitions(): array
    {
        return [
            self::PENDING => [self::ACTIVE, self::CANCELLED, self::FRAUD],
            self::ACTIVE => [self::SUSPENDED, self::TERMINATED, self::CANCELLED],
            self::SUSPENDED => [self::ACTIVE, self::TERMINATED, self::CANCELLED],
            self::TERMINATED => [], // Final state
            self::CANCELLED => [], // Final state
            self::FRAUD => [self::TERMINATED],
            self::PENDING_TRANSFER => [self::ACTIVE, self::PENDING],
        ];
    }
    
    /**
     * Check if transition is valid
     */
    public static function canTransition(string $from, string $to): bool
    {
        $validTransitions = self::getValidTransitions();
        return in_array($to, $validTransitions[$from] ?? []);
    }
    
    /**
     * Get lifecycle actions based on state
     */
    public static function getActions(string $state): array
    {
        $actions = [
            self::PENDING => [
                ['action' => 'approve', 'label' => 'Approve & Provision', 'icon' => 'check'],
                ['action' => 'cancel', 'label' => 'Cancel Order', 'icon' => 'times'],
                ['action' => 'fraud', 'label' => 'Mark as Fraud', 'icon' => 'exclamation'],
            ],
            self::ACTIVE => [
                ['action' => 'suspend', 'label' => 'Suspend Service', 'icon' => 'pause'],
                ['action' => 'terminate', 'label' => 'Terminate', 'icon' => 'trash'],
                ['action' => 'upgrade', 'label' => 'Upgrade/Downgrade', 'icon' => 'arrow-up'],
            ],
            self::SUSPENDED => [
                ['action' => 'unsuspend', 'label' => 'Reactivate', 'icon' => 'play'],
                ['action' => 'terminate', 'label' => 'Terminate', 'icon' => 'trash'],
            ],
            self::FRAUD => [
                ['action' => 'terminate', 'label' => 'Terminate', 'icon' => 'trash'],
                ['action' => 'restore', 'label' => 'Restore (False Positive)', 'icon' => 'undo'],
            ],
        ];
        
        return $actions[$state] ?? [];
    }
}
```

### Step 2: Automated State Transitions

Implement automated lifecycle management:

```php
// modules/addons/service_lifecycle/lifecycle.php

/**
 * Daily lifecycle processing cron
 */
add_hook('DailyCronJob', 1, function($vars) {
    $processor = new ServiceLifecycleProcessor();
    
    // Process pending services
    $processor->processPendingServices();
    
    // Check due for suspension
    $processor->processOverdueServices();
    
    // Check due for termination
    $processor->processTerminationQueue();
    
    // Process scheduled cancellations
    $processor->processScheduledCancellations();
    
    // Process pending transfers
    $processor->processDomainTransfers();
});

class ServiceLifecycleProcessor
{
    /**
     * Process pending services - provision after payment
     */
    public function processPendingServices(): void
    {
        $pendingServices = Capsule::table('tblhosting')
            ->join('tblorders', 'tblorders.id', '=', 'tblhosting.orderid')
            ->where('tblhosting.domainstatus', 'Pending')
            ->where('tblorders.status', 'Active')
            ->get();
        
        foreach ($pendingServices as $service) {
            $this->provisionService($service);
        }
    }
    
    /**
     * Provision a pending service
     */
    private function provisionService($service): void
    {
        $params = $this->buildModuleParams($service);
        $module = $params['configoption1']; // Module name
        
        // Call module CreateAccount function
        $result = \WHMCS\Module\Module::factory($module)->createAccount($params);
        
        if ($result === 'success') {
            $this->transitionServiceStatus($service->id, ServiceLifecycleStates::ACTIVE);
            
            logActivity("Service #{$service->id} provisioned successfully");
            
            // Send notification
            send_email('service_activated', ['id' => $service->id]);
        } else {
            logActivity("Service #{$service->id} provisioning failed: {$result}");
            
            // Log failure for retry
            $this->logProvisioningFailure($service->id, $result);
        }
    }
    
    /**
     * Process overdue services - suspend
     */
    public function processOverdueServices(): void
    {
        $overdueDays = get_config('SuspendAfterDays') ?? 14;
        $cutoffDate = date('Y-m-d', strtotime("-{$overdueDays} days"));
        
        $overdueServices = Capsule::table('tblhosting')
            ->join('tblinvoices', 'tblinvoices.id', '=', 'tblhosting.nextinvoicedate')
            ->where('tblhosting.domainstatus', 'Active')
            ->where('tblhosting.nextduedate', '<', $cutoffDate)
            ->whereNotNull('tblhosting.suspendreason')
            ->get();
        
        foreach ($overdueServices as $service) {
            $this->suspendService($service->id, 'Overdue payment');
        }
    }
    
    /**
     * Suspend a service
     */
    public function suspendService(int $serviceId, string $reason): bool
    {
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
        
        if ($service->domainstatus !== ServiceLifecycleStates::ACTIVE) {
            return false;
        }
        
        $params = $this->buildModuleParams($service);
        $module = \WHMCS\Module\Module::factory($service->package);
        
        // Call module SuspendAccount function
        $result = $module->suspend($params);
        
        if ($result === 'success') {
            $this->transitionServiceStatus($serviceId, ServiceLifecycleStates::SUSPENDED);
            $this->recordStateTransition($serviceId, 'Active', 'Suspended', $reason);
            
            logActivity("Service #{$serviceId} suspended: {$reason}");
            
            // Update suspension statistics
            Capsule::table('mod_service_stats')->updateOrInsert(
                ['service_id' => $serviceId],
                ['suspension_count' => Capsule::raw('suspension_count + 1')]
            );
            
            return true;
        }
        
        logActivity("Service #{$serviceId} suspension failed: {$result}");
        return false;
    }
    
    /**
     * Reactivate a suspended service
     */
    public function unsuspendService(int $serviceId): bool
    {
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
        
        if ($service->domainstatus !== ServiceLifecycleStates::SUSPENDED) {
            return false;
        }
        
        // Check for outstanding balance
        $outstandingBalance = calculateClientBalance($service->userid);
        
        if ($outstandingBalance > 0) {
            $this->sendPaymentRequiredNotification($serviceId);
            return false;
        }
        
        $params = $this->buildModuleParams($service);
        $module = \WHMCS\Module\Module::factory($service->package);
        
        $result = $module->unsuspend($params);
        
        if ($result === 'success') {
            $this->transitionServiceStatus($serviceId, ServiceLifecycleStates::ACTIVE);
            $this->recordStateTransition($serviceId, 'Suspended', 'Active', 'Payment received');
            
            logActivity("Service #{$serviceId} reactivated");
            
            return true;
        }
        
        return false;
    }
    
    /**
     * Terminate a service
     */
    public function terminateService(int $serviceId, string $reason): bool
    {
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
        
        if (!ServiceLifecycleStates::canTransition($service->domainstatus, 'Terminated')) {
            return false;
        }
        
        $params = $this->buildModuleParams($service);
        $module = \WHMCS\Module\Module::factory($service->package);
        
        // Call module TerminateAccount function
        $result = $module->terminate($params);
        
        if ($result === 'success') {
            $this->transitionServiceStatus($serviceId, ServiceLifecycleStates::TERMINATED);
            $this->recordStateTransition($serviceId, $service->domainstatus, 'Terminated', $reason);
            
            // Record termination
            Capsule::table('mod_service_terminations')->insert([
                'service_id' => $serviceId,
                'reason' => $reason,
                'terminated_at' => date('Y-m-d H:i:s'),
            ]);
            
            logActivity("Service #{$serviceId} terminated: {$reason}");
            
            return true;
        }
        
        return false;
    }
    
    /**
     * Transition service to new status
     */
    private function transitionServiceStatus(int $serviceId, string $newStatus): void
    {
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update([
                'domainstatus' => $newStatus,
                'updated_at' => date('Y-m-d H:i:s'),
            ]);
    }
    
    /**
     * Record state transition for audit
     */
    private function recordStateTransition(int $serviceId, string $from, string $to, string $reason): void
    {
        Capsule::table('mod_service_state_history')->insert([
            'service_id' => $serviceId,
            'from_status' => $from,
            'to_status' => $to,
            'reason' => $reason,
            'transitioned_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    /**
     * Build module parameters for service
     */
    private function buildModuleParams($service): array
    {
        $server = Capsule::table('tblservers')
            ->where('id', $service->server)
            ->first();
        
        return [
            'serviceid' => $service->id,
            'userid' => $service->userid,
            'domain' => $service->domain,
            'username' => $service->username,
            'password' => decrypt($service->password),
            'configoption1' => $service->configoption1,
            'configoption2' => $service->configoption2,
            'configoption3' => $service->configoption3,
            'configoption4' => $service->configoption4,
            'server' => [
                'id' => $server->id,
                'hostname' => $server->hostname,
                'username' => $server->username,
                'password' => decrypt($server->password),
                'ip' => $server->ipaddress,
            ],
        ];
    }
}
```

### Step 3: Service Upgrade/Downgrade

Implement product changes:

```php
// modules/addons/service_upgrades/upgrade.php

/**
 * Handle service upgrade/downgrade
 */
class ServiceUpgradeManager
{
    /**
     * Process upgrade or downgrade request
     */
    public function processUpgradeRequest(int $serviceId, int $newProductId, string $billingCycle): array
    {
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
        $newProduct = Capsule::table('tblproducts')->where('id', $newProductId)->first();
        $oldProduct = Capsule::table('tblproducts')->where('id', $service->packageid)->first();
        
        // Calculate price difference
        $pricing = $this->calculateUpgradePrice($service, $newProduct, $billingCycle);
        
        // Create prorated invoice
        $invoiceId = $this->createUpgradeInvoice($service, $pricing);
        
        // Schedule upgrade for after payment
        if ($invoiceId) {
            Capsule::table('mod_upgrade_queue')->insert([
                'service_id' => $serviceId,
                'old_product_id' => $service->packageid,
                'new_product_id' => $newProductId,
                'invoice_id' => $invoiceId,
                'status' => 'pending_payment',
                'scheduled_at' => null,
                'created_at' => date('Y-m-d H:i:s'),
            ]);
        }
        
        return $pricing;
    }
    
    /**
     * Calculate upgrade/downgrade pricing
     */
    private function calculateUpgradePrice($service, $newProduct, string $billingCycle): array
    {
        $oldPrice = getBillingCycleAmount($service->packageid, $service->billingcycle);
        $newPrice = getBillingCycleAmount($newProduct->id, $billingCycle);
        
        // Calculate unused value of current cycle
        $daysUsed = $this->calculateDaysUsed($service);
        $daysRemaining = 30 - $daysUsed;
        $unusedCredit = ($oldPrice / 30) * $daysRemaining;
        
        // Calculate new charges
        $proratedCharge = $this->calculateProratedCharge($newPrice, $daysRemaining);
        
        // Determine if upgrade or downgrade
        $isUpgrade = $newPrice > $oldPrice;
        
        $total = $isUpgrade ? max(0, $proratedCharge - $unusedCredit) : $unusedCredit - $proratedCharge;
        
        return [
            'old_price' => $oldPrice,
            'new_price' => $newPrice,
            'unused_credit' => $unusedCredit,
            'prorated_charge' => $proratedCharge,
            'difference' => $total,
            'is_upgrade' => $isUpgrade,
            'description' => $isUpgrade ? 'Upgrade' : 'Downgrade',
        ];
    }
    
    /**
     * Execute the upgrade after payment
     */
    public function executeUpgrade(int $serviceId): bool
    {
        $queueItem = Capsule::table('mod_upgrade_queue')
            ->where('service_id', $serviceId)
            ->where('status', 'pending_payment')
            ->first();
        
        if (!$queueItem) {
            return false;
        }
        
        // Check invoice is paid
        $invoice = Capsule::table('tblinvoices')
            ->where('id', $queueItem->invoice_id)
            ->first();
        
        if ($invoice->status !== 'Paid') {
            return false;
        }
        
        // Execute module change
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
        $params = $this->buildModuleParams($service);
        
        $module = \WHMCS\Module\Module::factory($service->package);
        $result = $module->changePackage($params);
        
        if ($result === 'success') {
            // Update service record
            Capsule::table('tblhosting')
                ->where('id', $serviceId)
                ->update([
                    'packageid' => $queueItem->new_product_id,
                    'billingcycle' => $invoice->billingcycle,
                    'amount' => $invoice->total,
                ]);
            
            // Update queue status
            Capsule::table('mod_upgrade_queue')
                ->where('id', $queueItem->id)
                ->update(['status' => 'completed']);
            
            // Record history
            $this->recordUpgradeHistory($queueItem);
            
            logActivity("Service #{$serviceId} upgraded to product #{$queueItem->new_product_id}");
            
            return true;
        }
        
        return false;
    }
}
```

### Step 4: Lifecycle Reporting

Generate lifecycle analytics:

```php
// modules/addons/lifecycle_analytics/analytics.php

/**
 * Generate lifecycle statistics
 */
function getLifecycleStatistics(array $dateRange): array
{
    $stats = [];
    
    // New activations
    $stats['new_activations'] = Capsule::table('tblhosting')
        ->whereBetween('regdate', [$dateRange['start'], $dateRange['end']])
        ->where('domainstatus', 'Active')
        ->count();
    
    // Suspended services
    $stats['suspensions'] = Capsule::table('mod_service_state_history')
        ->whereBetween('transitioned_at', [$dateRange['start'], $dateRange['end']])
        ->where('to_status', 'Suspended')
        ->count();
    
    // Reactivations
    $stats['reactivations'] = Capsule::table('mod_service_state_history')
        ->whereBetween('transitioned_at', [$dateRange['start'], $dateRange['end']])
        ->where('to_status', 'Active')
        ->where('from_status', 'Suspended')
        ->count();
    
    // Terminations
    $stats['terminations'] = Capsule::table('mod_service_state_history')
        ->whereBetween('transitioned_at', [$dateRange['start'], $dateRange['end']])
        ->where('to_status', 'Terminated')
        ->count();
    
    // Average lifecycle
    $stats['avg_lifecycle_days'] = Capsule::table('mod_service_terminations')
        ->selectRaw('AVG(DATEDIFF(terminated_at, (SELECT regdate FROM tblhosting WHERE id = service_id)))')
        ->value() ?? 0;
    
    // Survival rate
    $totalActive = Capsule::table('tblhosting')
        ->where('domainstatus', 'Active')
        ->count();
    
    $terminated = Capsule::table('tblhosting')
        ->where('domainstatus', 'Terminated')
        ->count();
    
    $stats['survival_rate'] = $totalActive / ($totalActive + $terminated) * 100;
    
    return $stats;
}
```

## Best Practices

1. **Automated transitions** - Reduce manual intervention
2. **Clear state transitions** - Define allowed paths
3. **Audit everything** - Track all state changes
4. **Graceful suspensions** - Warning before action
5. **Easy reactivation** - Minimize friction for payments
6. **Clean termination** - Proper data handling
7. **Monitor patterns** - Track recurring suspensions
8. **Regular reviews** - Analyze lifecycle data

## Common Pitfalls to Avoid

1. **Invalid transitions** - Allowing impossible state changes
2. **No audit trail** - Missing change history
3. **Abrupt terminations** - No warning period
4. **Data loss** - Not handling data on termination
5. **Billing sync issues** - Status vs billing mismatch
6. **Module failures** - Not handling provisioning errors
7. **Race conditions** - Concurrent state changes
8. **Missing notifications** - No user communication
