# WHMCS Usage Tracking Skill

## Purpose
Provides patterns for implementing usage tracking in WHMCS SaaS applications, monitoring resource consumption, tracking API usage, and calculating usage-based billing.

## Implementation Patterns

### Usage Tracker
```php
<?php
class UsageTracker {
    private $db;
    
    public function recordUsage($clientId, $resource, $amount, $metadata = []) {
        $record = [
            'client_id' => $clientId,
            'resource' => $resource,
            'amount' => $amount,
            'metadata' => json_encode($metadata),
            'recorded_at' => date('Y-m-d H:i:s')
        ];
        
        $this->db->insert('mod_usage_records', $record);
        
        // Update client usage summary
        $this->updateUsageSummary($clientId, $resource);
        
        // Check quotas
        $this->checkQuotaLimits($clientId, $resource);
    }
    
    private function updateUsageSummary($clientId, $resource) {
        $today = date('Y-m-d');
        $monthStart = date('Y-m-01');
        
        $existing = $this->db->select(
            "SELECT * FROM mod_usage_summary 
             WHERE client_id = ? AND resource = ? AND period_start = ?",
            [$clientId, $resource, $monthStart]
        );
        
        if ($existing) {
            $this->db->where('client_id', $clientId)->where('resource', $resource)
                ->where('period_start', $monthStart)->update('mod_usage_summary', [
                    'total_amount' => $existing['total_amount'] + $amount,
                    'record_count' => $existing['record_count'] + 1
                ]);
        } else {
            $this->db->insert('mod_usage_summary', [
                'client_id' => $clientId,
                'resource' => $resource,
                'period_start' => $monthStart,
                'total_amount' => $amount,
                'record_count' => 1
            ]);
        }
    }
    
    public function getClientUsage($clientId, $resource = null, $period = 'monthly') {
        $query = "SELECT resource, SUM(amount) as total FROM mod_usage_records 
                  WHERE client_id = ? AND recorded_at >= DATE_SUB(NOW(), INTERVAL ?";
        $params = [$clientId, $period];
        
        if ($resource) {
            $query .= " AND resource = ?";
            $params[] = $resource;
        }
        
        $query .= " GROUP BY resource";
        
        return $this->db->select($query, $params);
    }
    
    public function getResourceBreakdown($clientId, $resource) {
        return $this->db->select(
            "SELECT metadata, SUM(amount) as amount, COUNT(*) as count
             FROM mod_usage_records
             WHERE client_id = ? AND resource = ? AND recorded_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
             GROUP BY metadata",
            [$clientId, $resource]
        );
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_usage_records (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    resource VARCHAR(50),
    amount DECIMAL(15,4),
    metadata JSON,
    recorded_at DATETIME,
    INDEX idx_client_resource (client_id, resource)
);

CREATE TABLE mod_usage_summary (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    resource VARCHAR(50),
    period_start DATE,
    total_amount DECIMAL(15,4),
    record_count INT,
    UNIQUE KEY idx_client_resource_period (client_id, resource, period_start)
);
```

## Usage Examples
```php
$tracker = new UsageTracker();
$tracker->recordUsage($clientId, 'api_calls', 100, ['endpoint' => '/api/users']);

$usage = $tracker->getClientUsage($clientId, 'api_calls');
```
