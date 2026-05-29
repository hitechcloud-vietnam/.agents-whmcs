# WHMCS Tax Calculation

## Concept Explanation

Tax calculation in WHMCS handles various tax types including sales tax, VAT, GST, and custom tax rules. The system supports multiple tax levels, tax-exempt customers, and location-based tax rates. Proper tax configuration ensures compliance and accurate invoicing.

### Tax Types

- **Inclusive Tax**: Price includes tax
- **Exclusive Tax**: Tax added to price
- **Compound Tax**: Tax on tax (e.g., VAT + local tax)
- **US Sales Tax**: State/county/city based
- **EU VAT**: Based on customer location and status

## Code Patterns

### Tax Calculator

```php
<?php
// includes/TaxCalculator.php

class TaxCalculator {
    
    public static function calculateTax($amount, $clientId, $state = '', $country = '') {
        $client = getClientsDetails($clientId);
        $state = $state ?: $client['state'];
        $country = $country ?: $client['country'];
        
        // Check tax exempt
        if (self::isTaxExempt($clientId)) {
            return ['tax1' => 0, 'tax2' => 0, 'total' => $amount];
        }
        
        // Get applicable tax rates
        $taxRate1 = self::getTaxRate($country, $state, 1);
        $taxRate2 = self::getTaxRate($country, $state, 2);
        
        $tax1 = $amount * ($taxRate1 / 100);
        
        if (self::isCompoundTax()) {
            $tax2 = ($amount + $tax1) * ($taxRate2 / 100);
        } else {
            $tax2 = $amount * ($taxRate2 / 100);
        }
        
        return [
            'tax1' => round($tax1, 2),
            'tax2' => round($tax2, 2),
            'total' => round($amount + $tax1 + $tax2, 2),
            'tax1_rate' => $taxRate1,
            'tax2_rate' => $taxRate2
        ];
    }
    
    public static function getTaxRate($country, $state, $level) {
        $query = "SELECT taxrate FROM tbltax WHERE 
            (country = '" . e($country) . "' OR country = '') AND
            (state = '" . e($state) . "' OR state = '') AND
            level = " . (int)$level . "
            ORDER BY country DESC, state DESC
            LIMIT 1";
        $result = full_query($query);
        $row = mysql_fetch_array($result);
        return (float)($row['taxrate'] ?? 0);
    }
    
    public static function isTaxExempt($clientId) {
        $client = getClientsDetails($clientId);
        return $client['taxexempt'] == 'on';
    }
}
```

## Implementation Checklist

- [ ] Configure tax rates in WHMCS admin
- [ ] Set up EU VAT rules
- [ ] Configure US sales tax rates
- [ ] Enable tax exemption handling
- [ ] Test tax calculations
