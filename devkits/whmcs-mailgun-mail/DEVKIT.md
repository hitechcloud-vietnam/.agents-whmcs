# WHMCS Mailgun Mail Module

## Overview
Mailgun SMTP integration for WHMCS email delivery.

## Module File: mailgun.php

```php
<?php
/**
 * WHMCS Mailgun Mail Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Mail\Contract\ProviderInterface;
use WHMCS\Mail\Log as MailLog;

class Mailgun_Mail_Provider implements ProviderInterface
{
    protected $config;

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    public function getName(): string
    {
        return 'Mailgun';
    }

    public function getUniqueId(): string
    {
        return 'mailgun';
    }

    public function isConfigured(): bool
    {
        return !empty($this->config['mailgun_api_key']) && !empty($this->config['mailgun_domain']);
    }

    public function send(MailLog $mail): array
    {
        try {
            $apiKey = $this->config['mailgun_api_key'];
            $domain = $this->config['mailgun_domain'];

            $data = [
                'from' => $this->config['mailgun_sender_email'],
                'to' => $mail->recipient,
                'subject' => $mail->subject,
                'html' => $mail->body,
                'text' => strip_tags($mail->body),
            ];

            if ($mail->cc) {
                $data['cc'] = $mail->cc;
            }

            $response = wp_remote_post("https://api.mailgun.net/v3/{$domain}/messages", [
                'headers' => [
                    'Authorization' => 'Basic ' . base64_encode("api:{$apiKey}"),
                ],
                'body' => $data,
                'timeout' => 30,
            ]);

            if (is_wp_error($response)) {
                throw new \Exception($response->get_error_message());
            }

            $statusCode = wp_remote_retrieve_response_code($response);

            if ($statusCode >= 400) {
                throw new \Exception("Mailgun API error: {$statusCode}");
            }

            return [
                'status' => 'success',
                'message_id' => 'mg-' . uniqid(),
            ];

        } catch (\Exception $e) {
            return [
                'status' => 'error',
                'error' => $e->getMessage(),
            ];
        }
    }

    public static function getConfigurationFields(): array
    {
        return [
            'mailgun_api_key' => [
                'FriendlyName' => 'API Key',
                'Type' => 'password',
                'Size' => '50',
            ],
            'mailgun_domain' => [
                'FriendlyName' => 'Domain',
                'Type' => 'text',
                'Size' => '50',
            ],
            'mailgun_sender_email' => [
                'FriendlyName' => 'Sender Email',
                'Type' => 'text',
                'Size' => '50',
            ],
        ];
    }
}

function getConfig(string $key = null)
{
    $config = require __DIR__ . '/config.php';
    return $key ? ($config[$key] ?? null) : $config;
}
```

## Configuration File: config.php

```php
<?php
return [
    'mailgun_api_key' => '',
    'mailgun_domain' => '',
    'mailgun_sender_email' => 'noreply@yourdomain.com',
];
```

## Activation & Deactivation

```php
<?php
function whmcs_mailgun_mail_activate(): array
{
    return ['status' => 'success', 'description' => 'Mailgun Mail module activated'];
}

function whmcs_mailgun_mail_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Mailgun Mail module deactivated'];
}

function whmcs_mailgun_mail_config(): array
{
    return Mailgun_Mail_Provider::getConfigurationFields();
}
```

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
