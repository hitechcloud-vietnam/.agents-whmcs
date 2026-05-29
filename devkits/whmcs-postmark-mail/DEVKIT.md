# WHMCS Postmark Mail Module

## Overview
Postmark email delivery integration for WHMCS.

## Module File: postmark.php

```php
<?php
/**
 * WHMCS Postmark Mail Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Mail\Contract\ProviderInterface;
use WHMCS\Mail\Log as MailLog;

class Postmark_Mail_Provider implements ProviderInterface
{
    protected $config;
    protected $apiUrl = 'https://api.postmarkapp.com/email';

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    public function getName(): string
    {
        return 'Postmark';
    }

    public function getUniqueId(): string
    {
        return 'postmark';
    }

    public function isConfigured(): bool
    {
        return !empty($this->config['postmark_api_token']) && !empty($this->config['sender_email']);
    }

    public function send(MailLog $mail): array
    {
        try {
            $token = $this->config['postmark_api_token'];

            $data = [
                'From' => $this->config['sender_email'],
                'To' => $mail->recipient,
                'Subject' => $mail->subject,
                'HtmlBody' => $mail->body,
                'TextBody' => strip_tags($mail->body),
            ];

            if ($mail->cc) {
                $data['Cc'] = $mail->cc;
            }

            if ($mail->bcc) {
                $data['Bcc'] = $mail->bcc;
            }

            $response = wp_remote_post($this->apiUrl, [
                'headers' => [
                    'Accept' => 'application/json',
                    'Content-Type' => 'application/json',
                    'X-Postmark-Account-Token' => $token,
                ],
                'body' => json_encode($data),
                'timeout' => 30,
            ]);

            if (is_wp_error($response)) {
                throw new \Exception($response->get_error_message());
            }

            $body = json_decode(wp_remote_retrieve_body($response), true);
            $statusCode = wp_remote_retrieve_response_code($response);

            if ($statusCode >= 400 || isset($body['ErrorCode'])) {
                throw new \Exception($body['Message'] ?? 'Postmark API error');
            }

            return [
                'status' => 'success',
                'message_id' => $body['MessageID'] ?? 'pm-' . uniqid(),
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
            'postmark_api_token' => [
                'FriendlyName' => 'API Token',
                'Type' => 'password',
                'Size' => '50',
            ],
            'sender_email' => [
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

function whmcs_postmark_mail_activate(): array
{
    return ['status' => 'success', 'description' => 'Postmark Mail module activated'];
}

function whmcs_postmark_mail_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Postmark Mail module deactivated'];
}

function whmcs_postmark_mail_config(): array
{
    return Postmark_Mail_Provider::getConfigurationFields();
}
