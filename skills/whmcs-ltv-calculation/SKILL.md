# WHMCS LTV Calculation Skill

## Purpose
Provides patterns for implementing LTV (Lifetime Value) calculation in WHMCS, measuring customer value, predicting future revenue, and optimizing customer acquisition.

## Implementation Patterns

### LTV Calculator
```php
<?php
class LTVCalculator {
    private $db;
    
    public function calculateLTV($clientId) {
        $client = $this->getClient($clientId);
        
        // Calculate historical revenue
        $historicalRevenue = $this->getHistoricalRevenue($clientId);
        
        // Calculate average monthly revenue
        $avgMonthlyRevenue = $this->getAverageMonthlyRevenue($clientId);
        
        // Calculate customer lifespan
        $lifespanMonths = $this->calculateLifespan($clientId);
        
        // Calculate predicted future revenue
        $predictedRevenue = $avgMonthlyRevenue * $lifespanMonths;
        
        // Calculate LTV
        $ltv = $historicalRevenue + $predictedRevenue;
        
        return [
            'client_id' => $clientId,
            'historical_revenue' => $historicalRevenue,
            'avg_monthly_revenue' => $avgMonthlyRevenue,
            'lifespan_months' => $lifespanMonths,
            'predicted_future_revenue' => $predictedRevenue,
            'ltv' => $ltv,
            'ltv_tier' => $this->getLTVTier($ltv)
        ];
    }
    
    private function getHistoricalRevenue($clientId) {
        $result = $this->db->select(
            "SELECT SUM(amount) as total FROM tblaccounts WHERE userid = ?",
            [$clientId]
        );
        
        return $result['total'] ?? 0;
    }
    
    private function getAverageMonthlyRevenue($clientId) {
        $result = $this->db->select(
            "SELECT SUM(monthly_amount) as total FROM tblhosting
             WHERE userid = ? AND domainstatus = 'Active'",
            [$clientId]
        );
        
        return $result['total'] ?? 0;
    }
    
    private function calculateLifespan($clientId) {
        $client = $this->getClient($clientId);
        $monthsActive = (time() - strtotime($client['regdate'])) / (30 * 86400);
        
        // Use historical lifespan if established
        if ($monthsActive > 12) {
            // Project based on historical engagement
            $churnRate = $this->getChurnRate($clientId);
            $expectedLifespan = 1 / max(0.01, $churnRate);
            return min($expectedLifespan, 60); // Cap at 5 years
        }
        
        return $monthsActive * 2; // Conservative estimate for new customers
    }
    
    private function getChurnRate($clientId) {
        $totalServices = $this->db->select(
            "SELECT COUNT(*) as count FROM tblhosting WHERE userid = ?",
            [$clientId]
        )['count'];
        
        $churnedServices = $this->db->select(
            "SELECT COUNT(*) as count FROM tblhosting 
             WHERE userid = ? AND domainstatus = 'Terminated'",
            [$clientId]
        )['count'];
        
        return $totalServices > 0 ? $churnedServices / $totalServices : 0.1;
    }
    
    private function getLTVTier($ltv) {
        if ($ltv >= 50000) return 'platinum';
        if ($ltv >= 10000) return 'gold';
        if ($ltv >= 1000) return 'silver';
        return 'bronze';
    }
    
    public function getAverageLTV($period = '90d') {
        $clients = $this->db->select(
            "SELECT id FROM tblclients WHERE status = 'Active'"
        );
        
        $totalLTV = 0;
        foreach ($clients as $client) {
            $ltvData = $this->calculateLTV($client['id']);
            $totalLTV += $ltvData['ltv'];
        }
        
        return count($clients) > 0 ? $totalLTV / count($clients) : 0;
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_ltv_calculations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    historical_revenue DECIMAL(10,2),
    avg_monthly_revenue DECIMAL(10,2),
    lifespan_months DECIMAL(5,2),
    predicted_revenue DECIMAL(10,2),
    ltv DECIMAL(10,2),
    ltv_tier VARCHAR(20),
    calculated_at DATETIME
);
```

## Usage Examples
```php
$calculator = new LTVCalculator();
$ltv = $calculator->calculateLTV($clientId);

echo "Customer LTV: $" . number_format($ltv['ltv'], 2);
echo "Tier: " . $ltv['ltv_tier'];
```
