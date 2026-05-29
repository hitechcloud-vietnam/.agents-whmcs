# WHMCS Rollout Strategy Skill

## Purpose
Provides patterns for implementing gradual rollout strategies in WHMCS, controlling feature releases across different phases and user segments.

## Implementation Patterns

### Rollout Strategy Manager
```php
<?php
class RolloutStrategy {
    private $db;
    
    public function createStrategy($data) {
        $strategy = [
            'feature_name' => $data['feature_name'],
            'phases' => json_encode($data['phases']),
            'current_phase' => 0,
            'status' => 'draft',
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        return $this->db->insert('mod_rollout_strategies', $strategy);
    }
    
    public function advancePhase($strategyId) {
        $strategy = $this->getStrategy($strategyId);
        $phases = json_decode($strategy['phases'], true);
        
        $nextPhase = $strategy['current_phase'] + 1;
        
        if ($nextPhase >= count($phases)) {
            $this->completeStrategy($strategyId);
            return false;
        }
        
        $this->db->where('id', $strategyId)->update('mod_rollout_strategies', [
            'current_phase' => $nextPhase,
            'last_advanced_at' => date('Y-m-d H:i:s')
        ]);
        
        // Apply next phase rules
        $this->applyPhase($strategyId, $phases[$nextPhase]);
        
        return true;
    }
    
    private function applyPhase($strategyId, $phase) {
        // Update feature flag percentage
        $this->db->where('name', $phase['feature_name'])->update('mod_feature_flags', [
            'rollout_percentage' => $phase['percentage']
        ]);
        
        // Enable for specific segments
        if ($phase['segments']) {
            $this->enableForSegments($phase['feature_name'], $phase['segments']);
        }
    }
    
    public function getTargetUsers($strategyId) {
        $strategy = $this->getStrategy($strategyId);
        $phases = json_decode($strategy['phases'], true);
        $currentPhase = $phases[$strategy['current_phase']];
        
        return $this->getUsersForSegment($currentPhase['segments']);
    }
    
    private function getUsersForSegment($segments) {
        $query = "SELECT id FROM tblclients WHERE 1=1";
        
        if (in_array('beta_testers', $segments)) {
            $query .= " AND beta_tester = 1";
        }
        if (in_array('enterprise', $segments)) {
            $query .= " AND client_group = 'enterprise'";
        }
        if (in_array('new_customers', $segments)) {
            $query .= " AND regdate > DATE_SUB(NOW(), INTERVAL 30 DAY)";
        }
        
        return $this->db->select($query);
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_rollout_strategies (
    id INT AUTO_INCREMENT PRIMARY KEY,
    feature_name VARCHAR(100),
    phases JSON,
    current_phase INT DEFAULT 0,
    status ENUM('draft', 'active', 'paused', 'completed') DEFAULT 'draft',
    created_at DATETIME,
    last_advanced_at DATETIME,
    completed_at DATETIME
);
```

## Usage Examples
```php
$rollout = new RolloutStrategy();
$rollout->createStrategy([
    'feature_name' => 'new_dashboard',
    'phases' => [
        ['percentage' => 5, 'segments' => ['beta_testers']],
        ['percentage' => 25, 'segments' => ['beta_testers', 'enterprise']],
        ['percentage' => 100, 'segments' => ['all']]
    ]
]);
```
