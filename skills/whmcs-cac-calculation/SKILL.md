# WHMCS CAC Calculation Skill

## Purpose
Provides patterns for implementing CAC (Customer Acquisition Cost) calculation in WHMCS, tracking marketing spend, measuring acquisition efficiency, and optimizing marketing ROI.

## Implementation Patterns

### CAC Calculator
```php
<?php
class CACCalculator {
    private $db;
    
    public function calculateCAC($period = '30d') {
        // Total marketing spend
        $marketingSpend = $this->getMarketingSpend($period);
        
        // Number of new customers
        $newCustomers = $this->getNewCustomerCount($period);
        
        $cac = $newCustomers > 0 ? $marketingSpend / $newCustomers : 0;
        
        return [
            'period' => $period,
            'marketing_spend' => $marketingSpend,
            'new_customers' => $newCustomers,
            'cac' => $cac,
            'cac_tier' => $this->getCACTier($cac)
        ];
    }
    
    private function getMarketingSpend($period) {
        $spend = $this->db->select(
            "SELECT SUM(amount) as total FROM mod_marketing_spend
             WHERE spent_at >= DATE_SUB(NOW(), INTERVAL ?)",
            [$period]
        );
        
        return $spend['total'] ?? 0;
    }
    
    private function getNewCustomerCount($period) {
        $result = $this->db->select(
            "SELECT COUNT(*) as count FROM tblclients
             WHERE regdate >= DATE_SUB(NOW(), INTERVAL ?)",
            [$period]
        );
        
        return $result['count'] ?? 0;
    }
    
    private function getCACTier($cac) {
        if ($cac <= 50) return 'excellent';
        if ($cac <= 150) return 'good';
        if ($cac <= 300) return 'fair';
        return 'poor';
    }
    
    public function calculateBlendedCAC() {
        $channelCACs = [];
        
        $channels = ['google_ads', 'facebook_ads', 'referral', 'organic'];
        
        foreach ($channels as $channel) {
            $channelCACs[$channel] = $this->calculateChannelCAC($channel);
        }
        
        $totalSpend = array_sum(array_column($channelCACs, 'spend'));
        $totalCustomers = array_sum(array_column($channelCACs, 'customers'));
        
        return [
            'channels' => $channelCACs,
            'blended_cac' => $totalCustomers > 0 ? $totalSpend / $totalCustomers : 0
        ];
    }
    
    private function calculateChannelCAC($channel) {
        $spend = $this->db->select(
            "SELECT SUM(amount) as total FROM mod_marketing_spend
             WHERE channel = ? AND spent_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)",
            [$channel]
        )['total'] ?? 0;
        
        $customers = $this->db->select(
            "SELECT COUNT(*) as count FROM mod_customer_acquisitions
             WHERE channel = ? AND acquired_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)",
            [$channel]
        )['count'] ?? 0;
        
        return [
            'channel' => $channel,
            'spend' => $spend,
            'customers' => $customers,
            'cac' => $customers > 0 ? $spend / $customers : 0
        ];
    }
    
    public function calculateLTVtoCACRatio($clientId) {
        $ltvCalculator = new LTVCalculator();
        $ltv = $ltvCalculator->calculateLTV($clientId);
        
        $avgCAC = $this->calculateCAC()['cac'];
        
        return [
            'ltv' => $ltv['ltv'],
            'cac' => $avgCAC,
            'ratio' => $avgCAC > 0 ? $ltv['ltv'] / $avgCAC : 0
        ];
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_marketing_spend (
    id INT AUTO_INCREMENT PRIMARY KEY,
    channel VARCHAR(50),
    campaign VARCHAR(100),
    amount DECIMAL(10,2),
    spent_at DATETIME
);

CREATE TABLE mod_customer_acquisitions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    channel VARCHAR(50),
    campaign VARCHAR(100),
    cost DECIMAL(10,2),
    acquired_at DATETIME
);
```

## Usage Examples
```php
$cac = new CACCalculator();
$metrics = $cac->calculateCAC('30d');
echo "CAC: $" . number_format($metrics['cac'], 2);

$blended = $cac->calculateBlendedCAC();
echo "Blended CAC: $" . number_format($blended['blended_cac'], 2);
```
