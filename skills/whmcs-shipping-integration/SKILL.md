# WHMCS Shipping Integration

## Concept
Shipping provider integration.

## Code
```php
<?php
class ShippingIntegration {
    public static function getShippingRates($orderId) {
        $order = getOrder($orderId);
        $rates = [];
        
        foreach ($this->carriers as $carrier) {
            $rates[$carrier] = $carrier->getRates($order);
        }
        
        return $rates;
    }
    
    public static function createShipment($orderId, $carrier, $service) {
        return $carrier->createShipment($orderId, $service);
    }
}
```
