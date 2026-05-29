# WHMCS ARR Calculation Skill

## Purpose
Provides patterns for implementing ARR (Annual Recurring Revenue) calculation in WHMCS, measuring annualized revenue, tracking annual contract values, and forecasting annual revenue.

## Implementation Patterns

### ARR Calculator
```php
<?php
class ARRCalculator {
    private $db;
    
    public function calculateARR() {
        $mrr = $this->getMRR();
        
        return [
            'arr' => $mrr * 12,
            'mrr' => $mrr,
            'calculated_at' => date('Y-m-d H:i:s')
        ];
    }
    
    private function getMRR() {
        $result = $this->db->select(
            "SELECT SUM(monthly_amount) as total FROM tblhosting
             WHERE domainstatus = 'Active'"
        );
        
        return $result['total'] ?? 0;
    }
    
    public function calculateContractARR($contractId) {
        $contract = $this->db->select(
            "SELECT * FROM mod_contracts WHERE id = ?",
            [$contractId]
        );
        
        $annualValue = $contract['value'];
        
        return [
            'contract_id' => $contractId,
            'contract_value' => $contract['value'],
            'arr' => $contract['billing_cycle'] === 'Monthly' ? $contract['value'] * 12 : $contract['value'],
            'remaining_value' => $this->calculateRemainingValue($contract)
        ];
    }
    
    private function calculateRemainingValue($contract) {
        $monthsRemaining = $this->getMonthsRemaining($contract['expiration_date']);
        
        return $contract['value'] * $monthsRemaining;
    }
    
    private function getMonthsRemaining($expirationDate) {
        $now = time();
        $expiry = strtotime($expirationDate);
        
        if ($expiry <= $now) return 0;
        
        return ceil(($expiry - $now) / (30 * 86400));
    }
    
    public function getARRForecast($months = 12) {
        $currentARR = $this->calculateARR()['arr'];
        
        $growthRate = $this->getAverageGrowthRate();
        
        $forecast = [];
        for ($i = 1; $i <= $months; $i++) {
            $forecast[] = [
                'month' => $i,
                'arr' => $currentARR * pow(1 + $growthRate, $i),
                'growth_rate' => $growthRate
            ];
        }
        
        return $forecast;
    }
    
    private function getAverageGrowthRate() {
        $previousMonth = $this->db->select(
            "SELECT SUM(monthly_amount) as total FROM tblhosting
             WHERE domainstatus = 'Active' AND created_at < DATE_SUB(NOW(), INTERVAL 30 DAY)"
        )['total'] ?? 0;
        
        $currentMonth = $this->getMRR();
        
        return $previousMonth > 0 ? ($currentMonth - $previousMonth) / $previousMonth : 0;
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_arr_snapshots (
    id INT AUTO_INCREMENT PRIMARY KEY,
    snapshot_date DATE,
    arr DECIMAL(15,2),
    mrr DECIMAL(10,2),
    recorded_at DATETIME
);
```

## Usage Examples
```php
$arrCalc = new ARRCalculator();
$arr = $arrCalc->calculateARR();
echo "ARR: $" . number_format($arr['arr'], 2);

$forecast = $arrCalc->getARRForecast(12);
```
