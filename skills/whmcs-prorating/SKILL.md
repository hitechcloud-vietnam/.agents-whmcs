# WHMCS Prorating

## Overview
Master skill for prorated billing calculations in WHMCS. Covers upgrade/downgrade prorations, partial period calculations, and billing adjustments.

## Prorate Calculator

```php
<?php
// /includes/helpers/prorate_helper.php

class ProrateCalculator
{
    private $daysInMonth = 30;
    
    public function calculateUpgradeProrate(
        float $oldPrice,
        float $newPrice,
        string $billingCycle,
        \DateTime $nextDueDate
    ): float {
        $daysRemaining = $this->getDaysRemaining($nextDueDate);
        $daysInPeriod = $this->getDaysInPeriod($billingCycle);
        
        $dailyOldRate = $oldPrice / $daysInPeriod;
        $dailyNewRate = $newPrice / $daysInPeriod;
        
        $credit = $dailyOldRate * $daysRemaining;
        $charge = $dailyNewRate * $daysRemaining;
        
        return round($charge - $credit, 2);
    }
    
    public function calculateDowngradeProrate(
        float $oldPrice,
        float $newPrice,
        string $billingCycle,
        \DateTime $nextDueDate
    ): float {
        $daysRemaining = $this->getDaysRemaining($nextDueDate);
        $daysInPeriod = $this->getDaysInPeriod($billingCycle);
        
        $dailyOldRate = $oldPrice / $daysInPeriod;
        $dailyNewRate = $newPrice / $daysInPeriod;
        
        $credit = $dailyOldRate * $daysRemaining;
        $charge = $dailyNewRate * $daysRemaining;
        
        return round($credit - $charge, 2);
    }
    
    public function calculateCancellationRefund(
        float $price,
        string $billingCycle,
        \DateTime $nextDueDate
    ): float {
        $daysRemaining = $this->getDaysRemaining($nextDueDate);
        $daysInPeriod = $this->getDaysInPeriod($billingCycle);
        
        $dailyRate = $price / $daysInPeriod;
        
        return round($dailyRate * $daysRemaining, 2);
    }
    
    private function getDaysRemaining(\DateTime $nextDueDate): int
    {
        $today = new \DateTime();
        $diff = $today->diff($nextDueDate);
        
        return max(0, $diff->days);
    }
    
    private function getDaysInPeriod(string $billingCycle): int
    {
        $cycles = [
            "Monthly" => 30,
            "Quarterly" => 90,
            "Semi-Annually" => 180,
            "Annually" => 365,
            "Biennially" => 730,
        ];
        
        return $cycles[$billingCycle] ?? 30;
    }
    
    public function getProrateAmount(
        float $currentAmount,
        float $newAmount,
        \DateTime $periodStart,
        \DateTime $periodEnd,
        \DateTime $changeDate
    ): float {
        $totalDays = $periodStart->diff($periodEnd)->days;
        $usedDays = $periodStart->diff($changeDate)->days;
        $remainingDays = $totalDays - $usedDays;
        
        $dailyCurrent = $currentAmount / $totalDays;
        $dailyNew = $newAmount / $totalDays;
        
        $credit = $dailyCurrent * $remainingDays;
        $charge = $dailyNew * $remainingDays;
        
        return round($charge - $credit, 2);
    }
}
```

## Best Practices

1. **Fair Calculations**: Always use accurate daily rates
2. **Prorate Direction**: Calculate credit vs charge correctly
3. **Rounding**: Round to 2 decimal places consistently
4. **Transparency**: Show prorate breakdown to customers
5. **Billing Cycle**: Consider full billing cycle lengths
6. **Immediate vs Scheduled**: Decide when to apply changes
7. **Documentation**: Record all prorate calculations
8. **Testing**: Verify calculations with edge cases
