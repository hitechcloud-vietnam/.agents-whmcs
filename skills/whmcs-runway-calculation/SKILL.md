# WHMCS Runway Calculation Skill

## Purpose
Provides patterns for implementing runway analysis in WHMCS SaaS businesses, calculating financial runway, forecasting cash needs, and planning for sustainability.

## Implementation Patterns

### Runway Calculator
```php
<?php
class RunwayCalculator {
    private $db;
    
    public function calculateRunway($cashBalance = null, $burnRate = null) {
        $cashBalance = $cashBalance ?? $this->getCashBalance();
        $burnRate = $burnRate ?? $this->getBurnRate()['monthly'];
        
        $runwayMonths = $burnRate > 0 ? $cashBalance / abs($burnRate) : 999;
        
        return [
            'cash_balance' => $cashBalance,
            'monthly_burn' => abs($burnRate),
            'runway_months' => $runwayMonths,
            'runway_weeks' => $runwayMonths * 4.33,
            'runway_date' => $burnRate > 0 ? date('Y-m-d', strtotime("+{$runwayMonths} months")) : null,
            'runway_end' => $burnRate > 0 ? date('Y-m-d', strtotime("+" . floor($runwayMonths) . " months")) : null
        ];
    }
    
    private function getCashBalance() {
        $result = $this->db->select(
            "SELECT SUM(total) as balance FROM tblaccounts"
        );
        
        return $result['balance'] ?? 0;
    }
    
    private function getBurnRate() {
        $expenses = $this->db->select(
            "SELECT SUM(amount) as total FROM mod_expenses
             WHERE incurred_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)"
        )['total'] ?? 0;
        
        $revenue = $this->db->select(
            "SELECT SUM(amount) as total FROM tblaccounts
             WHERE date >= DATE_SUB(NOW(), INTERVAL 30 DAY)"
        )['total'] ?? 0;
        
        return [
            'monthly' => $expenses - $revenue
        ];
    }
    
    public function getRunwayScenario($cashBalance, $scenarios) {
        $results = [];
        
        foreach ($scenarios as $name => $burnRate) {
            $runway = $cashBalance / abs($burnRate);
            
            $results[$name] = [
                'monthly_burn' => $burnRate,
                'runway_months' => $runway,
                'runway_date' => date('Y-m-d', strtotime("+{$runway} months")),
                'months_remaining' => floor($runway)
            ];
        }
        
        return $results;
    }
    
    public function getCashNeedsProjection($months = 12) {
        $currentBurn = $this->getBurnRate()['monthly'];
        $projections = [];
        
        for ($i = 1; $i <= $months; $i++) {
            $projections[] = [
                'month' => $i,
                'date' => date('Y-m-01', strtotime("+{$i} months")),
                'projected_cash' => $this->getCashBalance() - ($currentBurn * $i),
                'burn_rate' => $currentBurn
            ];
        }
        
        return $projections;
    }
}
```

## Usage Examples
```php
$runway = new RunwayCalculator();
$analysis = $runway->calculateRunway();

echo "Cash: $" . number_format($analysis['cash_balance']);
echo "Runway: " . round($analysis['runway_months']) . " months";
echo "Runway Date: " . $analysis['runway_date'];
```
