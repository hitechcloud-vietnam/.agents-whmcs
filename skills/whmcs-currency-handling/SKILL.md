# WHMCS Currency Handling

## Concept Explanation
Multi-currency support for international customers.

## Code Patterns

```php
<?php
class CurrencyHandler {
    public static function convertAmount($amount, $from, $to) {
        $rate = self::getExchangeRate($from, $to);
        return $amount * $rate;
    }
}
```
