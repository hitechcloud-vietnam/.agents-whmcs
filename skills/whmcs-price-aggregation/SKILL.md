# WHMCS Price Aggregation

## Concept
Price comparison and aggregation.

## Code
```php
<?php
class PriceAggregation {
    public static function aggregatePrices($productId) {
        $prices = [];
        foreach ($this->sources as $source) {
            $prices[$source->name] = $source->getPrice($productId);
        }
        return ["min" => min($prices), "max" => max($prices), "avg" => array_sum($prices) / count($prices)];
    }
}
```
