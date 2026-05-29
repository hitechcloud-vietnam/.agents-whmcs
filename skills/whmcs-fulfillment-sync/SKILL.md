# WHMCS Fulfillment Sync

## Concept
Order fulfillment synchronization.

## Code
```php
<?php
class FulfillmentSync {
    public static function syncOrderStatus($orderId) {
        $order = getOrder($orderId);
        $fulfillment = $this->getFulfillmentService($order["fulfillment_id"]);
        $fulfillment->updateStatus(["order_id" => $order["id"], "status" => $order["status"]]);
    }
}
```
