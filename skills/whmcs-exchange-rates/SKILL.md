# WHMCS Exchange Rates

## Concept
Real-time exchange rate updates for accurate multi-currency transactions.

## Code
```php
<?php
class ExchangeRateService {
    public static function updateRates() {
        $rates = file_get_contents("https://api.exchangerate-api.com/v4/latest/USD");
        $data = json_decode($rates, true);
        
        foreach ($data["rates"] as $code => $rate) {
            full_query("UPDATE tblcurrencies SET rate = " . (float)$rate . " WHERE code =  . e() . ");
        }
    }
}
add_hook("DailyCronJob", 1, function() { ExchangeRateService::updateRates(); });
```
