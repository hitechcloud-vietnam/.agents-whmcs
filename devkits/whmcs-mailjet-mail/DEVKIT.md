# WHMCS Mailjet Mail Module

## Overview
Mailjet SMTP integration for WHMCS.

## Module File: mailjet.php

```php
<?php
/**
 * WHMCS Mailjet Mail Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Mail\Contract\ProviderInterface;
use WHMCS\Mail\Log as MailLog;

class Mailjet_Mail_Provider implements ProviderInterface
{
    protected $config;
    protected $apiUrl = 'https://api.mailjet.com/v3.1/send';

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    public function getName(): string
    {
        return 'Mailjet';
    }

    public function getUniqueId(): string
    {
        return 'mailjet';
    }

    public function isConfigured(): bool
    {
        return !empty($this->config['mailjet_api_key']) && !empty($this->config['mailjet_secret_key']);
    }

    public function send(MailLog $mail): array
    {
        try {
            $apiKey = $this->config['mailjet_api_key'];
            $secretKey = $this->config['mailjet_secret_key'];

            $data = [
                'Messages' => [[
                    'From' => [
                        'Email' => $this->config['sender_email'],
                        'Name' => $this->config['sender_name'] ?? '',
                    ],
                    'To' => [['Email' => $mail->recipient]],
                    'Subject' => $mail->subject,
                    'HTMLPart' => $mail->body,
                    'TextPart' => strip_tags($mail->body),
                ]],
            ];

            $response = wp_remote_post($this->apiUrl, [
                'headers' => [
                    'Authorization' => 'Basic ' . base64_encode("{$apiKey}:{$secretKey}"),
                    'Content-Type' => 'application/json',
                ],
                'body' => json_encode($data),
                'timeout' => 30,
            ]);

            if (is_wp_error($response)) {
                throw new \Exception($response->get_error_message());
            }

            $body = json_decode(wp_remote_retrieve_body($response), true);

            if (!empty($body[0]['Status']) && $body[0]['Status'] === 'error') {
                throw new \Exception($body[0]['StatusMessage'] ?? 'Mailjet error');
            }

            return [
                'status' => 'success',
                'message_id' => $body[0]['To'][0]['MessageID'] ?? 'mj-' . uniqid(),
            ];

        } catch (\Exception $e) {
            return ['status' => 'error', 'error' => $e->getMessage()];
        }
    }

    public static function getConfigurationFields(): array
    {
        return [
            'mailjet_api_key' => ['FriendlyName' => 'API Key', 'Type' => 'text', 'Size' => '50'],
            'mailjet_secret_key' => ['FriendlyName' => 'Secret Key', 'Type' => 'password', 'Size' => '50'],
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

function whmcs_mailjet_mail_activate(): array
{
    return ['status' => 'success', 'description' => 'Mailjet Mail module activated'];
}

function whmcs_mailjet_mail_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Mailjet Mail module deactivated'];
}

function whmcs_mailjet_mail_config(): array
{
    return Mailjet_Mail_Provider::getConfigurationFields();
}
```

## Configuration File: config.php

```php
<?php
return [
    'mailjet_api_key' => '',
    'mailjet_secret_key' => '',
    'sender_email' => 'noreply@yourdomain.com',
    'sender_name' => '',
];
```
