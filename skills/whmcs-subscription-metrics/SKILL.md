# WHMCS Subscription Metrics Skill

## Purpose
Provides patterns for implementing subscription metrics tracking in WHMCS, monitoring subscription health, tracking plan changes, and analyzing subscription trends.

## Implementation Patterns

### Subscription Metrics
```php
<?php
class SubscriptionMetrics {
    private $db;
    
    public function getSubscriptionHealth($clientId) {
        $subscriptions = $this->db->select(
            "SELECT * FROM tblhosting WHERE userid = ? AND domainstatus = 'Active'",
            [$clientId]
        );
        
        $health = [
            'total_subscriptions' => count($subscriptions),
            'total_monthly_value' => array_sum(array_column($subscriptions, 'billingcycle') === 'Monthly' ? 
                array_column($subscriptions, 'amount') : array_column($subscriptions, 'amount') / 12),
            'average_age' => $this->calculateAverageAge($subscriptions),
            'renewal_rate' => $this->calculateRenewalRate($clientId)
        ];
        
        return $health;
    }
    
    public function trackPlanChange($clientId, $fromPlan, $toPlan) {
        $this->db->insert('mod_plan_changes', [
            'client_id' => $clientId,
            'from_plan' => $fromPlan,
            'to_plan' => $toPlan,
            'changed_at' => date('Y-m-d H:i:s')
        ]);
    }
    
    public function getSubscriptionTrends($period = '90d') {
        return $this->db->select(
            "SELECT DATE(created_at) as date, COUNT(*) as new_subs,
                    SUM(monthly_amount) as mrr_added
             FROM tblhosting
             WHERE created_at >= DATE_SUB(NOW(), INTERVAL ?)
             GROUP BY DATE(created_at)",
            [$period]
        );
    }
    
    private function calculateRenewalRate($clientId) {
        $total = $this->db->select(
            "SELECT COUNT(*) as count FROM tblhosting WHERE userid = ?",
            [$clientId]
        )['count'];
        
        $renewed = $this->db->select(
            "SELECT COUNT(*) as count FROM tblhosting 
             WHERE userid = ? AND next_invoice_date <= CURDATE() + INTERVAL 30 DAY",
            [$clientId]
        )['count'];
        
        return $total > 0 ? ($renewed / $total) * 100 : 0;
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_plan_changes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT,
    from_plan VARCHAR(100),
    to_plan VARCHAR(100),
    changed_at DATETIME
);
```

## Usage Examples
```php
$metrics = new SubscriptionMetrics();
$health = $metrics->getSubscriptionHealth($clientId);
$trends = $metrics->getSubscriptionTrends('90d');
```
