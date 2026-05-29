# WHMCS Retention Analysis Skill

## Purpose
Provides patterns for implementing retention analysis in WHMCS, measuring customer retention rates, identifying churn patterns, and tracking retention KPIs.

## Implementation Patterns

### Retention Analyzer
```php
<?php
class RetentionAnalyzer {
    private $db;
    
    public function calculateRetentionRate($period = '30d') {
        $startCustomers = $this->getStartCustomerCount($period);
        $endCustomers = $this->getEndCustomerCount($period);
        $newCustomers = $this->getNewCustomerCount($period);
        
        $retained = $endCustomers - $newCustomers;
        $retentionRate = $startCustomers > 0 ? ($retained / $startCustomers) * 100 : 0;
        
        return [
            'period' => $period,
            'start_customers' => $startCustomers,
            'end_customers' => $endCustomers,
            'new_customers' => $newCustomers,
            'retained' => $retained,
            'retention_rate' => $retentionRate,
            'churn_rate' => 100 - $retentionRate
        ];
    }
    
    private function getStartCustomerCount($period) {
        $result = $this->db->select(
            "SELECT COUNT(DISTINCT userid) as count FROM tblhosting
             WHERE created_at < DATE_SUB(NOW(), INTERVAL ?)",
            [$period]
        );
        
        return $result['count'] ?? 0;
    }
    
    private function getEndCustomerCount($period) {
        $result = $this->db->select(
            "SELECT COUNT(DISTINCT userid) as count FROM tblhosting
             WHERE domainstatus = 'Active' AND created_at <= NOW()"
        );
        
        return $result['count'] ?? 0;
    }
    
    private function getNewCustomerCount($period) {
        $result = $this->db->select(
            "SELECT COUNT(DISTINCT userid) as count FROM tblhosting
             WHERE created_at >= DATE_SUB(NOW(), INTERVAL ?)",
            [$period]
        );
        
        return $result['count'] ?? 0;
    }
    
    public function getRetentionTrends($period = '180d') {
        return $this->db->select(
            "SELECT DATE_FORMAT(recorded_at, '%Y-%m') as month,
                    retention_rate, churn_rate, active_customers
             FROM mod_retention_snapshots
             WHERE recorded_at >= DATE_SUB(NOW(), INTERVAL ?)
             ORDER BY month ASC",
            [$period]
        );
    }
    
    public function getRetentionBySegment($segmentField) {
        $segments = $this->db->select(
            "SELECT {$segmentField}, COUNT(*) as total,
                    COUNT(CASE WHEN domainstatus = 'Active' THEN 1 END) as active
             FROM tblclients c
             JOIN tblhosting h ON h.userid = c.id
             GROUP BY {$segmentField}"
        );
        
        $results = [];
        foreach ($segments as $segment) {
            $results[$segment[$segmentField]] = [
                'total' => $segment['total'],
                'active' => $segment['active'],
                'retention_rate' => $segment['total'] > 0 ? ($segment['active'] / $segment['total']) * 100 : 0
            ];
        }
        
        return $results;
    }
    
    public function getRetentionFactors($clientId) {
        $factors = [];
        
        // Contract length
        $contractLength = $this->db->select(
            "SELECT MAX(months) as max_months FROM tblhosting WHERE userid = ?",
            [$clientId]
        )['max_months'] ?? 0;
        $factors['long_term_contract'] = $contractLength >= 12;
        
        // Payment history
        $latePayments = $this->db->select(
            "SELECT COUNT(*) as count FROM tblinvoices
             WHERE userid = ? AND duedate < datepaid AND datepaid IS NOT NULL",
            [$clientId]
        )['count'];
        $factors['payment_issues'] = $latePayments > 0;
        
        // Engagement
        $lastLogin = $this->db->select(
            "SELECT lastlogin FROM tblclients WHERE id = ?",
            [$clientId]
        )['lastlogin'];
        $factors['recent_login'] = (time() - strtotime($lastLogin)) < 86400 * 7;
        
        return $factors;
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_retention_snapshots (
    id INT AUTO_INCREMENT PRIMARY KEY,
    period VARCHAR(20),
    start_customers INT,
    end_customers INT,
    new_customers INT,
    active_customers INT,
    retention_rate DECIMAL(5,2),
    churn_rate DECIMAL(5,2),
    recorded_at DATETIME
);
```

## Usage Examples
```php
$analyzer = new RetentionAnalyzer();
$retention = $analyzer->calculateRetentionRate('30d');

echo "Retention Rate: " . round($retention['retention_rate'], 1) . "%";
echo "Churn Rate: " . round($retention['churn_rate'], 1) . "%";

$byPlan = $analyzer->getRetentionBySegment('client_group');
```
