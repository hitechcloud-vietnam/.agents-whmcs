# WHMCS Mandrill Mail Module

## Overview
Mandrill mail provider integration for WHMCS.

## Module File: mandrill.php

```php
<?php
/**
 * WHMCS Mandrill Mail Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Mail\Contract\ProviderInterface;
use WHMCS\Mail\Log as MailLog;

class Mandrill_Mail_Provider implements ProviderInterface
{
    protected $config;
    protected $apiUrl = 'https://mandrillapp.com/api/1.0/messages/send';

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    public function getName(): string
    {
        return 'Mandrill';
    }

    public function getUniqueId(): string
    {
        return 'mandrill';
    }

    public function isConfigured(): bool
    {
        return !empty($this->config['mandrill_api_key']) && !empty($this->config['sender_email']);
    }

    public function send(MailLog $mail): array
    {
        try {
            $apiKey = $this->config['mandrill_api_key'];

            $message = [
                'html' => $mail->body,
                'subject' => $mail->subject,
                'from_email' => $this->config['sender_email'],
                'from_name' => $this->config['sender_name'] ?? '',
                'to' => [['email' => $mail->recipient]],
            ];

            $data = [
                'key' => $apiKey,
                'message' => $message,
            ];

            $response = wp_remote_post($this->apiUrl, [
                'headers' => ['Content-Type' => 'application/json'],
                'body' => json_encode($data),
                'timeout' => 30,
            ]);

            if (is_wp_error($response)) {
                throw new \Exception($response->get_error_message());
            }

            $body = json_decode(wp_remote_retrieve_body($response), true);

            if (!empty($body[0]['status']) && $body[0]['status'] === 'error') {
                throw new \Exception($body[0]['reject_reason'] ?? 'Mandrill error');
            }

            return [
                'status' => 'success',
                'message_id' => $body[0]['_id'] ?? 'md-' . uniqid(),
            ];

        } catch (\Exception $e) {
            return ['status' => 'error', 'error' => $e->getMessage()];
        }
    }

    public static function getConfigurationFields(): array
    {
        return [
            'mandrill_api_key' => ['FriendlyName' => 'API Key', 'Type' => 'password', 'Size' => '50'],
            'sender_email' => ['FriendlyName' => 'Sender Email', 'Type' => 'text', 'Size' => '50'],
            'sender_name' => ['FriendlyName' => 'Sender Name', 'Type' => 'text', 'Size' => '50'],
        ];
    }
}

function getConfig(string $key = null)
{
    $config = require __DIR__ . '/config.php';
    return $key ? ($config[$key] ?? null) : $config;
}

function whmcs_mandrill_mail_activate(): array
{
    return ['status' => 'success', 'description' => 'Mandrill Mail module activated'];
}

function whmcs_mandrill_mail_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Mandrill Mail module deactivated'];
}

function whmcs_mandrill_mail_config(): array
{
    return Mandrill_Mail_Provider::getConfigurationFields();
}
```

## Configuration File: config.php

```php
<?php
return [
    'mandrill_api_key' => '',
    'sender_email' => 'noreply@yourdomain.com',
    'sender_name' => '',
];
```
