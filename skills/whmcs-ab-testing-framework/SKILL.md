# WHMCS A/B Testing Framework Skill

## Purpose
Provides patterns for implementing A/B testing in WHMCS, creating experiments, assigning variants, tracking conversions, and analyzing test results.

## Implementation Patterns

### A/B Testing Framework
```php
<?php
class ABTestingFramework {
    private $db;
    
    public function createExperiment($data) {
        $experiment = [
            'name' => $data['name'],
            'description' => $data['description'],
            'variants' => json_encode($data['variants']),
            'traffic_split' => json_encode($data['traffic_split']),
            'goal_metric' => $data['goal_metric'],
            'status' => 'draft',
            'start_date' => $data['start_date'],
            'end_date' => $data['end_date'],
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        return $this->db->insert('mod_ab_experiments', $experiment);
    }
    
    public function getVariant($experimentId, $clientId = null) {
        $experiment = $this->getExperiment($experimentId);
        
        if ($experiment['status'] !== 'active') {
            return null;
        }
        
        // Check for existing assignment
        $assignment = $this->getAssignment($experimentId, $clientId);
        if ($assignment) {
            return $assignment['variant'];
        }
        
        // Assign new variant based on traffic split
        $hash = crc32(($clientId ?? 'anon') . $experimentId);
        $bucket = $hash % 100;
        
        $cumulative = 0;
        $variants = json_decode($experiment['traffic_split'], true);
        
        foreach ($variants as $variant => $percentage) {
            $cumulative += $percentage;
            if ($bucket < $cumulative) {
                $this->assignVariant($experimentId, $clientId, $variant);
                return $variant;
            }
        }
        
        return array_key_first($variants);
    }
    
    public function trackConversion($experimentId, $clientId, $value = 1) {
        $assignment = $this->getAssignment($experimentId, $clientId);
        
        if (!$assignment) return false;
        
        $this->db->insert('mod_ab_conversions', [
            'experiment_id' => $experimentId,
            'variant' => $assignment['variant'],
            'client_id' => $clientId,
            'value' => $value,
            'converted_at' => date('Y-m-d H:i:s')
        ]);
        
        return true;
    }
    
    public function getResults($experimentId) {
        $conversions = $this->db->select(
            "SELECT variant, COUNT(*) as impressions, 
                    SUM(value) as conversions, AVG(value) as avg_value
             FROM mod_ab_conversions
             WHERE experiment_id = ? AND converted_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
             GROUP BY variant",
            [$experimentId]
        );
        
        $results = [];
        foreach ($conversions as $cv) {
            $results[$cv['variant']] = [
                'impressions' => $cv['impressions'],
                'conversions' => $cv['conversions'],
                'conversion_rate' => $cv['impressions'] > 0 ? ($cv['conversions'] / $cv['impressions']) * 100 : 0
            ];
        }
        
        return $results;
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_ab_experiments (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255),
    description TEXT,
    variants JSON,
    traffic_split JSON,
    goal_metric VARCHAR(50),
    status ENUM('draft', 'active', 'paused', 'completed') DEFAULT 'draft',
    start_date DATE,
    end_date DATE,
    created_at DATETIME
);

CREATE TABLE mod_ab_assignments (
    id INT AUTO_INCREMENT PRIMARY KEY,
    experiment_id INT,
    client_id INT,
    variant VARCHAR(50),
    assigned_at DATETIME
);

CREATE TABLE mod_ab_conversions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    experiment_id INT,
    variant VARCHAR(50),
    client_id INT,
    value DECIMAL(10,2),
    converted_at DATETIME
);
```
