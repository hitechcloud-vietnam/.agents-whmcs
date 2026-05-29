# WHMCS Live Chat Widget Module

## Overview
Live chat integration module for WHMCS client area.

## Module File: live_chat.php

```php
<?php
/**
 * WHMCS Live Chat Widget Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Database\Capsule;

class WHMCS_Live_Chat
{
    protected $config;

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    /**
     * Get chat widget code
     */
    public function getWidgetCode(): string
    {
        $widgetId = $this->config['widget_id'] ?? '';
        $apiKey = $this->config['api_key'] ?? '';

        return <<<HTML
<script>
    window.__lc = {
        license: {$widgetId},
        api_key: '{$apiKey}',
        params: {
            session_id: '{$this->getSessionId()}'
        }
    };
</script>
<script async src="https://cdn.livechat.com/livechat.js"></script>
HTML;
    }

    /**
     * Get session ID
     */
    protected function getSessionId(): string
    {
        if (isset($_SESSION['livechat_session'])) {
            return $_SESSION['livechat_session'];
        }
        $sessionId = session_id() . '_' . time();
        $_SESSION['livechat_session'] = $sessionId;
        return $sessionId;
    }

    /**
     * Log chat conversation
     */
    public function logChat(int $ticketId, array $messages): bool
    {
        foreach ($messages as $msg) {
            Capsule::table('mod_livechat_logs')->insert([
                'ticket_id' => $ticketId,
                'message' => $msg['content'],
                'sender' => $msg['sender'],
                'timestamp' => $msg['timestamp'],
                'created_at' => date('Y-m-d H:i:s'),
            ]);
        }
        return true;
    }
}

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $chat = new WHMCS_Live_Chat();
    return $chat->getWidgetCode();
});

function whmcs_live_chat_activate(): array
{
    try {
        if (!Capsule::schema()->hasTable('mod_livechat_logs')) {
            Capsule::schema()->create('mod_livechat_logs', function ($table) {
                $table->increments('id');
                $table->integer('ticket_id')->unsigned();
                $table->text('message');
                $table->enum('sender', ['client', 'agent', 'system']);
                $table->timestamp('timestamp');
                $table->timestamp('created_at')->useCurrent();
            });
        }
        return ['status' => 'success', 'description' => 'Live Chat Widget activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_live_chat_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Live Chat Widget deactivated'];
}

function whmcs_live_chat_config(): array
{
    return [
        'widget_id' => ['FriendlyName' => 'Widget License ID', 'Type' => 'text', 'Size' => '20'],
        'api_key' => ['FriendlyName' => 'API Key', 'Type' => 'password', 'Size' => '50'],
        'auto_launch' => ['FriendlyName' => 'Auto Launch', 'Type' => 'yesno'],
    ];
}
```

## Configuration File: config.php

```php
<?php
return [
    'widget_id' => '',
    'api_key' => '',
    'auto_launch' => false,
];
```

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
