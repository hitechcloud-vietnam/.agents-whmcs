# WHMCS Burn Rate Skill

## Purpose
Provides patterns for implementing burn rate tracking in WHMCS SaaS businesses, measuring cash consumption, tracking runway, and forecasting financial sustainability.

## Implementation Patterns

### Burn Rate Tracker
```php
<?php
class BurnRateTracker {
    private $db;
    
    public function calculateBurnRate($period = '30d') {
        $expenses = $this->getExpenses($period);
        $revenue = $this->getRevenue($period);
        
        $grossBurn = $expenses['total'];
        $netBurn = $grossBurn - $revenue;
        
        return [
            'period' => $period,
            'gross_burn' => $grossBurn,
            'revenue' => $revenue,
            'net_burn' => $netBurn,
            'daily_burn' => $netBurn / $this->getDaysInPeriod($period),
            'monthly_burn' => $netBurn
        ];
    }
    
    private function getExpenses($period) {
        $expenses = $this->db->select(
            "SELECT SUM(amount) as total FROM mod_expenses
             WHERE incurred_at >= DATE_SUB(NOW(), INTERVAL ?)",
            [$period]
        );
        
        $breakdown = $this->db->select(
            "SELECT category, SUM(amount) as total FROM mod_expenses
             WHERE incurred_at >= DATE_SUB(NOW(), INTERVAL ?)
             GROUP BY category",
            [$period]
        );
        
        return [
            'total' => $expenses['total'] ?? 0,
            'breakdown' => $breakdown
        ];
    }
    
    private function getRevenue($period) {
        $result = $this->db->select(
            "SELECT SUM(amount) as total FROM tblaccounts
             WHERE date >= DATE_SUB(NOW(), INTERVAL ?)",
            [$period]
        );
        
        return $result['total'] ?? 0;
    }
    
    private function getDaysInPeriod($period) {
        $daysMap = ['7d' => 7, '30d' => 30, '90d' => 90, '365d' => 365];
        return $daysMap[$period] ?? 30;
    }
    
    public function calculateRunway($cashBalance = null) {
        if (!$cashBalance) {
            $cashBalance = $this->getCashBalance();
        }
        
        $burnRate = $this->calculateBurnRate()['monthly_burn'];
        
        return [
            'cash_balance' => $cashBalance,
            'monthly_burn' => abs($burnRate),
            'runway_months' => $burnRate > 0 ? $cashBalance / abs($burnRate) : 999,
            'runway_date' => $burnRate > 0 ? date('Y-m-d', strtotime("+" . floor($cashBalance / abs($burnRate)) . " months")) : null
        ];
    }
    
    private function getCashBalance() {
        // Get from accounting system or manual entry
        $result = $this->db->select(
            "SELECT value FROM mod_financial_config WHERE key = 'cash_balance'"
        );
        
        return $result['value'] ?? 0;
    }
    
    public function getBurnRateTrends($period = '180d') {
        return $this->db->select(
            "SELECT DATE(recorded_at) as date, net_burn, gross_burn
             FROM mod_burn_rate_snapshots
             WHERE recorded_at >= DATE_SUB(NOW(), INTERVAL ?)
             ORDER BY date ASC",
            [$period]
        );
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_expenses (
    id INT AUTO_INCREMENT PRIMARY KEY,
    category VARCHAR(50),
    description VARCHAR(255),
    amount DECIMAL(10,2),
    incurred_at DATETIME
);

CREATE TABLE mod_burn_rate_snapshots (
    id INT AUTO_INCREMENT PRIMARY KEY,
    net_burn DECIMAL(10,2),
    gross_burn DECIMAL(10,2),
    revenue DECIMAL(10,2),
    recorded_at DATETIME
);
```

## Usage Examples
```php
$tracker = new BurnRateTracker();
$burnRate = $tracker->calculateBurnRate('30d');
echo "Monthly Burn: $" . number_format($burnRate['monthly_burn'], 2);

$runway = $tracker->calculateRunway(100000);
echo "Runway: " . round($runway['runway_months']) . " months";
```
