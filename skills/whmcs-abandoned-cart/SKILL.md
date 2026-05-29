# WHMCS Abandoned Cart

## Concept
Cart abandonment recovery campaigns.

## Code
```php
<?php
class AbandonedCartRecovery {
    public static function sendReminder($cartId) {
        $cart = getAbandonedCart($cartId);
        send_email("abandoned_cart_reminder", $cart["userid"], ["cart_id" => $cartId]);
    }
}
add_hook("DailyCronJob", 1, function() {
    $abandoned = getAbandonedCarts(24);
    foreach ($abandoned as $cart) {
        if ($cart["reminder_count"] < 3) {
            AbandonedCartRecovery::sendReminder($cart["id"]);
        }
    }
});
```
