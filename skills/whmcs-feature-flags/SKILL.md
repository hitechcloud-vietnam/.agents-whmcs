# WHMCS Feature Flags Skill

## Purpose
Provides patterns for implementing feature flags in WHMCS, enabling gradual rollouts, A/B testing, and controlling feature availability per client or segment.

## Implementation Patterns

### Feature Flags Manager
```php
<?php
class FeatureFlags {
    private $db;
    private $cache;
    
    public function __construct() {
        $this->cache = \WHMCS\Application\Services\CacheService::getInstance();
    }
    
    public function isEnabled($flagName, $clientId = null) {
        $flag = $this->getFlag($flagName);
        
        if (!$flag || !$flag['is_active']) {
            return false;
        }
        
        // Check percentage rollout
        if ($flag['rollout_percentage'] < 100) {
            $hash = crc32($clientId ?? 'anonymous');
            $bucket = $hash % 100;
            
            if ($bucket >= $flag['rollout_percentage']) {
                return false;
            }
        }
        
        // Check client-specific override
        if ($clientId) {
            $override = $this->getClientOverride($clientId, $flagName);
            if ($override !== null) {
                return $override;
            }
        }
        
        // Check segment rules
        if ($flag['segment_rules']) {
            return $this->evaluateSegmentRules($flag['segment_rules'], $clientId);
        }
        
        return true;
    }
    
    private function getFlag($name) {
        $cacheKey = "feature_flag_{$name}";
        $cached = $this->cache->get($cacheKey);
        
        if ($cached) return $cached;
        
        $flag = $this->db->select(
            "SELECT * FROM mod_feature_flags WHERE name = ?",
            [$name]
        );
        
        $this->cache->set($cacheKey, $flag, 300);
        return $flag;
    }
    
    public function enableForClient($clientId, $flagName) {
        $this->db->insert('mod_feature_flag_overrides', [
            'client_id' => $clientId,
            'flag_name' => $flagName,
            'enabled' => 1
        ], true);
    }
    
    public function disableForClient($clientId, $flagName) {
        $this->db->insert('mod_feature_flag_overrides', [
            'client_id' => $clientId,
            'flag_name' => $flagName,
            'enabled' => 0
        ], true);
    }
    
    public function createFlag($data) {
        return $this->db->insert('mod_feature_flags', [
            'name' => $data['name'],
            'description' => $data['description'],
            'rollout_percentage' => $data['rollout_percentage'] ?? 0,
            'segment_rules' => json_encode($data['segment_rules'] ?? []),
            'is_active' => 1,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_feature_flags (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) UNIQUE,
    description TEXT,
    rollout_percentage INT DEFAULT 0,
    segment_rules JSON,
    is_active TINYINT(1) DEFAULT 1,
    created_at DATETIME
);

CREATE TABLE mod_feature_flag_overrides (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    flag_name VARCHAR(100),
    enabled TINYINT(1),
    INDEX idx_client_flag (client_id, flag_name)
);
```

## Usage Examples
```php
$flags = new FeatureFlags();

if ($flags->isEnabled('new_dashboard', $clientId)) {
    // Show new dashboard
} else {
    // Show legacy dashboard
}
```
