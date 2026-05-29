# WHMCS Cost Tracking Skill

## Purpose
Provides patterns for implementing cost tracking in WHMCS, monitoring expenses, analyzing cost drivers, and providing cost allocation insights.

## Implementation Patterns

### Cost Tracker
```php
<?php
class CostTracker {
    private $db;
    
    public function trackCost($category, $amount, $metadata = []) {
        $this->db->insert('mod_cost_entries', [
            'category' => $category,
            'amount' => $amount,
            'metadata' => json_encode($metadata),
            'recorded_at' => date('Y-m-d H:i:s')
        ]);
    }
    
    public function getCostByCategory($period = '30d') {
        return $this->db->select(
            "SELECT category, SUM(amount) as total, AVG(amount) as avg
             FROM mod_cost_entries
             WHERE recorded_at >= DATE_SUB(NOW(), INTERVAL ?)
             GROUP BY category
             ORDER BY total DESC",
            [$period]
        );
    }
    
    public function getCostTrends($period = '90d') {
        return $this->db->select(
            "SELECT DATE(recorded_at) as date, category, SUM(amount) as total
             FROM mod_cost_entries
             WHERE recorded_at >= DATE_SUB(NOW(), INTERVAL ?)
             GROUP BY DATE(recorded_at), category
             ORDER BY date ASC",
            [$period]
        );
    }
    
    public function allocateCosts($allocationKey) {
        $costs = $this->getCostByCategory('30d');
        $totalCost = array_sum(array_column($costs, 'total'));
        
        $clients = $this->db->select("SELECT id, revenue FROM tblclients WHERE status = 'Active'");
        $totalRevenue = array_sum(array_column($clients, 'revenue'));
        
        $allocations = [];
        foreach ($clients as $client) {
            $allocations[] = [
                'client_id' => $client['id'],
                'allocated_cost' => $totalCost * ($client['revenue'] / $totalRevenue)
            ];
        }
        
        return $allocations;
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_cost_entries (
    id INT AUTO_INCREMENT PRIMARY KEY,
    category VARCHAR(50),
    amount DECIMAL(10,2),
    metadata JSON,
    recorded_at DATETIME
);
```

## Usage Examples
```php
$tracker = new CostTracker();
$tracker->trackCost('infrastructure', 500, ['server' => 'web-01']);
$costs = $tracker->getCostByCategory('30d');
```
