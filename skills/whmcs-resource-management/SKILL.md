# WHMCS Resource Management Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for resource allocation and management in WHMCS including server resources, service quotas, and capacity planning.

## When to Use

- Managing server capacity allocations
- Implementing resource quotas for clients
- Tracking resource usage across services
- Building resource pool management systems

## Resource Allocation Patterns

### 1. Server Resource Tracking

```php
<?php
namespace WHMCS\Resource;

class ServerResourceManager {
    private array $metrics = [];

    public function trackResourceUsage(int $serverId, array $usage): void {
        $server = Capsule::table('tblservers')->find($serverId);
        if (!$server) return;

        $existing = Capsule::table('mod_resource_tracking')
            ->where('server_id', $serverId)
            ->where('date', date('Y-m-d'))
            ->first();

        $data = [
            'server_id' => $serverId,
            'date' => date('Y-m-d'),
            'cpu_usage' => $usage['cpu'] ?? 0,
            'memory_usage' => $usage['memory'] ?? 0,
            'disk_usage' => $usage['disk'] ?? 0,
            'bandwidth_used' => $usage['bandwidth'] ?? 0,
            'updated_at' => date('Y-m-d H:i:s'),
        ];

        if ($existing) {
            Capsule::table('mod_resource_tracking')
                ->where('server_id', $serverId)
                ->where('date', date('Y-m-d'))
                ->update($data);
        } else {
            Capsule::table('mod_resource_tracking')->insert($data);
        }
    }

    public function getServerCapacity(int $serverId): array {
        $stats = Capsule::table('tblservers')
            ->selectRaw(' COUNT(*) as total_services')
            ->selectRaw(' SUM(tblhosting.dedicatedip != "") as dedicated_ips')
            ->join('tblhosting', 'tblhosting.server', '=', 'tblservers.id')
            ->where('tblservers.id', $serverId)
            ->where('tblhosting.domainstatus', 'Active')
            ->first();

        return [
            'total_services' => $stats->total_services ?? 0,
            'dedicated_ips' => $stats->dedicated_ips ?? 0,
            'available_slots' => $this->calculateAvailableSlots($serverId),
        ];
    }

    private function calculateAvailableSlots(int $serverId): int {
        $server = Capsule::table('tblservers')->find($serverId);
        $maxClients = $server->maxclients ?? 100;
        $currentClients = Capsule::table('tblhosting')
            ->where('server', $serverId)
            ->where('domainstatus', 'Active')
            ->count();

        return max(0, $maxClients - $currentClients);
    }
}
```

### 2. Client Resource Quotas

```php
<?php
namespace WHMCS\Resource;

class ClientQuotaManager {
    public function checkQuota(int $userId, string $resourceType, float $amount): bool {
        $quota = $this->getClientQuota($userId, $resourceType);
        $used = $this->getUsedResources($userId, $resourceType);

        return ($used + $amount) <= $quota['limit'];
    }

    public function getClientQuota(int $userId, string $resourceType): array {
        $client = Capsule::table('tblclients')->find($userId);
        $tierQuota = $this->getTierQuotas($client['groupid'] ?? 0);

        return [
            'type' => $resourceType,
            'limit' => $tierQuota[$resourceType] ?? PHP_INT_MAX,
            'used' => $this->getUsedResources($userId, $resourceType),
            'tier' => $client['groupid'] ?? 0,
        ];
    }

    private function getTierQuotas(int $groupId): array {
        $tier = Capsule::table('mod_resource_tiers')->find($groupId);
        if (!$tier) {
            return [
                'services' => 10,
                'storage_gb' => 100,
                'bandwidth_gb' => 1000,
                'domains' => 50,
            ];
        }

        return [
            'services' => $tier->services_limit ?? 10,
            'storage_gb' => $tier->storage_limit ?? 100,
            'bandwidth_gb' => $tier->bandwidth_limit ?? 1000,
            'domains' => $tier->domains_limit ?? 50,
        ];
    }

    private function getUsedResources(int $userId, string $resourceType): float {
        switch ($resourceType) {
            case 'services':
                return (float) Capsule::table('tblhosting')
                    ->where('userid', $userId)
                    ->where('domainstatus', 'Active')
                    ->count();

            case 'storage_gb':
                return (float) Capsule::table('tblhosting')
                    ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
                    ->where('tblhosting.userid', $userId)
                    ->where('tbltblhosting.domainstatus', 'Active')
                    ->sum('tblproducts.diskspace');

            case 'domains':
                return (float) Capsule::table('tbldomains')
                    ->where('userid', $userId)
                    ->where('status', 'Active')
                    ->count();

            default:
                return 0;
        }
    }

    public function allocateResource(int $userId, string $resourceType, float $amount): bool {
        if (!$this->checkQuota($userId, $resourceType, $amount)) {
            return false;
        }

        Capsule::table('mod_resource_usage')->insert([
            'user_id' => $userId,
            'resource_type' => $resourceType,
            'amount' => $amount,
            'allocated_at' => date('Y-m-d H:i:s'),
        ]);

        return true;
    }
}
```

### 3. Resource Pool Management

```php
<?php
namespace WHMCS\Resource;

class ResourcePoolManager {
    private array $poolConfig;

    public function __construct() {
        $this->poolConfig = $this->loadPoolConfiguration();
    }

    public function findOptimalServer(string $productType, array $requirements): ?array {
        $servers = Capsule::table('tblservers')
            ->where('disabled', 0)
            ->where('type', $productType)
            ->get();

        $candidates = [];
        foreach ($servers as $server) {
            $capacity = $this->getServerCapacity($server->id);
            $score = $this->calculateFitnessScore($server, $requirements, $capacity);

            if ($score > 0) {
                $candidates[] = [
                    'server' => $server,
                    'capacity' => $capacity,
                    'score' => $score,
                ];
            }
        }

        if (empty($candidates)) {
            return null;
        }

        // Sort by score descending
        usort($candidates, fn($a, $b) => $b['score'] <=> $a['score']);

        return $candidates[0]['server'];
    }

    private function calculateFitnessScore($server, array $reqs, array $capacity): float {
        $score = 100;

        // Deduct points for insufficient resources
        if (isset($reqs['memory']) && $capacity['memory_available'] < $reqs['memory']) {
            $score -= 50;
        }
        if (isset($reqs['disk']) && $capacity['disk_available'] < $reqs['disk']) {
            $score -= 50;
        }
        if ($capacity['available_slots'] <= 0) {
            $score -= 100;
        }

        // Prefer servers with more headroom
        $score -= ($capacity['load_average'] ?? 0) * 5;

        return max(0, $score);
    }

    private function getServerCapacity(int $serverId): array {
        $server = Capsule::table('tblservers')->find($serverId);

        return [
            'memory_available' => ($server->maxmemory ?? 0) - ($server->usedmemory ?? 0),
            'disk_available' => ($server->maxdisk ?? 0) - ($server->useddisk ?? 0),
            'bandwidth_available' => ($server->maxbw ?? 0) - ($server->usedbw ?? 0),
            'available_slots' => $this->getAvailableSlots($serverId),
            'load_average' => $this->getServerLoad($serverId),
        ];
    }

    private function getAvailableSlots(int $serverId): int {
        $maxClients = Capsule::table('tblservers')->find($serverId)->maxclients ?? 100;
        $currentClients = Capsule::table('tblhosting')
            ->where('server', $serverId)
            ->where('domainstatus', 'Active')
            ->count();

        return max(0, $maxClients - $currentClients);
    }

    private function getServerLoad(int $serverId): float {
        $today = Capsule::table('mod_resource_tracking')
            ->where('server_id', $serverId)
            ->where('date', date('Y-m-d'))
            ->orderBy('created_at', 'desc')
            ->first();

        return $today ? ($today->cpu_usage / 100) : 0;
    }

    private function loadPoolConfiguration(): array {
        return [
            'balance_algorithm' => 'least_loaded',
            'failover_enabled' => true,
            'health_check_interval' => 300,
        ];
    }
}
```

## Capacity Planning

### 4. Forecasting Resource Needs

```php
<?php
namespace WHMCS\Resource;

class CapacityForecaster {
    public function forecastDemand(int $serverId, int $daysAhead = 30): array {
        $history = Capsule::table('mod_resource_tracking')
            ->where('server_id', $serverId)
            ->where('date', '>=', date('Y-m-d', strtotime("-60 days")))
            ->orderBy('date')
            ->get();

        if ($history->isEmpty()) {
            return ['growth_rate' => 0, 'projected_usage' => [], 'recommendation' => 'insufficient_data'];
        }

        $cpuTrend = $this->calculateTrend($history->pluck('cpu_usage')->toArray());
        $memoryTrend = $this->calculateTrend($history->pluck('memory_usage')->toArray());
        $bandwidthTrend = $this->calculateTrend($history->pluck('bandwidth_used')->toArray());

        $server = Capsule::table('tblservers')->find($serverId);

        return [
            'server_id' => $serverId,
            'cpu' => [
                'current' => $history->last()->cpu_usage ?? 0,
                'growth_rate' => $cpuTrend,
                'projected' => $this->projectUsage($history->last()->cpu_usage ?? 0, $cpuTrend, $daysAhead),
                'capacity' => 100,
            ],
            'memory' => [
                'current' => $history->last()->memory_usage ?? 0,
                'growth_rate' => $memoryTrend,
                'projected' => $this->projectUsage($history->last()->memory_usage ?? 0, $memoryTrend, $daysAhead),
                'capacity' => $server->maxmemory ?? 0,
            ],
            'bandwidth' => [
                'current' => $history->last()->bandwidth_used ?? 0,
                'growth_rate' => $bandwidthTrend,
                'projected' => $this->projectUsage($history->last()->bandwidth_used ?? 0, $bandwidthTrend, $daysAhead),
                'capacity' => $server->maxbw ?? 0,
            ],
            'recommendation' => $this->generateRecommendation($server),
        ];
    }

    private function calculateTrend(array $values): float {
        if (count($values) < 2) return 0;

        $n = count($values);
        $x = range(0, $n - 1);
        $y = $values;

        $sumX = array_sum($x);
        $sumY = array_sum($y);
        $sumXY = array_sum(array_map(fn($x, $y) => $x * $y, $x, $y));
        $sumX2 = array_sum(array_map(fn($x) => $x * $x, $x));

        $denominator = ($n * $sumX2 - $sumX * $sumX);
        if ($denominator == 0) return 0;

        $slope = ($n * $sumXY - $sumX * $sumY) / $denominator;
        return $slope;
    }

    private function projectUsage(float $current, float $trend, int $days): float {
        return $current + ($trend * $days);
    }

    private function generateRecommendation($server): string {
        $utilization = ($server->usedmemory ?? 0) / ($server->maxmemory ?? 1) * 100;

        if ($utilization > 90) {
            return 'critical_consider_new_server';
        } elseif ($utilization > 75) {
            return 'warning_plan_scaling';
        } elseif ($utilization > 50) {
            return 'normal_adequate_headroom';
        }

        return return 'underutilized_optimize_resources';
    }
}
```

## Hook Integration

```php
// Auto-scale resources based on demand
add_hook('AfterModuleCreate', 1, function($vars) {
    $manager = new ServerResourceManager();
    $capacity = $manager->getServerCapacity($vars['serverid']);

    if ($capacity['available_slots'] < 5) {
        sendAdminNotification('critical', 'Server capacity warning', [
            'server' => $vars['serverid'],
            'available_slots' => $capacity['available_slots'],
        ]);
    }
});

// Track resource usage daily
add_hook('DailyCronJob', 1, function($vars) {
    $servers = Capsule::table('tblservers')->where('disabled', 0)->get();

    foreach ($servers as $server) {
        $usage = queryServerMetrics($server);
        $manager->trackResourceUsage($server->id, $usage);
    }
});
```

## Checklist

- [ ] Resource tracking table schema
- [ ] Server capacity calculation
- [ ] Client quota enforcement
- [ ] Resource pool selection algorithm
- [ ] Capacity forecasting
- [ ] Admin notification triggers
- [ ] Usage reporting

---

**Related Skills:**
- whmcs-server-builder
- whmcs-monitoring
- whmcs-metrics-analytics
- whmcs-service-billing
