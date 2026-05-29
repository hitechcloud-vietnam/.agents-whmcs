# WHMCS Quota Management Skill

## Purpose
Provides patterns for implementing quota management in WHMCS SaaS applications, setting usage limits, tracking consumption, and enforcing resource quotas.

## Implementation Patterns

### Quota Manager
```php
<?php
class QuotaManager {
    private $db;
    
    public function checkQuota($clientId, $resourceType, $requestedAmount = 1) {
        $quota = $this->getClientQuota($clientId, $resourceType);
        $currentUsage = $this->getCurrentUsage($clientId, $resourceType);
        $available = $quota['limit'] - $currentUsage;
        
        return [
            'allowed' => $available >= $requestedAmount,
            'limit' => $quota['limit'],
            'used' => $currentUsage,
            'available' => $available,
            'requested' => $requestedAmount,
            'overage_allowed' => $quota['allow_overage']
        ];
    }
    
    public function consumeQuota($clientId, $resourceType, $amount) {
        $this->db->insert('mod_quota_usage', [
            'client_id' => $clientId,
            'resource_type' => $resourceType,
            'amount' => $amount,
            'consumed_at' => date('Y-m-d H:i:s')
        ]);
    }
    
    public function resetQuotas($clientId, $period = 'monthly') {
        $periodStart = date('Y-m-01');
        
        $this->db->where('client_id', $clientId)
            ->where('period_start', $periodStart)
            ->update('mod_quota_limits', ['reset' => 1]);
    }
    
    public function getQuotas($clientId) {
        return $this->db->select(
            "SELECT q.*, u.total as usage, (q.limit - u.total) as available
             FROM mod_quota_limits q
             LEFT JOIN (
                 SELECT client_id, resource_type, SUM(amount) as total
                 FROM mod_quota_usage
                 WHERE period_start = ?
                 GROUP BY client_id, resource_type
             ) u ON u.client_id = q.client_id AND u.resource_type = q.resource_type
             WHERE q.client_id = ?",
            [date('Y-m-01'), $clientId]
        );
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_quota_limits (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    resource_type VARCHAR(50),
    limit_amount DECIMAL(15,4),
    period_start DATE,
    allow_overage TINYINT(1) DEFAULT 0,
    overage_rate DECIMAL(10,2)
);

CREATE TABLE mod_quota_usage (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    resource_type VARCHAR(50),
    amount DECIMAL(15,4),
    consumed_at DATETIME
);
```
