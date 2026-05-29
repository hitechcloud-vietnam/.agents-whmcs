# WHMCS Early Termination

## Concept Explanation
Early termination fees compensate for revenue loss when customers cancel before contract end.

### Termination Fee Calculation
- **Prorated Refund**: Unused portion minus fee
- **Contract Penalty**: Percentage of remaining value
- **Admin Fee**: Processing fee only

## Code Patterns

```php
<?php
class TerminationFeeCalculator {
    
    const EARLY_TERMINATION_PENALTY = 0.25; // 25% of remaining value
    const MINIMUM_FEE = 49.99;
    
    public static function calculateTerminationFee($serviceId) {
        $service = getService($serviceId);
        
        if ($service['billingcycle'] === 'Monthly') return 0;
        
        $monthsRemaining = getMonthsRemaining($service['nextduedate']);
        if ($monthsRemaining <= 0) return 0;
        
        $monthlyValue = getProduct($service['packageid'])['monthly'];
        $remainingValue = $monthlyValue * $monthsRemaining;
        $terminationFee = $remainingValue * self::EARLY_TERMINATION_PENALTY;
        
        return max($terminationFee, self::MINIMUM_FEE);
    }
    
    public static function processEarlyTermination($serviceId) {
        $fee = self::calculateTerminationFee($serviceId);
        $refund = ProrationCalculator::calculateUnusedCredit($serviceId);
        $refundAmount = max(0, $refund['credit_amount'] - $fee);
        
        if ($refundAmount > 0) {
            CreditManager::addCredit($service['userid'], $refundAmount, "Early termination refund");
        }
        
        localAPI('ModuleTerminate', ['serviceid' => $serviceId]);
        return ['fee' => $fee, 'refund' => $refundAmount];
    }
}
```
