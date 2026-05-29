# WHMCS SendGrid Mail Module

## Overview
SendGrid email delivery integration for WHMCS.

## Module File: sendgrid.php

```php
<?php
/**
 * WHMCS SendGrid Mail Module
 * 
 * @package    WHMCS\Mail
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Mail\Contract\ProviderInterface;
use WHMCS\Mail\Log as MailLog;
use PHPMailer\PHPMailer\PHPMailer;

class SendGrid_Mail_Provider implements ProviderInterface
{
    /**
     * @var array Configuration
     */
    protected $config;

    /**
     * @var string API endpoint
     */
    protected $apiEndpoint = 'https://api.sendgrid.com/v3/';

    /**
     * Constructor
     */
    public function __construct()
    {
        $this->config = getConfig();
    }

    /**
     * Get provider name
     */
    public function getName(): string
    {
        return 'SendGrid';
    }

    /**
     * Get provider ID
     */
    public function getUniqueId(): string
    {
        return 'sendgrid';
    }

    /**
     * Check if provider is configured
     */
    public function isConfigured(): bool
    {
        $apiKey = $this->config['SendGridApiKey'] ?? '';
        $senderEmail = $this->config['SendGridSenderEmail'] ?? '';
        
        return !empty($apiKey) && !empty($senderEmail);
    }

    /**
     * Send email
     */
    public function send(MailLog $mail): array
    {
        try {
            $apiKey = $this->config['SendGridApiKey'];
            $senderEmail = $this->config['SendGridSenderEmail'];
            $senderName = $this->config['SendGridSenderName'] ?? '';

            $message = $this->buildMessage($mail);

            $data = [
                'personalizations' => [[
                    'to' => $this->formatRecipients($mail->recipient),
                    'subject' => $mail->subject,
                ]],
                'from' => [
                    'email' => $senderEmail,
                    'name' => $senderName ?: $this->getConfigValue('CompanyName'),
                ],
                'content' => [
                    [
                        'type' => 'text/plain',
                        'value' => $message['plain'],
                    ],
                    [
                        'type' => 'text/html',
                        'value' => $message['html'],
                    ],
                ],
            ];

            // Add CC/BCC if present
            if ($mail->cc) {
                $data['personalizations'][0]['cc'] = $this->formatRecipients($mail->cc);
            }
            if ($mail->bcc) {
                $data['personalizations'][0]['bcc'] = $this->formatRecipients($mail->bcc);
            }

            // Add attachments
            if (!empty($mail->attachments)) {
                $data['attachments'] = $this->formatAttachments($mail->attachments);
            }

            // Add tracking settings
            if (!empty($this->config['SendGridTrackOpens'])) {
                $data['tracking_settings'] = [
                    'open_tracking' => ['enable' => true],
                    'click_tracking' => ['enable' => true],
                ];
            }

            $response = $this->makeRequest('mail/send', $data, $apiKey);

            return [
                'status' => 'success',
                'message_id' => $this->extractMessageId($response),
            ];

        } catch (\Exception $e) {
            return [
                'status' => 'error',
                'error' => $e->getMessage(),
            ];
        }
    }

    /**
     * Build email message
     */
    protected function buildMessage(MailLog $mail): array
    {
        $plain = strip_tags($mail->body);
        $html = $mail->body;

        return [
            'plain' => $plain,
            'html' => $html,
        ];
    }

    /**
     * Format recipients for SendGrid API
     */
    protected function formatRecipients(string $recipients): array
    {
        $formatted = [];
        $pairs = explode(',', $recipients);

        foreach ($pairs as $pair) {
            $pair = trim($pair);
            if (preg_match('/^(.+)\s+<(.+)>$/', $pair, $matches)) {
                $formatted[] = [
                    'name' => trim($matches[1]),
                    'email' => trim($matches[2]),
                ];
            } else {
                $formatted[] = ['email' => $pair];
            }
        }

        return $formatted;
    }

    /**
     * Format attachments
     */
    protected function formatAttachments(array $attachments): array
    {
        $formatted = [];

        foreach ($attachments as $attachment) {
            $content = chunk_split(base64_encode(file_get_contents($attachment['path'])));
            $formatted[] = [
                'content' => $content,
                'filename' => $attachment['name'],
                'type' => $attachment['type'] ?: 'application/octet-stream',
            ];
        }

        return $formatted;
    }

    /**
     * Make API request
     */
    protected function makeRequest(string $endpoint, array $data, string $apiKey): array
    {
        $url = $this->apiEndpoint . $endpoint;

        $args = [
            'headers' => [
                'Authorization' => 'Bearer ' . $apiKey,
                'Content-Type' => 'application/json',
            ],
            'body' => json_encode($data),
            'timeout' => 30,
        ];

        $response = wp_remote_post($url, $args);

        if (is_wp_error($response)) {
            throw new \Exception($response->get_error_message());
        }

        $statusCode = wp_remote_retrieve_response_code($response);

        if ($statusCode >= 400) {
            $body = wp_remote_retrieve_body($response);
            throw new \Exception("SendGrid API error: {$statusCode} - {$body}");
        }

        return [
            'status_code' => $statusCode,
            'body' => json_decode(wp_remote_retrieve_body($response), true),
        ];
    }

    /**
     * Extract message ID from response
     */
    protected function extractMessageId(array $response): string
    {
        return 'sg-' . uniqid();
    }

    /**
     * Get configuration fields
     */
    public static function getConfigurationFields(): array
    {
        return [
            'SendGridApiKey' => [
                'FriendlyName' => 'API Key',
                'Type' => 'password',
                'Size' => '50',
                'Description' => 'Your SendGrid API key',
            ],
            'SendGridSenderEmail' => [
                'FriendlyName' => 'Sender Email',
                'Type' => 'text',
                'Size' => '50',
                'Description' => 'Default sender email address',
            ],
            'SendGridSenderName' => [
                'FriendlyName' => 'Sender Name',
                'Type' => 'text',
                'Size' => '50',
                'Description' => 'Default sender name',
            ],
            'SendGridTrackOpens' => [
                'FriendlyName' => 'Track Opens',
                'Type' => 'yesno',
                'Description' => 'Enable open tracking',
            ],
        ];
    }

    /**
     * Test connection
     */
    public function testConnection(): array
    {
        try {
            $apiKey = $this->config['SendGridApiKey'];
            
            $response = $this->makeRequest('user/profile', [], $apiKey);
            
            return [
                'success' => true,
                'message' => 'Connection successful',
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'message' => $e->getMessage(),
            ];
        }
    }
}

/**
 * Get configuration value
 */
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
    'SendGridApiKey' => '',
    'SendGridSenderEmail' => 'noreply@yourdomain.com',
    'SendGridSenderName' => '',
    'SendGridTrackOpens' => true,
];
```

## Activation & Deactivation

```php
<?php
function whmcs_sendgrid_mail_activate(): array
{
    try {
        return [
            'status' => 'success',
            'description' => 'SendGrid Mail module activated. Configure API key in module settings.',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to activate: ' . $e->getMessage(),
        ];
    }
}

function whmcs_sendgrid_mail_deactivate(): array
{
    return ['status' => 'success', 'description' => 'SendGrid Mail module deactivated'];
}

function whmcs_sendgrid_mail_config(): array
{
    return SendGrid_Mail_Provider::getConfigurationFields();
}
```

## Provider Interface Implementation

The mail module implements `WHMCS\Mail\Contract\ProviderInterface` requiring:

| Method | Description |
|--------|-------------|
| getName() | Return provider display name |
| getUniqueId() | Return unique identifier |
| isConfigured() | Check if provider is properly configured |
| send(MailLog $mail) | Send the email |

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
- cURL extension
