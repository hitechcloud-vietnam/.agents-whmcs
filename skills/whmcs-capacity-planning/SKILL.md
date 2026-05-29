# WHMCS Capacity Planning Skill

## Purpose
Provides strategies and implementations for capacity planning in WHMCS, forecasting resource needs, planning infrastructure scaling, and optimizing resource utilization.

## Implementation Patterns

### Capacity Forecasting Engine
```php
<?php
// includes/CapacityPlanning.class.php

class CapacityPlanningEngine {
    private $db;
    private $metricsHistory;
    private $forecastModel;
    
    public function __construct() {
        $this->db = console::db();
        $this->metricsHistory = new MetricsHistory();
        $this->forecastModel = new ForecastModel();
    }
    
    // Generate capacity forecast
    public function generateForecast($serverId, $horizon = 90) {
        $historicalData = $this->metricsHistory->getData($serverId, $horizon);
        $growthRate = $this->calculateGrowthRate($historicalData);
        $seasonality = $this->detectSeasonality($historicalData);
        
        $forecast = [];
        $currentCapacity = $this->getCurrentCapacity($serverId);
        
        for ($days = 1; $days <= $horizon; $days++) {
            $date = date('Y-m-d', strtotime("+{$days} days"));
            $projectedUsage = $this->projectUsage(
                $historicalData,
                $growthRate,
                $seasonality,
                $days
            );
            
            $forecast[$date] = [
                'projected_usage' => $projectedUsage,
                'capacity' => $currentCapacity,
                'headroom' => $currentCapacity - $projectedUsage,
                'utilization_pct' => ($projectedUsage / $currentCapacity) * 100,
                'confidence' => $this->calculateConfidence($days, count($historicalData))
            ];
        }
        
        $this->cacheForecast($serverId, $forecast);
        return $forecast;
    }
    
    private function projectUsage($historicalData, $growthRate, $seasonality, $daysAhead) {
        $baseUsage = end($historicalData)['usage'];
        $growth = pow(1 + $growthRate, $daysAhead / 30); // Monthly growth
        $seasonalFactor = $this->getSeasonalFactor($seasonality, $daysAhead);
        
        return $baseUsage * $growth * $seasonalFactor;
    }
    
    private function calculateGrowthRate($data) {
        if (count($data) < 2) return 0;
        
        $firstHalf = array_slice($data, 0, floor(count($data) / 2));
        $secondHalf = array_slice($data, floor(count($data) / 2));
        
        $firstAvg = array_sum(array_column($firstHalf, 'usage')) / count($firstHalf);
        $secondAvg = array_sum(array_column($secondHalf, 'usage')) / count($secondHalf);
        
        return ($secondAvg - $firstAvg) / $firstAvg;
    }
    
    public function detectSeasonality($data) {
        if (count($data) < 60) return null; // Need 60+ days
        
        $weeklyPattern = [];
        foreach ($data as $record) {
            $dayOfWeek = date('N', strtotime($record['date']));
            if (!isset($weeklyPattern[$dayOfWeek])) {
                $weeklyPattern[$dayOfWeek] = [];
            }
            $weeklyPattern[$dayOfWeek][] = $record['usage'];
        }
        
        foreach ($weeklyPattern as $day => $values) {
            $weeklyPattern[$day] = array_sum($values) / count($values);
        }
        
        $avg = array_sum($weeklyPattern) / count($weeklyPattern);
        foreach ($weeklyPattern as $day => $value) {
            $weeklyPattern[$day] = $value / $avg;
        }
        
        return $weeklyPattern;
    }
}
```

### Infrastructure Planning
```php
class InfrastructurePlanner {
    public function planScaling($currentDemand, $targetDate, $options = []) {
        $defaults = [
            'headroom_pct' => 20,
            'max_server_size' => 64,
            'min_servers' => 1,
            'load_balancing' => true
        ];
        $config = array_merge($defaults, $options);
        
        $demandForecast = $this->forecastDemand($currentDemand, $targetDate);
        $requiredCapacity = $demandForecast * (1 + $config['headroom_pct'] / 100);
        
        $servers = $this->determineOptimalServerConfig(
            $requiredCapacity,
            $config['max_server_size']
        );
        
        $plan = [
            'timeline' => $targetDate,
            'forecasted_demand' => $demandForecast,
            'required_capacity' => $requiredCapacity,
            'recommended_servers' => $servers,
            'estimated_cost' => $this->estimateCost($servers),
            'implementation_steps' => $this->generateSteps($servers),
            'risks' => $this->identifyRisks($servers)
        ];
        
        return $plan;
    }
    
    private function determineOptimalServerConfig($requiredCapacity, $maxSize) {
        $servers = [];
        $remaining = $requiredCapacity;
        
        while ($remaining > 0) {
            $serverSize = min($remaining, $maxSize);
            $servers[] = [
                'size' => $serverSize,
                'type' => $this->recommendServerType($serverSize),
                'estimated_cost' => $this->getServerCost($serverSize)
            ];
            $remaining -= $serverSize;
        }
        
        return $servers;
    }
    
    public function generateMigrationPlan($fromServer, $toServer) {
        $services = $this->getMigratableServices($fromServer);
        $migrationPlan = [];
        
        // Group services by dependency
        $groups = $this->groupByDependency($services);
        
        foreach ($groups as $group) {
            $migrationPlan[] = [
                'services' => array_column($group, 'id'),
                'sequence' => $this->determineSequence($group),
                'downtime_estimate' => $this->estimateDowntime($group),
                'rollback_plan' => $this->createRollbackPlan($group)
            ];
        }
        
        return $migrationPlan;
    }
}
```

### Resource Utilization Analyzer
```php
class ResourceUtilizationAnalyzer {
    public function analyze($timeRange = '30d') {
        $servers = $this->getAllServers();
        $analysis = [];
        
        foreach ($servers as $server) {
            $utilization = $this->calculateUtilization($server['id'], $timeRange);
            $efficiency = $this->calculateEfficiency($server, $utilization);
            
            $analysis[$server['id']] = [
                'server_name' => $server['name'],
                'avg_utilization' => $utilization['avg'],
                'peak_utilization' => $utilization['peak'],
                'idle_time_pct' => $utilization['idle'],
                'efficiency_score' => $efficiency['score'],
                'recommendations' => $efficiency['recommendations'],
                'cost_per_unit' => $this->calculateCostPerUnit($server, $utilization)
            ];
        }
        
        return $analysis;
    }
    
    private function calculateUtilization($serverId, $timeRange) {
        $metrics = $this->db->select(
            "SELECT 
                AVG(cpu_usage) as avg_cpu,
                MAX(cpu_usage) as max_cpu,
                AVG(memory_usage) as avg_ram,
                MAX(memory_usage) as max_ram,
                AVG(disk_usage) as avg_disk,
                MAX(disk_usage) as max_disk,
                AVG(bandwidth_usage) as avg_bandwidth,
                MAX(bandwidth_usage) as max_bandwidth
             FROM mod_server_metrics
             WHERE server_id = ? AND timestamp > DATE_SUB(NOW(), INTERVAL ?)",
            [$serverId, $timeRange]
        );
        
        $idleTime = $this->db->select(
            "SELECT COUNT(*) / (SELECT COUNT(*) FROM mod_server_metrics 
             WHERE server_id = ?) * 100 as idle_pct
             FROM mod_server_metrics 
             WHERE server_id = ? AND cpu_usage < 10",
            [$serverId, $serverId]
        );
        
        return [
            'avg' => ($metrics['avg_cpu'] + $metrics['avg_ram']) / 2,
            'peak' => max($metrics['max_cpu'], $metrics['max_ram']),
            'idle' => $idleTime['idle_pct']
        ];
    }
    
    private function calculateEfficiency($server, $utilization) {
        $recommendations = [];
        
        if ($utilization['avg'] < 30) {
            $recommendations[] = [
                'type' => 'consolidate',
                'message' => 'Server is underutilized. Consider consolidating workloads.',
                'potential_savings' => $this->estimateSavings($server, 'consolidation')
            ];
        }
        
        if ($utilization['avg'] > 80) {
            $recommendations[] = [
                'type' => 'upgrade',
                'message' => 'Server is approaching capacity. Plan for upgrade.',
                'urgency' => 'high'
            ];
        }
        
        $score = $this->calculateEfficiencyScore($utilization);
        return ['score' => $score, 'recommendations' => $recommendations];
    }
}
```

### Capacity Planning Dashboard Data
```php
class CapacityDashboard {
    public function getDashboardData($userId = null) {
        $dashboard = [
            'overview' => $this->getOverviewStats(),
            'forecasts' => $this->getActiveForecasts(),
            'alerts' => $this->getCapacityAlerts(),
            'servers' => $this->getServerCapacitySummary(),
            'trends' => $this->getUtilizationTrends(),
            'recommendations' => $this->getRecommendations()
        ];
        
        if ($userId) {
            $dashboard['permissions'] = $this->getUserPermissions($userId);
            $dashboard['custom_alerts'] = $this->getUserAlertPrefs($userId);
        }
        
        return $dashboard;
    }
    
    private function getOverviewStats() {
        return [
            'total_servers' => $this->db->count('servers'),
            'total_clients' => $this->db->count('tblclients'),
            'avg_utilization' => $this->getAverageUtilization(),
            'capacity_remaining' => $this->getTotalRemainingCapacity(),
            'upcoming_upgrades' => $this->getScheduledUpgrades(),
            'monthly_growth_rate' => $this->calculateMonthlyGrowth()
        ];
    }
    
    public function getCapacityAlerts() {
        $alerts = [];
        
        // Check servers approaching capacity
        $serversAtRisk = $this->db->select(
            "SELECT s.*, 
                (SUM(m.allocated_cpu) / s.total_cpu) * 100 as cpu_utilization,
                (SUM(m.allocated_ram) / s.total_ram) * 100 as ram_utilization
             FROM servers s
             LEFT JOIN mod_resource_allocation m ON m.server_id = s.id
             GROUP BY s.id
             HAVING cpu_utilization > 80 OR ram_utilization > 80"
        );
        
        foreach ($serversAtRisk as $server) {
            $alerts[] = [
                'type' => 'capacity_warning',
                'severity' => $server['cpu_utilization'] > 90 ? 'critical' : 'warning',
                'server' => $server['name'],
                'message' => "Server at {$server['cpu_utilization']}% CPU utilization",
                'recommendation' => 'Schedule capacity upgrade'
            ];
        }
        
        // Check for demand spikes
        $demandSpike = $this->detectDemandSpike();
        if ($demandSpike) {
            $alerts[] = [
                'type' => 'demand_spike',
                'severity' => 'info',
                'message' => 'Unusual demand increase detected',
                'details' => $demandSpike
            ];
        }
        
        return $alerts;
    }
}
```

### Scaling Recommendation Engine
```php
class ScalingRecommendationEngine {
    public function generateRecommendations() {
        $recommendations = [];
        
        // Analyze current capacity
        $utilization = $this->analyzeCurrentUtilization();
        
        foreach ($utilization as $serverType => $data) {
            if ($data['avg_utilization'] > 85) {
                $recommendations[] = $this->generateUpgradeRec($serverType, $data);
            }
            
            if ($data['utilization_growth'] > 0.15) {
                $recommendations[] = $this->generateExpansionRec($serverType, $data);
            }
        }
        
        // Sort by impact and urgency
        usort($recommendations, function($a, $b) {
            return $b['impact'] <=> $a['impact'];
        });
        
        return $recommendations;
    }
    
    private function generateUpgradeRec($serverType, $data) {
        $currentCost = $this->getMonthlyCost($serverType);
        $upgradedCost = $this->getMonthlyCost($this->getNextTier($serverType));
        
        return [
            'type' => 'upgrade',
            'server_type' => $serverType,
            'current_utilization' => $data['avg_utilization'],
            'recommended_action' => 'Upgrade to ' . $this->getNextTier($serverType),
            'estimated_cost_increase' => $upgradedCost - $currentCost,
            'roi_days' => $this->calculateUpgradeROI($serverType, $data),
            'priority' => $data['avg_utilization'] > 95 ? 'urgent' : 'high'
        ];
    }
    
    public function simulateScenario($changes) {
        $simulation = [
            'baseline' => $this->getCurrentState(),
            'changes' => $changes,
            'projected' => $this->calculateProjectedState($changes),
            'impact' => $this->calculateImpact($changes)
        ];
        
        return $simulation;
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_capacity_forecasts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    server_id INT NOT NULL,
    forecast_date DATE NOT NULL,
    projected_usage DECIMAL(10,4),
    capacity DECIMAL(10,4),
    confidence DECIMAL(5,2),
    created_at DATETIME,
    UNIQUE KEY idx_server_date (server_id, forecast_date),
    INDEX idx_forecast (forecast_date)
);

CREATE TABLE mod_capacity_alerts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    server_id INT,
    alert_type VARCHAR(50),
    severity ENUM('info', 'warning', 'critical'),
    message TEXT,
    created_at DATETIME,
    acknowledged_at DATETIME,
    acknowledged_by INT,
    INDEX idx_severity (severity),
    INDEX idx_server (server_id)
);

CREATE TABLE mod_resource_metrics_history (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    server_id INT NOT NULL,
    metric_type VARCHAR(32),
    value DECIMAL(15,4),
    recorded_at DATETIME,
    INDEX idx_server_type (server_id, metric_type),
    INDEX idx_recorded (recorded_at)
);

CREATE TABLE mod_scaling_recommendations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    server_type VARCHAR(50),
    recommendation_type VARCHAR(50),
    priority ENUM('low', 'medium', 'high', 'urgent'),
    details JSON,
    impact_score DECIMAL(5,2),
    created_at DATETIME,
    implemented_at DATETIME,
    INDEX idx_priority (priority)
);
```

## Usage Examples

### Generate Capacity Report
```php
$planner = new CapacityPlanningEngine();
$forecast = $planner->generateForecast($serverId, 90);

// Check for capacity issues
foreach ($forecast as $date => $data) {
    if ($data['utilization_pct'] > 90) {
        alert("Capacity warning: {$date}");
    }
}
```

### Get Scaling Recommendations
```php
$engine = new ScalingRecommendationEngine();
$recs = $engine->generateRecommendations();

foreach ($recs as $rec) {
    echo "{$rec['priority']}: {$rec['recommendation']}\n";
    echo "  Estimated cost: $" . number_format($rec['estimated_cost_increase']) . "/mo\n";
}
```

### Run Capacity Simulation
```php
$simulator = new ScalingRecommendationEngine();
$result = $simulator->simulateScenario([
    ['add_servers' => 2, 'type' => 'large'],
    ['upgrade_tier' => 'premium']
]);

print_r($result['projected']);
print_r($result['impact']);
```

## Best Practices

1. **Monitor trends**: Track usage over time to identify patterns
2. **Plan ahead**: Generate forecasts 90+ days in advance
3. **Maintain headroom**: Keep 20-30% capacity buffer
4. **Automate alerts**: Set up alerts at 70%, 80%, 90% thresholds
5. **Review regularly**: Weekly capacity review meetings
6. **Cost optimization**: Balance capacity with cost efficiency
7. **Test scenarios**: Simulate growth scenarios to validate plans