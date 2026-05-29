# WHMCS Canary Deployment Skill

## Purpose
Provides patterns for implementing canary deployments in WHMCS, gradually releasing features to a subset of users while monitoring stability and performance.

## Implementation Patterns

### Canary Deployment Manager
```php
<?php
class CanaryDeployment {
    private $db;
    
    public function deploy($featureId, $initialPercentage = 5) {
        $deployment = [
            'feature_id' => $featureId,
            'status' => 'rolling_out',
            'current_percentage' => $initialPercentage,
            'target_percentage' => 100,
            'health_score' => 100,
            'started_at' => date('Y-m-d H:i:s')
        ];
        
        $deploymentId = $this->db->insert('mod_canary_deployments', $deployment);
        
        // Schedule incremental increases
        $this->scheduleIncrements($deploymentId);
        
        return $deploymentId;
    }
    
    public function updateHealthScore($deploymentId) {
        $deployment = $this->getDeployment($deploymentId);
        
        $metrics = $this->getDeploymentMetrics($deploymentId);
        
        // Calculate health score
        $errorRate = $metrics['error_count'] / max(1, $metrics['request_count']);
        $latencyScore = min(100, (200 / max(1, $metrics['avg_latency'])) * 100);
        
        $healthScore = (1 - $errorRate) * 50 + $latencyScore * 0.5;
        
        $this->db->where('id', $deploymentId)->update('mod_canary_deployments', [
            'health_score' => $healthScore,
            'last_check_at' => date('Y-m-d H:i:s')
        ]);
        
        if ($healthScore < 70) {
            $this->rollback($deploymentId);
        }
        
        return $healthScore;
    }
    
    public function incrementPercentage($deploymentId) {
        $deployment = $this->getDeployment($deploymentId);
        
        $newPercentage = min(100, $deployment['current_percentage'] + 10);
        
        $this->db->where('id', $deploymentId)->update('mod_canary_deployments', [
            'current_percentage' => $newPercentage
        ]);
        
        if ($newPercentage >= 100) {
            $this->completeDeployment($deploymentId);
        }
    }
    
    public function rollback($deploymentId) {
        $this->db->where('id', $deploymentId)->update('mod_canary_deployments', [
            'status' => 'rolled_back',
            'rolled_back_at' => date('Y-m-d H:i:s')
        ]);
        
        // Disable feature for all canary users
        $this->disableFeatureForCanaryUsers($deploymentId);
    }
    
    private function getDeploymentMetrics($deploymentId) {
        return $this->db->select(
            "SELECT COUNT(*) as request_count, SUM(errors) as error_count, AVG(latency) as avg_latency
             FROM mod_canary_metrics WHERE deployment_id = ? AND recorded_at > DATE_SUB(NOW(), INTERVAL 5 MINUTE)",
            [$deploymentId]
        );
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_canary_deployments (
    id INT AUTO_INCREMENT PRIMARY KEY,
    feature_id VARCHAR(100),
    status ENUM('rolling_out', 'completed', 'rolled_back'),
    current_percentage INT,
    target_percentage INT,
    health_score DECIMAL(5,2),
    started_at DATETIME,
    completed_at DATETIME,
    rolled_back_at DATETIME
);

CREATE TABLE mod_canary_metrics (
    id INT AUTO_INCREMENT PRIMARY KEY,
    deployment_id INT,
    request_count INT,
    error_count INT,
    avg_latency DECIMAL(10,2),
    recorded_at DATETIME
);
```

## Usage Examples
```php
$canary = new CanaryDeployment();
$deploymentId = $canary->deploy('new_feature', 5);

// Monitor health
$health = $canary->updateHealthScore($deploymentId);
if ($health >= 90) {
    $canary->incrementPercentage($deploymentId);
}
```
