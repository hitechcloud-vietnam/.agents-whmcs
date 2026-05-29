# WHMCS WhatsApp Business

## Concept
WhatsApp integration for customer communication.

## Code
```php
<?php
class WhatsAppNotifier {
    public static function sendMessage($phone, $message) {
        $response = wp_remote_post(WHATSAPP_API_URL . "/messages", [
            "body" => json_encode(["to" => $phone, "text" => ["body" => $message]])
        ]);
        return json_decode($response["body"], true);
    }
}
```
