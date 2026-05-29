# WHMCS Notification Module API

Complete reference for notification provider module development in WHMCS.

## Overview

Notification modules allow sending alerts through various channels like Slack, Discord, SMS, and more.

## Module Structure

```php
<?php
/**
 * WHMCS Notification Provider Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function yournotifier_MetaData()
{
    return [
        'DisplayName' => 'Your Notifier',
        'Description' => 'Send notifications via your channel',
        'Channels' => ['slack', 'webhook'],
        'Parameters' => [],
    ];
}

function yournotifier_ConfigArray()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Your Notifier',
        ],
        'WebhookUrl' => [
            'FriendlyName' => 'Webhook URL',
            'Type' => 'text',
            'Size' => '100',
            'Description' => 'Enter your webhook URL',
        ],
    ];
}
```

## Notification Functions

### send()

```php
/**
 * Send notification
 */
function yournotifier_send(array $params)
{
    try {
        $webhookUrl = $params['config']['webhookurl'];
        
        $payload = [
            'text' => $params['title'],
            'attachments' => [
                [
                    'text' => $params['body'],
                    'color' => mapSeverityToColor($params['severity']),
                    'fields' => array_map(function($field) {
                        return [
                            'title' => $field['label'],
                            'value' => $field['value'],
                            'short' => true,
                        ];
                    }, $params['fields'] ?? []),
                ],
            ],
        ];
        
        $ch = curl_init($webhookUrl);
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
        curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 10);
        
        $response = curl_exec($ch);
        $error = curl_error($ch);
        curl_close($ch);
        
        if ($error) {
            throw new Exception("Webhook failed: {$error}");
        }
        
        return ['success' => true];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * Map severity to color
 */
function mapSeverityToColor(string $severity): string
{
    $colors = [
        'critical' => '#FF0000',
        'important' => '#FFA500',
        'info' => '#00AA00',
    ];
    
    return $colors[$severity] ?? '#808080';
}
```

## Related Documentation

- [whmcs-integration-slack.md](../integration/whmcs-integration-slack.md)
- [whmcs-integration-discord.md](../integration/whmcs-integration-discord.md)