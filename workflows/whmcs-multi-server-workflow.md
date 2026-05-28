# WHMCS Multi-Server Management Workflow

## Purpose

Procedures for managing multiple hosting servers through WHMCS, including provisioning automation, server monitoring, load balancing, and failover configuration.

## Prerequisites

- Multiple servers configured in WHMCS
- Server API credentials
- Server monitoring tools
- Load balancing solution (if applicable)

## Workflow Steps

### Step 1: Configure Server Groups

Organize servers for efficient management:

```php
<?php
// Configuration: WHMCS Admin > System > Servers
// Create server groups via API

use WHMCS\Module\Server;

class ServerGroupManager
{
    public function createServerGroup($name, $description, $assignments = [])
    {
        $serverGroup = Capsule::table('tblservergroups')->insertGetId([
            'name' => $name,
            'description' => $description,
            'created_at' => Carbon::now()->toDateTimeString(),
        ]);
        
        // Assign servers to group
        foreach ($assignments as $order => $serverId) {
            Capsule::table('tblservergroupsrel')->insert([
                'groupid' => $serverGroup,
                'serverid' => $serverId,
                'sortorder' => $order,
            ]);
        }
        
        return $serverGroup;
    }
    
    public function getServerGroup($groupId)
    {
        $group = Capsule::table('tblservergroups')
            ->where('id', $groupId)
            ->first();
        
        $servers = Capsule::table('tblservergroupsrel')
            ->join('tblservers', 'tblservergroupsrel.serverid', '=', 'tblservers.id')
            ->where('tblservergroupsrel.groupid', $groupId)
            ->orderBy('tblservergroupsrel.sortorder')
            ->select('tblservers.*')
            ->get();
        
        return [
            'group' => $group,
            'servers' => $servers,
        ];
    }
    
    public function getBestServer($groupId)
    {
        $servers = $this->getServerGroup($groupId)['servers'];
        
        // Sort by load and availability
        usort($servers, function($a, $b) {
            // Prefer servers with lower load
            $aLoad = $this->getServerLoad($a);
            $bLoad = $this->getServerLoad($b);
            
            // Skip unavailable servers
            if ($aLoad === null && $bLoad !== null) return 1;
            if ($bLoad === null && $aLoad !== null) return -1;
            if ($aLoad === null && $bLoad === null) return 0;
            
            return $aLoad <=> $bLoad;
        });
        
        return $servers[0] ?? null;
    }
    
    private function getServerLoad($server)
    {
        // Check server status via API
        try {
            $response = $this->callServerAPI($server, 'status');
            return $response['load'] ?? null;
        } catch (\Exception $e) {
            return null; // Server unavailable
        }
    }
    
    private function callServerAPI($server, $action)
    {
        // Implementation depends on server module
    }
}
```

### Step 2: Implement Server Health Checks

Monitor server availability:

```php
<?php
// modules/custom/server_health_check.php
// Run every 5 minutes via cron

require_once __DIR__ . '/init.php';

class ServerHealthChecker
{
    private $alerts = [];
    
    public function checkAllServers()
    {
        $servers = Capsule::table('tblservers')
            ->where('disabled', 0)
            ->get();
        
        foreach ($servers as $server) {
            $this->checkServer($server);
        }
        
        $this->processAlerts();
    }
    
    private function checkServer($server)
    {
        $checks = [
            'ping' => $this->checkPing($server),
            'port' => $this->checkPort($server),
            'api' => $this->checkAPI($server),
            'load' => $this->checkLoad($server),
            'disk' => $this->checkDiskSpace($server),
            'memory' => $this->checkMemory($server),
        ];
        
        // Update server status in WHMCS
        $online = $checks['ping'] && $checks['port'] && $checks['api'];
        
        Capsule::table('tblservers')
            ->where('id', $server->id)
            ->update([
                'disabled' => $online ? 0 : 1,
                'last_updated' => Carbon::now()->toDateTimeString(),
            ]);
        
        // Log check results
        $this->logServerHealth($server->id, $checks);
        
        // Generate alerts for issues
        $this->generateAlerts($server, $checks);
    }
    
    private function checkPing($server)
    {
        $host = gethostbyname($server->hostname);
        $exec = exec("ping -c 1 -W 5 {$host}");
        return strpos($exec, '1 received') !== false || strpos($exec, '1 packets') !== false;
    }
    
    private function checkPort($server)
    {
        $port = $server->port ?: 22;
        $connection = @fsockopen($server->ipaddress, $port, $errno, $errstr, 5);
        
        if ($connection) {
            fclose($connection);
            return true;
        }
        
        return false;
    }
    
    private function checkAPI($server)
    {
        // Test server module API connection
        $params = [
            'server' => $server,
            'action' => 'status',
        ];
        
        try {
            $result = ServerFunctions::call($server->type, 'TestConnection', $params);
            return $result['success'] ?? false;
        } catch (\Exception $e) {
            return false;
        }
    }
    
    private function checkLoad($server)
    {
        try {
            $response = $this->getServerMetrics($server);
            $load = $response['load'] ?? 0;
            
            // Alert if load > 80%
            if ($load > 80) {
                $this->alerts[] = [
                    'server' => $server->hostname,
                    'type' => 'high_load',
                    'severity' => $load > 95 ? 'critical' : 'warning',
                    'message' => "Server load is {$load}%",
                ];
            }
            
            return $load;
        } catch (\Exception $e) {
            return null;
        }
    }
    
    private function checkDiskSpace($server)
    {
        try {
            $response = $this->getServerMetrics($server);
            $diskUsed = $response['disk_used_percent'] ?? 0;
            
            if ($diskUsed > 90) {
                $this->alerts[] = [
                    'server' => $server->hostname,
                    'type' => 'disk_space',
                    'severity' => $diskUsed > 95 ? 'critical' : 'warning',
                    'message' => "Disk space at {$diskUsed}%",
                ];
            }
            
            return $diskUsed;
        } catch (\Exception $e) {
            return null;
        }
    }
    
    private function checkMemory($server)
    {
        try {
            $response = $this->getServerMetrics($server);
            $memUsed = $response['memory_used_percent'] ?? 0;
            
            if ($memUsed > 90) {
                $this->alerts[] = [
                    'server' => $server->hostname,
                    'type' => 'memory',
                    'severity' => $memUsed > 95 ? 'critical' : 'warning',
                    'message' => "Memory usage at {$memUsed}%",
                ];
            }
            
            return $memUsed;
        } catch (\Exception $e) {
            return null;
        }
    }
    
    private function getServerMetrics($server)
    {
        // Call server's API/monitoring endpoint
        $url = "https://{$server->ipaddress}:4083/api/metrics";
        
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
        curl_setopt($ch, CURLOPT_TIMEOUT, 10);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true) ?? [];
    }
    
    private function logServerHealth($serverId, $checks)
    {
        Capsule::table('mod_server_health_logs')->insert([
            'server_id' => $serverId,
            'ping_ok' => $checks['ping'],
            'port_ok' => $checks['port'],
            'api_ok' => $checks['api'],
            'load' => $checks['load'],
            'disk' => $checks['disk'],
            'memory' => $checks['memory'],
            'checked_at' => Carbon::now()->toDateTimeString(),
        ]);
    }
    
    private function generateAlerts($server, $checks)
    {
        if (!$checks['ping']) {
            $this->alerts[] = [
                'server' => $server->hostname,
                'type' => 'offline',
                'severity' => 'critical',
                'message' => 'Server is not responding to ping',
            ];
        }
        
        if (!$checks['api']) {
            $this->alerts[] = [
                'server' => $server->hostname,
                'type' => 'api_error',
                'severity' => 'warning',
                'message' => 'Server API is not responding',
            ];
        }
    }
    
    private function processAlerts()
    {
        foreach ($this->alerts as $alert) {
            // Log to activity log
            logActivity("Server Alert: {$alert['server']} - {$alert['message']}");
            
            // Send notification for critical alerts
            if ($alert['severity'] === 'critical') {
                sendAdminNotification(
                    'Server Critical Alert',
                    $alert
                );
            }
        }
        
        // Store alerts in database
        if (!empty($this->alerts)) {
            Capsule::table('mod_server_alerts')->insert($this->alerts);
        }
    }
}

// Run health check
$healthChecker = new ServerHealthChecker();
$healthChecker->checkAllServers();
```

### Step 3: Implement Load Balancing

Route services based on server capacity:

```php
<?php
// modules/custom/load_balancer.php

namespace WHMCS\Custom;

class LoadBalancer
{
    private $strategy = 'least_connections';
    
    public function selectServer($serverGroupId, $requirements = [])
    {
        $servers = $this->getAvailableServers($serverGroupId);
        
        if (empty($servers)) {
            throw new \Exception('No available servers in group');
        }
        
        switch ($this->strategy) {
            case 'round_robin':
                return $this->roundRobin($servers);
                
            case 'least_connections':
                return $this->leastConnections($servers);
                
            case 'weighted':
                return $this->weightedSelection($servers);
                
            case 'geolocation':
                return $this->geoAwareSelection($servers, $requirements['location'] ?? null);
                
            default:
                return $this->roundRobin($servers);
        }
    }
    
    private function getAvailableServers($groupId)
    {
        return Capsule::table('tblservers')
            ->join('tblservergroupsrel', 'tblservers.id', '=', 'tblservergroupsrel.serverid')
            ->leftJoin('mod_server_health_logs', 'tblservers.id', '=', 'mod_server_health_logs.server_id')
            ->where('tblservergroupsrel.groupid', $groupId)
            ->where('tblservers.disabled', 0)
            ->where(function($query) {
                $query->whereNull('mod_server_health_logs.checked_at')
                    ->orWhere('mod_server_health_logs.checked_at', '>', Carbon::now()->subMinutes(10)->toDateTimeString());
            })
            ->select([
                'tblservers.*',
                'mod_server_health_logs.load',
                'mod_server_health_logs.disk',
                'mod_server_health_logs.memory',
                'mod_server_health_logs.checked_at',
            ])
            ->get();
    }
    
    private function leastConnections($servers)
    {
        // Filter out unhealthy servers
        $healthyServers = array_filter($servers, function($server) {
            return $server->load < 90 && $server->disk < 90 && $server->memory < 90;
        });
        
        if (empty($healthyServers)) {
            // Fall back to least loaded even if over thresholds
            $healthyServers = $servers;
        }
        
        // Sort by load
        usort($healthyServers, function($a, $b) {
            return $a->load <=> $b->load;
        });
        
        return $healthyServers[0];
    }
    
    private function roundRobin($servers)
    {
        // Get last used server from cache
        $lastUsed = Cache::get('loadbalancer_last_' . md5(json_encode($servers->pluck('id')->toArray())));
        
        $lastIndex = $lastUsed ? array_search($lastUsed, $servers->pluck('id')->toArray()) : -1;
        $nextIndex = ($lastIndex + 1) % count($servers);
        
        Cache::set('loadbalancer_last_' . md5(json_encode($servers->pluck('id')->toArray())), $servers[$nextIndex]->id, 300);
        
        return $servers[$nextIndex];
    }
    
    private function weightedSelection($servers)
    {
        // Servers with lower load have higher weight
        $weightedServers = [];
        
        foreach ($servers as $server) {
            $weight = max(1, 100 - ($server->load ?? 50));
            for ($i = 0; $i < $weight; $i++) {
                $weightedServers[] = $server;
            }
        }
        
        return $weightedServers[array_rand($weightedServers)];
    }
    
    private function geoAwareSelection($servers, $clientLocation)
    {
        if (!$clientLocation) {
            return $this->leastConnections($servers);
        }
        
        // Sort by geographic proximity
        usort($servers, function($a, $b) use ($clientLocation) {
            $aDistance = $this->calculateDistance($clientLocation, $a);
            $bDistance = $this->calculateDistance($clientLocation, $b);
            return $aDistance <=> $bDistance;
        });
        
        // Return closest server that's healthy
        foreach ($servers as $server) {
            if (($server->load ?? 50) < 90) {
                return $server;
            }
        }
        
        return $servers[0];
    }
    
    private function calculateDistance($loc1, $server)
    {
        // Simplified distance calculation
        $serverLocation = $server->location ?? ['lat' => 0, 'lon' => 0];
        
        return sqrt(
            pow($loc1['lat'] - $serverLocation['lat'], 2) +
            pow($loc1['lon'] - $serverLocation['lon'], 2)
        );
    }
}
```

### Step 4: Implement Failover

Configure automatic failover:

```php
<?php
// modules/custom/hooks/failover_hooks.php

// Hook: When server becomes unavailable
add_hook('ServerOffline', 1, function($params) {
    $server = $params['server'];
    
    logActivity("Server failover triggered for {$server->hostname}");
    
    // Mark server as disabled
    Capsule::table('tblservers')
        ->where('id', $server->id)
        ->update(['disabled' => 1]);
    
    // Find alternative servers
    $failover = new \WHMCS\Custom\FailoverManager();
    $alternativeServer = $failover->findAlternative($server);
    
    if ($alternativeServer) {
        // Migrate services
        $failover->migrateServices($server, $alternativeServer);
        
        sendAdminNotification(
            'Server Failover Completed',
            [
                'original_server' => $server->hostname,
                'new_server' => $alternativeServer->hostname,
                'services_migrated' => $failover->getMigratedCount(),
            ]
        );
    }
});

class FailoverManager
{
    private $migratedCount = 0;
    
    public function findAlternative($failedServer)
    {
        // Find servers in same group or with same capabilities
        $alternatives = Capsule::table('tblservers')
            ->where('id', '!=', $failedServer->id)
            ->where('type', $failedServer->type)
            ->where('disabled', 0)
            ->where('maxaccounts', '>', Capsule::raw('numaccounts'))
            ->orderBy('maxaccounts', 'asc')
            ->first();
        
        return $alternatives;
    }
    
    public function migrateServices($fromServer, $toServer)
    {
        // Get services on failed server
        $services = Capsule::table('tblhosting')
            ->where('server', $fromServer->id)
            ->where('domainstatus', 'Active')
            ->get();
        
        foreach ($services as $service) {
            $this->migrateService($service, $toServer);
        }
    }
    
    private function migrateService($service, $newServer)
    {
        // Update service record
        Capsule::table('tblhosting')
            ->where('id', $service->id)
            ->update(['server' => $newServer->id]);
        
        // Attempt to recreate account on new server
        try {
            $params = [
                'serviceid' => $service->id,
                'model' => $service,
            ];
            
            $result = ServerFunctions::call($newServer->type, 'CreateAccount', $params);
            
            if ($result === 'success') {
                $this->migratedCount++;
                logActivity("Service {$service->id} migrated to {$newServer->hostname}");
            }
        } catch (\Exception $e) {
            logActivity("Service {$service->id} migration failed: " . $e->getMessage());
        }
    }
    
    public function getMigratedCount()
    {
        return $this->migratedCount;
    }
}
```

### Step 5: Create Server Management Dashboard

Admin interface for multi-server management:

```php
<?php
// admin/server_management.php

require_once __DIR__ . '/../init.php';

$action = $_GET['action'] ?? 'overview';

switch ($action) {
    case 'overview':
        echo $twig->render('admin/servers/overview.html', [
            'serverGroups' => getServerGroups(),
            'healthSummary' => getHealthSummary(),
            'recentAlerts' => getRecentAlerts(),
        ]);
        break;
        
    case 'server_details':
        $serverId = (int) $_GET['id'];
        echo $twig->render('admin/servers/details.html', [
            'server' => getServerDetails($serverId),
            'services' => getServerServices($serverId),
            'healthLogs' => getServerHealthLogs($serverId, 100),
        ]);
        break;
        
    case 'migrate_services':
        $fromServer = (int) $_GET['from'];
        $toServer = (int) $_GET['to'];
        migrateServicesBetweenServers($fromServer, $toServer);
        redir('action=server_details&id=' . $toServer);
        break;
}

function getServerGroups()
{
    return Capsule::table('tblservergroups')
        ->with('servers')
        ->get()
        ->map(function($group) {
            $group->servers = Capsule::table('tblservergroupsrel')
                ->join('tblservers', 'tblservergroupsrel.serverid', '=', 'tblservers.id')
                ->where('tblservergroupsrel.groupid', $group->id)
                ->select('tblservers.*')
                ->get();
            
            $group->activeServers = $group->servers->filter(fn($s) => !$s->disabled)->count();
            $group->totalServices = Capsule::table('tblhosting')
                ->whereIn('server', $group->servers->pluck('id'))
                ->count();
            
            return $group;
        });
}

function getHealthSummary()
{
    return [
        'total' => Capsule::table('tblservers')->count(),
        'online' => Capsule::table('tblservers')->where('disabled', 0)->count(),
        'offline' => Capsule::table('tblservers')->where('disabled', 1)->count(),
        'warnings' => Capsule::table('mod_server_alerts')
            ->where('created_at', '>', Carbon::now()->subHours(24)->toDateTimeString())
            ->count(),
    ];
}
```

## Verification Checklist

- [ ] Server groups configured
- [ ] Health check cron running
- [ ] Server monitoring active
- [ ] Load balancing strategy selected
- [ ] Failover mechanism tested
- [ ] Migration procedures documented
- [ ] Admin dashboard accessible
- [ ] Alert notifications configured
- [ ] Recovery procedures tested

## Related Skills and Documentation

- [WHMCS Provisioning Automation](whmcs-provisioning-automation-workflow.md)
- [WHMCS Disaster Recovery](whmcs-disaster-recovery-workflow.md)
- WHMCS Server Configuration: https://docs.whmcs.com/Server_Configuration

## Notes

- Regularly test failover procedures
- Monitor server capacity and plan for growth
- Keep server modules updated
- Document server-specific configurations
- Maintain server access credentials securely
- Review and optimize load balancing regularly
