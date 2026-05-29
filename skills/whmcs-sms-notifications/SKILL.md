# WHMCS SMS Notifications

## Concept
SMS messaging for urgent alerts and notifications.

## Code
```php
<?php
class SMSNotifier {
    public static function sendSMS($phone, $message) {
        $gateway = getSMSGateway();
        return $gateway->send(["to" => $phone, "message" => $message]);
    }
}
```
