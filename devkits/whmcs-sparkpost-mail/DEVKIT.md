# WHMCS SparkPost Mail Module

## Overview
SparkPost email delivery integration for WHMCS.

## Module File: sparkpost.php

```php
<?php
/**
 * WHMCS SparkPost Mail Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Mail\Contract\ProviderInterface;
use WHMCS\Mail\Log as MailLog;

class Sparkpost_Mail_Provider implements ProviderInterface
{
    protected $config;
    protected $apiUrl = 'https://api.sparkpost.com/api/v1/transmissions';

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    public function getName(): string
    {
        return 'SparkPost';
    }

    public function getUniqueId(): string
    {
        return 'sparkpost';
    }

    public function isConfigured(): bool
    {
        return !empty($this->config['sparkpost_api_key']) && !empty($this->config['sender_email']);
    }

    public function send(MailLog $mail): array
    {
        try {
            $apiKey = $this->config['sparkpost_api_key'];

            $recipients = [['address' => ['email' => $mail->recipient]]];

            $data = [
                'options' => [
                    'sandbox' => !empty($this->config['sparkpost_sandbox']),
                ],
                'content' => [
                    'from' => [
                        'email' => $this->config['sender_email'],
                        'name' => $this->config['sender_name'] ?? '',
                    ],
                    'subject' => $mail->subject,
                    'html' => $mail->body,
                    'text' => strip_tags($mail->body),
                ],
                'recipients' => $recipients,
            ];

            $response = wp_remote_post($this->apiUrl, [
                'headers' => [
                    'Authorization' => $apiKey,
                    'Content-Type' => 'application/json',
                ],
                'body' => json_encode($data),
                'timeout' => 30,
            ]);

            if (is_wp_error($response)) {
                throw new \Exception($response->get_error_message());
            }

            $body = json_decode(wp_remote_retrieve_body($response), true);
            $statusCode = wp_remote_retrieve_response_code($response);

            if ($statusCode >= 400 || !empty($body['errors'])) {
                throw new \Exception($body['errors'][0]['message'] ?? 'SparkPost API error');
            }

            return [
                'status' => 'success',
                'message_id' => $body['results']['id'] ?? 'sp-' . uniqid(),
            ];

        } catch (\Exception $e) {
            return ['status' => 'error', 'error' => $e->getMessage()];
        }
    }

    public static function getConfigurationFields(): array
    {
        return [
            'sparkpost_api_key' => ['FriendlyName' => 'API Key', 'Type' => 'password', 'Size' => '50'],
            'sender_email' => ['FriendlyName' => 'Sender Email', 'Type' => 'text', 'Size' => '50'],
            'sender_name' => ['FriendlyName' => 'Sender Name', 'Type' => 'text', 'Size' => '50'],
            'sparkpost_sandbox' => ['FriendlyName' => 'Sandbox Mode', 'Type' => 'yesno'],
        ];
    }
}

function getConfig(string $key = null)
{
    $config = require __DIR__ . '/config.php';
    return $key ? ($config[$key] ?? null) : $config;
}

function whmcs_sparkpost_mail_activate(): array
{
    return ['status' => 'success', 'description' => 'SparkPost Mail module activated'];
}

function whmcs_sparkpost_mail_deactivate(): array
{
    return ['status' => 'success', 'description' => 'SparkPost Mail module deactivated'];
}

function whmcs_sparkpost_mail_config(): array
{
    return Sparkpost_Mail_Provider::getConfigurationFields();
}
```

## Configuration File: config.php

```php
<?php
return [
    'sparkpost_api_key' => '',
    'sender_email' => 'noreply@yourdomain.com',
    'sender_name' => '',
    'sparkpost_sandbox' => false,
];
```
