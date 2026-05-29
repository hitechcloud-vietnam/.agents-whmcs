# WHMCS Payment Aggregation

## Concept
Payment aggregator integration.

## Code
```php
<?php
class PaymentAggregator {
    public static function processPayment($invoiceId, $gateway) {
        $invoice = getInvoice($invoiceId);
        return $gateway->charge(["amount" => $invoice["total"], "customer" => $invoice["userid"]]);
    }
    
    public static function getAggregatedFees($gateway) {
        return ["processing_fee" => $gateway->getFeeRate(), "fixed_fee" => $gateway->getFixedFee()];
    }
}
```
