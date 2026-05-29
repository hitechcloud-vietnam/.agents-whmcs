# WHMCS Telegram Automation Workflow

## Overview
This workflow implements Telegram bot integration for WHMCS notifications.

## Prerequisites
- WHMCS installation
- Telegram Bot Token
- Telegram Chat ID(s)

## Step-by-Step Process

### Step 1: Create Telegram Manager
```php
<?php
// /includes/telegram/TelegramManager.php

namespace WHMCS\Telegram;

class TelegramManager
{
    private $botToken;
    private $chatIds = [];

    public function __construct()
    {
        $this->botToken = getConfig('telegram_bot_token');
        $this->chatIds = [
            'default' => getConfig('telegram_chat_id'),
            'alerts' => getConfig('telegram_alerts_chat_id'),
            'sales' => getConfig('telegram_sales_chat_id')
        ];
    }

    /**
     * Send message
     */
    public function send(string $message, string $chatKey = 'default', array $options = []): array
    {
        $chatId = $this->chatIds[$chatKey] ?? $this->chatIds['default'];

        if (!$this->botToken || !$chatId) {
            return ['success' => false, 'error' => 'Telegram not configured'];
        }

        $url = "https://api.telegram.org/bot{$this->botToken}/sendMessage";

        $payload = [
            'chat_id' => $chatId,
            'text' => $message,
            'parse_mode' => $options['parse_mode'] ?? 'Markdown'
        ];

        if (isset($options['keyboard'])) {
            $payload['reply_markup'] = json_encode($options['keyboard']);
        }

        return $this->makeRequest($url, $payload);
    }

    /**
     * Send with inline keyboard
     */
    public function sendWithKeyboard(string $message, array $buttons, string $chatKey = 'default'): array
    {
        $keyboard = [
            'inline_keyboard' => array_map(function($button) {
                return [['text' => $button['text'], 'url' => $button['url']]];
            }, $buttons)
        ];

        return $this->send($message, $chatKey, ['keyboard' => $keyboard]);
    }

    /**
     * Send formatted notification
     */
    public function sendNotification(string $title, string $message, string $chatKey = 'default'): array
    {
        $formattedMessage = "*WHMCS Notification*\n\n";
        $formattedMessage .= "*{$title}*\n";
        $formattedMessage .= $message;
        $formattedMessage .= "\n\n_" . date('Y-m-d H:i:s') . "_";

        return $this->send($formattedMessage, $chatKey);
    }

    /**
     * Send invoice notification
     */
    public function invoiceNotification(int $invoiceId, string $event): array
    {
        $invoice = getInvoice($invoiceId);
        $client = getClientsDetails($invoice['userid']);

        $events = [
            'created' => ['icon' => '📝', 'title' => 'New Invoice Created'],
            'paid' => ['icon' => '💰', 'title' => 'Payment Received!'],
            'overdue' => ['icon' => '⚠️', 'title' => 'Invoice Overdue']
        ];

        $config = $events[$event] ?? $events['created'];

        $message = "{$config['icon']} *{$config['title']}*\n\n";
        $message .= "Invoice: #{$invoiceId}\n";
        $message .= "Client: {$client['fullname']}\n";
        $message .= "Amount: " . formatCurrency($invoice['total']) . "\n";
        $message .= "Status: {$invoice['status']}";

        return $this->send($message, 'sales');
    }

    /**
     * Send service notification
     */
    public function serviceNotification(int $serviceId, string $event): array
    {
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();

        $client = getClientsDetails($service->userid);

        $events = [
            'created' => ['icon' => '✅', 'title' => 'New Service'],
            'suspended' => ['icon' => '⏸️', 'title' => 'Service Suspended'],
            'terminated' => ['icon' => '❌', 'title' => 'Service Terminated']
        ];

        $config = $events[$event] ?? $events['created'];

        $message = "{$config['icon']} *{$config['title']}*\n\n";
        $message .= "Service: {$service->domain}\n";
        $message .= "Client: {$client['fullname']}\n";
        $message .= "Status: {$service->domainstatus}";

        return $this->send($message);
    }

    private function makeRequest(string $url, array $payload): array
    {
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $payload,
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        $result = json_decode($response, true);

        if (isset($result['ok']) && $result['ok']) {
            return ['success' => true, 'message_id' => $result['result']['message_id']];
        }

        return ['success' => false, 'error' => $result['description'] ?? 'Unknown error'];
    }
}
```

### Step 2: Create Telegram Hooks
```php
<?php
// /includes/hooks/telegram_hooks.php

use WHMCS\Telegram\TelegramManager;

$telegramManager = new TelegramManager();

add_hook('InvoicePaid', 1, function($vars) use ($telegramManager) {
    $telegramManager->invoiceNotification($vars['invoiceid'], 'paid');
});

add_hook('ServiceSuspended', 1, function($vars) use ($telegramManager) {
    $telegramManager->serviceNotification($vars['serviceid'], 'suspended');
});
```

## Related Workflows
- [WHMCS Slack Automation](./whmcs-slack-automation.md)
- [WHMCS Discord Automation](./whmcs-discord-automation.md)