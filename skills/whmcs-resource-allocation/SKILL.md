# WHMCS Resource Allocation Skill

## Purpose
Provides patterns for implementing resource allocation in WHMCS, managing server capacity, bandwidth, storage, and computational resources across hosting services.

## Implementation Patterns

### Resource Quota System
```php
<?php
// includes/ResourceAllocation.class.php

class WHMCSResourceAllocation {
    private $db;
    private $cache;
    
    public function __construct() {
        $this->db = console::db();
        $this->cache = console::cache();
    }
    
    // Initialize resource allocation for a client
    public function initializeAllocation($clientId, $planId) {
        $plan = $this->getPlanResources($planId);
        
        $allocation = [
            'client_id' => $clientId,
            'cpu_cores' => $plan['cpu_cores'],
            'ram_mb' => $plan['ram_mb'],
            'disk_gb' => $plan['disk_gb'],
            'bandwidth_tb' => $plan['bandwidth_tb'],
            'iops_limit' => $plan['iops_limit'],
            'concurrent_connections' => $plan['connections'],
            'allocated_at' => date('Y-m-d H:i:s'),
            'current_usage' => json_encode([
                'cpu' => 0,
                'ram' => 0,
                'disk' => 0,
                'bandwidth' => 0
            ])
        ];
        
        return $this->db->insert('mod_resource_allocation', $allocation);
    }
    
    // Check resource availability before provisioning
    public function checkAvailability($serverId, $resourceType, $requiredAmount) {
        $server = $this->getServerCapacity($serverId);
        $allocated = $this->getAllocatedResources($serverId, $resourceType);
        $available = $server[$resourceType] - $allocated;
        
        return [
            'available' => $available >= $requiredAmount,
            'requested' => $requiredAmount,
            'available_amount' => $available,
            'utilization' => ($allocated / $server[$resourceType]) * 100
        ];
    }
    
    // Allocate resources to a service
    public function allocateToService($serviceId, $resourceType, $amount) {
        $this->db->where('service_id', $serviceId)
            ->update('mod_resource_allocation', [
                "{$resourceType}_allocated" => $this->db->inc($amount)
            ]);
        
        $this->invalidateCache("server_resources");
        $this->logAllocation($serviceId, $resourceType, $amount);
    }
    
    // Get real-time resource usage
    public function getResourceUsage($clientId) {
        $cacheKey = "resource_usage_{$clientId}";
        if ($cached = $this->cache->get($cacheKey)) {
            return $cached;
        }
        
        $usage = $this->db->select(
            'SELECT SUM(cpu_cores) as cpu, SUM(ram_mb) as ram, 
                    SUM(disk_gb) as disk, SUM(bandwidth_tb) as bandwidth
             FROM mod_resource_allocation WHERE client_id = ?',
            [$clientId]
        );
        
        $this->cache->set($cacheKey, $usage, 300);
        return $usage;
    }
    
    // Enforce resource limits
    public function enforceLimits($userId, $resourceType) {
        $allocation = $this->getUserAllocation($userId);
        $usage = $this->getCurrentUsage($userId, $resourceType);
        
        if ($usage > $allocation[$resourceType]) {
            $this->triggerResourceLimitAction($userId, $resourceType);
            return false;
        }
        return true;
    }
}
```

### Server Capacity Manager
```php
class ServerCapacityManager {
    public function calculateOptimalDistribution($services) {
        $servers = $this->getAvailableServers();
        $distribution = [];
        
        foreach ($servers as $server) {
            $capacity = $this->calculateServerCapacity($server);
            $distribution[$server['id']] = [
                'can_accept' => $capacity['available'] > 0,
                'recommended_load' => $capacity['recommended_load'],
                'current_load' => $capacity['current_load'],
                'suitable_services' => []
            ];
        }
        
        // Sort services by resource requirements
        usort($services, function($a, $b) {
            return $b['resource_score'] <=> $a['resource_score'];
        });
        
        // Assign services to optimal servers
        foreach ($services as $service) {
            $bestServer = $this->findBestServer($service, $distribution);
            if ($bestServer) {
                $distribution[$bestServer]['assigned_services'][] = $service;
                $this->updateServerLoad($distribution, $bestServer, $service);
            }
        }
        
        return $distribution;
    }
    
    private function calculateServerCapacity($server) {
        $metrics = $this->getServerMetrics($server['id']);
        
        return [
            'cpu_available' => $server['cpu_cores'] - $metrics['cpu_allocated'],
            'ram_available' => $server['ram_mb'] - $metrics['ram_allocated'],
            'disk_available' => $server['disk_gb'] - $metrics['disk_allocated'],
            'current_load' => ($metrics['cpu_allocated'] / $server['cpu_cores']) * 100,
            'recommended_load' => 70, // Keep 30% headroom
            'health_score' => $this->calculateHealthScore($metrics)
        ];
    }
    
    public function getResourceMetrics($serverId, $timeRange = '24h') {
        return $this->db->select(
            "SELECT 
                AVG(cpu_usage) as avg_cpu,
                MAX(cpu_usage) as max_cpu,
                AVG(memory_usage) as avg_ram,
                MAX(memory_usage) as max_ram,
                SUM(bandwidth_in) as bandwidth_in,
                SUM(bandwidth_out) as bandwidth_out,
                AVG(iops) as avg_iops
             FROM mod_server_metrics
             WHERE server_id = ? AND timestamp > DATE_SUB(NOW(), INTERVAL ?)",
            [$serverId, $this->parseTimeRange($timeRange)]
        );
    }
}
```

### Resource Reservation System
```php
class ResourceReservation {
    public function createReservation($clientId, $resources, $duration) {
        $reservationId = $this->generateReservationId();
        
        $reservation = [
            'id' => $reservationId,
            'client_id' => $clientId,
            'resources' => json_encode($resources),
            'reserved_at' => date('Y-m-d H:i:s'),
            'expires_at' => date('Y-m-d H:i:s', strtotime($duration)),
            'status' => 'active',
            'priority' => $this->calculatePriority($clientId)
        ];
        
        // Hold resources in pending state
        foreach ($resources as $type => $amount) {
            $this->holdResources($type, $amount);
        }
        
        return $this->db->insert('mod_resource_reservations', $reservation);
    }
    
    public function commitReservation($reservationId, $serviceId) {
        $reservation = $this->getReservation($reservationId);
        $resources = json_decode($reservation['resources'], true);
        
        foreach ($resources as $type => $amount) {
            $this->allocateResources($serviceId, $type, $amount);
            $this->releaseHold($type, $amount);
        }
        
        $this->updateReservationStatus($reservationId, 'committed');
    }
    
    public function releaseReservation($reservationId) {
        $reservation = $this->getReservation($reservationId);
        $resources = json_decode($reservation['resources'], true);
        
        foreach ($resources as $type => $amount) {
            $this->releaseResources($type, $amount);
        }
        
        $this->updateReservationStatus($reservationId, 'released');
    }
    
    public function expireReservations() {
        $expired = $this->db->select(
            "SELECT * FROM mod_resource_reservations 
             WHERE status = 'active' AND expires_at < NOW()"
        );
        
        foreach ($expired as $reservation) {
            $this->releaseReservation($reservation['id']);
        }
        
        return count($expired);
    }
}
```

### Billing Integration
```php
class ResourceAllocationBilling {
    public function calculateOverageCharges($clientId, $period = 'monthly') {
        $allocation = $this->getClientAllocation($clientId);
        $usage = $this->getResourceUsage($clientId, $period);
        
        $charges = [];
        $resourceTypes = ['bandwidth', 'disk', 'cpu', 'ram'];
        
        foreach ($resourceTypes as $type) {
            $included = $allocation[$type];
            $used = $usage[$type];
            $overage = max(0, $used - $included);
            
            if ($overage > 0) {
                $rate = $this->getOverageRate($clientId, $type);
                $charges[$type] = [
                    'included' => $included,
                    'used' => $used,
                    'overage' => $overage,
                    'rate' => $rate,
                    'amount' => $overage * $rate
                ];
            }
        }
        
        return $charges;
    }
    
    public function generateResourceStatement($clientId, $period) {
        $statement = [
            'client_id' => $clientId,
            'period' => $period,
            'allocations' => $this->getAllocations($clientId),
            'usage' => $this->getResourceUsage($clientId, $period),
            'overage_charges' => $this->calculateOverageCharges($clientId, $period),
            'total' => $this->calculateTotal($clientId, $period),
            'generated_at' => date('Y-m-d H:i:s')
        ];
        
        $this->saveStatement($statement);
        return $statement;
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_resource_allocation (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    service_id INT,
    server_id INT,
    cpu_cores INT DEFAULT 0,
    ram_mb INT DEFAULT 0,
    disk_gb INT DEFAULT 0,
    bandwidth_tb DECIMAL(10,2) DEFAULT 0,
    iops_limit INT DEFAULT 0,
    concurrent_connections INT DEFAULT 0,
    allocated_at DATETIME,
    updated_at DATETIME,
    INDEX idx_client (client_id),
    INDEX idx_server (server_id)
);

CREATE TABLE mod_resource_reservations (
    id VARCHAR(64) PRIMARY KEY,
    client_id INT NOT NULL,
    resources JSON,
    reserved_at DATETIME,
    expires_at DATETIME,
    status ENUM('pending', 'active', 'committed', 'released', 'expired'),
    priority INT DEFAULT 0,
    INDEX idx_client (client_id),
    INDEX idx_status (status)
);

CREATE TABLE mod_resource_usage_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    service_id INT,
    resource_type VARCHAR(32),
    amount DECIMAL(15,4),
    recorded_at DATETIME,
    INDEX idx_client_time (client_id, recorded_at)
);

CREATE TABLE mod_server_capacity (
    server_id INT PRIMARY KEY,
    total_cpu_cores INT,
    total_ram_mb INT,
    total_disk_gb INT,
    total_bandwidth_tb DECIMAL(10,2),
    reserved_cpu_cores INT DEFAULT 0,
    reserved_ram_mb INT DEFAULT 0,
    reserved_disk_gb INT DEFAULT 0,
    reserved_bandwidth_tb DECIMAL(10,2) DEFAULT 0,
    updated_at DATETIME
);
```

## Usage Examples

### Provisioning with Resource Check
```php
$allocator = new WHMCSResourceAllocation();

// Check server availability
$check = $allocator->checkAvailability($serverId, 'cpu_cores', 4);
if (!$check['available']) {
    throw new Exception("Insufficient CPU resources on server");
}

// Check if adding this service would exceed recommended load
if ($check['utilization'] > 80) {
    logActivity("Warning: Server {$serverId} at high capacity", "warning");
}

// Initialize allocation for the client
$allocator->initializeAllocation($clientId, $productId);
```

### Real-time Usage Monitoring
```php
$allocator = new WHMCSResourceAllocation();
$usage = $allocator->getResourceUsage($clientId);

echo "CPU Cores: {$usage['cpu']}/{$allocation['cpu_cores']}";
echo "RAM: {$usage['ram']}MB/{$allocation['ram_mb']}MB";
```

## Best Practices

1. **Always maintain headroom**: Keep 20-30% server capacity unused for burst handling
2. **Set hard limits**: Enforce limits before resources are exhausted
3. **Log all allocations**: Track resource usage for billing and planning
4. **Use caching**: Cache server metrics to reduce database load
5. **Implement waiting queues**: If resources are unavailable, queue requests with priority
6. **Monitor trends**: Track resource usage patterns to predict capacity needs

## Integration Points

- WHMCS provisioning hooks for automatic resource assignment
- Module callback handlers for resource monitoring agents
- Admin interface for manual resource adjustments
- Client area widgets for usage visualization
- Invoice generation for overage charges