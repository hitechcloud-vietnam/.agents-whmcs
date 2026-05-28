# WHMCS Resource Allocation Workflow
# Version: 1.0 | Created: 2026-05-28

## Purpose

Guide resource allocation strategies including server resources, bandwidth, storage, and Compute resource distribution across client accounts.

## Prerequisites

- WHMCS installation with provisioning modules configured
- Multiple server configurations
- Resource monitoring tools
- Admin access to WHMCS configuration

## Workflow Steps

### Step 1: Configure Server Resources

Set up server resource definitions via ConfigOptions in provisioning modules:

```php
// modules/servers/myprovider/myprovider.php

function myprovider_ConfigOptions(array $params): array
{
    $configOptions = [
        'resource_tier' => [
            'Type'        => 'dropdown',
            'FriendlyName' => 'Resource Tier',
            'Options'     => [
                'basic'    => 'Basic - 1 vCPU, 2GB RAM, 50GB SSD',
                'standard' => 'Standard - 2 vCPU, 4GB RAM, 100GB SSD',
                'premium'  => 'Premium - 4 vCPU, 8GB RAM, 200GB SSD',
                'business' => 'Business - 8 vCPU, 16GB RAM, 500GB SSD',
            ],
            'Default' => 'standard',
        ],
        'bandwidth_quota' => [
            'Type'        => 'dropdown',
            'FriendlyName' => 'Monthly Bandwidth',
            'Options'     => '1000GB,2000GB,5000GB,unlimited',
            'Default'     => '1000GB',
        ],
        'backup_slots' => [
            'Type'        => 'dropdown',
            'FriendlyName' => 'Backup Slots',
            'Options'     => '1,3,5,10',
            'Default'     => '3',
        ],
    ];

    // Dynamic pricing based on resources
    $pricing = [
        'bandwidth_quota' => [
            '1000GB' => ['monthly' => 5],
            '2000GB' => ['monthly' => 8],
            '5000GB' => ['monthly' => 15],
            'unlimited' => ['monthly' => 25],
        ],
    ];

    return $configOptions;
}
```

### Step 2: Create Resource Tracking Hook

Implement hooks to track and enforce resource allocation:

```php
// includes/hooks/resource_tracking.php

use WHMCS\Database\Capsule;

/**
 * Track resource usage per client
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $userId = $vars['userid'];

    Capsule::table('mod_resource_tracking')->insert([
        'service_id' => $serviceId,
        'user_id' => $userId,
        'cpu_usage' => 0,
        'ram_usage_mb' => 0,
        'disk_usage_gb' => 0,
        'bandwidth_used_gb' => 0,
        'last_updated' => date('Y-m-d H:i:s'),
    ]);
});

/**
 * Check resource limits before provisioning
 */
add_hook('PreModuleCreate', 1, function($vars) {
    $userId = $vars['userid'];
    $user = Capsule::table('tblclients')->where('id', $userId)->first();

    // Check user's total allocated resources
    $currentAllocation = Capsule::table('mod_resource_tracking')
        ->where('user_id', $userId)
        ->sum('ram_usage_mb');

    $maxAllowed = $user->max_ram_allocation ?? 32768; // 32GB default

    if ($currentAllocation > $maxAllowed) {
        return ['error' => 'Resource allocation limit exceeded for this user'];
    }
});

/**
 * Update resource usage on periodic cron
 */
add_hook('DailyCronJob', 1, function($vars) {
    $trackedServices = Capsule::table('mod_resource_tracking')->get();

    foreach ($trackedServices as $tracking) {
        $service = Capsule::table('tblhosting')
            ->where('id', $tracking->service_id)
            ->first();

        if (!$service || $service->domainstatus !== 'Active') {
            continue;
        }

        // Fetch current usage from provider API
        $usage = fetchServerResourceUsage($service->server);

        Capsule::table('mod_resource_tracking')
            ->where('id', $tracking->id)
            ->update([
                'cpu_usage' => $usage['cpu_percent'],
                'ram_usage_mb' => $usage['ram_mb'],
                'disk_usage_gb' => $usage['disk_gb'],
                'bandwidth_used_gb' => $usage['bandwidth_gb'],
                'last_updated' => date('Y-m-d H:i:s'),
            ]);
    }
});
```

### Step 3: Implement Resource Quota Enforcement

Create resource quota enforcement for overages:

```php
// includes/modules/resource_quotas.php

class ResourceQuotaManager
{
    /**
     * Check if client has exceeded resource quota
     */
    public function checkQuota(int $userId, string $resourceType): array
    {
        $user = Capsule::table('tblclients')->where('id', $userId)->first();
        $limits = $this->getUserLimits($user);

        $currentUsage = Capsule::table('mod_resource_tracking')
            ->where('user_id', $userId)
            ->sum($this->mapResourceColumn($resourceType));

        return [
            'within_limit' => $currentUsage < $limits[$resourceType],
            'current'     => $currentUsage,
            'limit'       => $limits[$resourceType],
            'percent_used' => round(($currentUsage / $limits[$resourceType]) * 100, 2),
        ];
    }

    /**
     * Handle resource overage billing
     */
    public function processOverages(int $invoiceId): void
    {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

        if ($invoice->status !== 'Unpaid') {
            return;
        }

        $userId = $invoice->userid;
        $overages = $this->calculateOverages($userId);

        foreach ($overages as $resource => $amount) {
            if ($amount > 0) {
                $this->addOverageItem($invoiceId, $resource, $amount);
            }
        }
    }

    private function calculateOverages(int $userId): array
    {
        $usage = Capsule::table('mod_resource_tracking')
            ->where('user_id', $userId)
            ->first();

        $limits = $this->getUserLimits(
            Capsule::table('tblclients')->where('id', $userId)->first()
        );

        $overages = [];

        if ($usage->bandwidth_used_gb > $limits['bandwidth']) {
            $overages['bandwidth'] = ($usage->bandwidth_used_gb - $limits['bandwidth'])
                * $limits['bandwidth_overage_rate'];
        }

        return $overages;
    }

    private function mapResourceColumn(string $resource): string
    {
        $mapping = [
            'cpu'       => 'cpu_usage',
            'ram'       => 'ram_usage_mb',
disk'      => 'disk_usage_gb',
            'bandwidth' => 'bandwidth_used_gb',
        ];

        return $mapping[$resource] ?? 'bandwidth_used_gb';
    }

    private function getUserLimits($user): array
    {
        return [
            'cpu'       => $user->max_cpu_cores ?? 8,
            'ram'       => ($user->max_ram_mb ?? 32768),
            'disk'      => ($user->max_disk_gb ?? 500),
            'bandwidth' => ($user->monthly_bandwidth_gb ?? 1000),
            'bandwidth_overage_rate' => 0.05, // per GB
        ];
    }

    private function addOverageItem(int $invoiceId, string $resource, float $amount): void
    {
        Capsule::table('tblinvoiceitems')->insert([
            'invoiceid'   => $invoiceId,
            'userid'      => Capsule::table('tblinvoices')->where('id', $invoiceId)->first()->userid,
            'description' => 'Resource overage: ' . ucfirst($resource),
            'amount'      => $amount,
            'taxed'       => 1,
        ]);
    }
}
```

### Step 4: Configure Load-Based Allocation

Implement auto-scaling based on resource demand:

```php
// includes/hooks/auto_scaling.php

add_hook('DailyCronJob', 2, function($vars) {
    $scaler = new ResourceScaler();

    // Get all active services with high resource usage
    $highUsageServices = Capsule::table('mod_resource_tracking')
        ->where('cpu_usage', '>', 80)
        ->orWhere('ram_usage_mb', '>', 8192)
        ->get();

    foreach ($highUsageServices as $tracking) {
        $currentTier = Capsule::table('tblhosting')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->where('tblhosting.id', $tracking->service_id)
            ->first();

        // Check if upgrade is beneficial
        if ($scaler->shouldAutoScaleUp($tracking)) {
            $nextTier = $scaler->getNextResourceTier($currentTier->configoption1);
            $scaler->scheduleUpgrade($tracking->service_id, $nextTier);
        }
    }
});

class ResourceScaler
{
    private array $tierResources = [
        'basic'    => ['cpu' => 1, 'ram' => 2048, 'disk' => 50],
        'standard' => ['cpu' => 2, 'ram' => 4096, 'disk' => 100],
        'premium'  => ['cpu' => 4, 'ram' => 8192, 'disk' => 200],
        'business' => ['cpu' => 8, 'ram' => 16384, 'disk' => 500],
    ];

    public function shouldAutoScaleUp($tracking): bool
    {
        // Scale up if consistently high usage
        $recentHistory = Capsule::table('mod_resource_history')
            ->where('service_id', $tracking->service_id)
            ->where('recorded_at', '>', date('Y-m-d H:i:s', strtotime('-72 hours')))
            ->avg('cpu_usage');

        return ($recentHistory > 75) || ($tracking->ram_usage_mb > 7680);
    }

    public function getNextResourceTier(string $currentTier): string
    {
        $tiers = array_keys($this->tierResources);
        $currentIndex = array_search($currentTier, $tiers);

        if ($currentIndex === false || $currentIndex >= count($tiers) - 1) {
            return $currentTier;
        }

        return $tiers[$currentIndex + 1];
    }

    public function scheduleUpgrade(int $serviceId, string $newTier): void
    {
        Capsule::table('mod_resource_scaling_queue')->insert([
            'service_id'     => $serviceId,
            'action'         => 'upgrade',
            'target_tier'    => $newTier,
            'scheduled_for'  => date('Y-m-d H:i:s', strtotime('+1 hour')),
            'status'         => 'pending',
        ]);
    }
}
```

### Step 5: Set Up Resource Monitoring Dashboard

Create admin area dashboard for resource monitoring:

```php
// modules/addons/resource_monitor/resource_monitor.php

function resource_monitor_config(): array
{
    return [
        'name'        => 'Resource Monitor',
        'description' => 'Monitor and manage server resource allocation',
        'version'     => '1.0',
    ];
}

function resource_monitor_activate(): array
{
    Capsule::schema()->create('mod_resource_tracking', function($t) {
        $t->increments('id');
        $t->integer('service_id');
        $t->integer('user_id');
        $t->decimal('cpu_usage', 5, 2)->default(0);
        $t->integer('ram_usage_mb')->default(0);
        $t->integer('disk_usage_gb')->default(0);
        $t->integer('bandwidth_used_gb')->default(0);
        $t->timestamp('last_updated')->useCurrent();
    });

    Capsule::schema()->create('mod_resource_scaling_queue', function($t) {
        $t->increments('id');
        $t->integer('service_id');
        $t->enum('action', ['upgrade', 'downgrade']);
        $t->string('target_tier');
        $t->timestamp('scheduled_for');
        $t->enum('status', ['pending', 'in_progress', 'completed', 'failed']);
    });

    return ['status' => 'success'];
}

function resource_monitor_output(array $vars): void
{
    $action = $_REQUEST['action'] ?? 'dashboard';

    switch ($action) {
        case 'dashboard':
            $renderer = new ResourceDashboardRenderer();
            echo $renderer->renderDashboard();
            break;
        case 'usage_details':
            $renderer = new ResourceDashboardRenderer();
            echo $renderer->renderUsageDetails($_REQUEST['user_id']);
            break;
        case 'allocate':
            check_token('WHMCS.admin.default');
            $manager = new ResourceAllocationManager();
            $manager->allocateResources($_POST);
            redir('module=resource_monitor&action=dashboard');
            break;
    }
}
```

---

## Best Practices

1. **Set realistic default limits** - Start conservative, adjust based on actual usage patterns
2. **Implement gradual scaling** - Avoid sudden resource changes that could disrupt services
3. **Monitor for abuse** - Set up alerts for unusual resource consumption patterns
4. **Document resource tiers** - Clearly communicate what each tier includes
5. **Automate cleanup** - Schedule regular cleanup of unused resources
6. **Implement fair usage policies** - Ensure equitable distribution across all clients
7. **Plan for peak demand** - Keep reserve capacity for burst traffic
8. **Use resource pools** - Group resources for efficient allocation

---

## Verification Checklist

- [ ] Server provisioning module configured with resource options
- [ ] Resource tracking hook captures initial allocation
- [ ] Monitoring collected data visible in admin dashboard
- [ ] Resource quota enforcement notifies clients approaching limits
- [ ] Overage billing adds correct line items to invoices
- [ ] Auto-scaling triggers when configured thresholds exceeded
- [ ] Manual resource adjustment capability available in admin
- [ ] Client can view their resource usage in portal
- [ ] Historical resource data retained for reporting
- [ ] Alerts configured for critical resource thresholds
