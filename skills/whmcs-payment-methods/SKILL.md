# WHMCS Payment Methods

## Concept Explanation
Supporting multiple payment methods increases conversion rates.

## Code Patterns

```php
<?php
class PaymentMethodManager {
    public static function setPreferredMethod($clientId, $gateway, $token = null) {
        update_query("tblclients", ["defaultgateway" => $gateway], ["id" => $clientId]);
    }
}
```
