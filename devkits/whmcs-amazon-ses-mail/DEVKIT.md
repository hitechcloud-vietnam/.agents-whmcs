# WHMCS Amazon SES Mail Module

## Overview
AWS SES email delivery integration for WHMCS.

## Module File: ses.php

```php
<?php
/**
 * WHMCS Amazon SES Mail Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Mail\Contract\ProviderInterface;
use WHMCS\Mail\Log as MailLog;

class Amazon_SES_Mail_Provider implements ProviderInterface
{
    protected $config;

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    public function getName(): string
    {
        return 'Amazon SES';
    }

    public function getUniqueId(): string
    {
        return 'amazon_ses';
    }

    public function isConfigured(): bool
    {
        return !empty($this->config['aws_access_key']) && 
               !empty($this->config['aws_secret_key']) &&
               !empty($this->config['aws_region']);
    }

    public function send(MailLog $mail): array
    {
        try {
            $accessKey = $this->config['aws_access_key'];
            $secretKey = $this->config['aws_secret_key'];
            $region = $this->config['aws_region'];
            $senderEmail = $this->config['sender_email'];

            $to = $mail->recipient;
            $subject = $mail->subject;
            $htmlBody = $mail->body;
            $textBody = strip_tags($mail->body);

            $payload = [
                'Source' => $senderEmail,
                'Destination' => [
                    'ToAddresses' => [$to],
                ],
                'Message' => [
                    'Subject' => ['Data' => $subject],
                    'Body' => [
                        'Html' => ['Data' => $htmlBody],
                        'Text' => ['Data' => $textBody],
                    ],
                ],
            ];

            if ($mail->cc) {
                $payload['Destination']['CcAddresses'] = explode(',', $mail->cc);
            }

            $service = 'email';
            $host = 'email.' . $region . '.amazonaws.com';
            $endpoint = 'https://' . $host . '/';

            $date = gmdate('Ymd\THis\Z');
            $amzDate = substr($date, 0, 8) . 'T' . substr($date, 9, 6) . 'Z';
            $algorithm = 'AWS4-HMAC-SHA256';
            $credentialScope = substr($date, 0, 8) . '/' . $region . '/' . $service . '/aws4_request';

            $headers = [
                'Content-Type' => 'application/x-www-form-urlencoded',
                'Host' => $host,
                'X-Amz-Date' => $amzDate,
            ];

            $signedHeaders = 'host;x-amz-date';
            $canonicalHeaders = "host:{$host}\nx-amz-date:{$amzDate}\n";
            $canonicalRequest = "POST\n/\n\n{$canonicalHeaders}\n{$signedHeaders}\n" . hash('sha256', http_build_query($payload));

            $stringToSign = "{$algorithm}\n{$amzDate}\n{$credentialScope}\n" . hash('sha256', $canonicalRequest);
            $kSecret = 'AWS4' . $secretKey;
            $kDate = hash_hmac('sha256', substr($date, 0, 8), $kSecret, true);
            $kRegion = hash_hmac('sha256', $region, $kDate, true);
            $kService = hash_hmac('sha256', $service, $kRegion, true);
            $kSigning = hash_hmac('sha256', 'aws4_request', $kService, true);
            $signature = hash_hmac('sha256', $stringToSign, $kSigning);

            $authorization = "{$algorithm} Credential={$accessKey}/{$credentialScope}, SignedHeaders={$signedHeaders}, Signature={$signature}";

            $headers['Authorization'] = $authorization;

            $response = wp_remote_post($endpoint, [
                'headers' => $headers,
                'body' => http_build_query($payload),
                'timeout' => 30,
            ]);

            if (is_wp_error($response)) {
                throw new \Exception($response->get_error_message());
            }

            $statusCode = wp_remote_retrieve_response_code($response);

            if ($statusCode >= 400) {
                throw new \Exception("AWS SES error: {$statusCode}");
            }

            return [
                'status' => 'success',
                'message_id' => 'ses-' . uniqid(),
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
            'aws_access_key' => [
                'FriendlyName' => 'AWS Access Key',
                'Type' => 'text',
                'Size' => '50',
            ],
            'aws_secret_key' => [
                'FriendlyName' => 'AWS Secret Key',
                'Type' => 'password',
                'Size' => '50',
            ],
            'aws_region' => [
                'FriendlyName' => 'AWS Region',
                'Type' => 'dropdown',
                'Options' => 'us-east-1,us-west-2,eu-west-1,eu-central-1,ap-southeast-1,ap-northeast-1',
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
```

## Configuration File: config.php

```php
<?php
return [
    'aws_access_key' => '',
    'aws_secret_key' => '',
    'aws_region' => 'us-east-1',
    'sender_email' => 'noreply@yourdomain.com',
];
```

## Activation & Deactivation

```php
<?php
function whmcs_amazon_ses_mail_activate(): array
{
    return ['status' => 'success', 'description' => 'Amazon SES Mail module activated'];
}

function whmcs_amazon_ses_mail_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Amazon SES Mail module deactivated'];
}

function whmcs_amazon_ses_mail_config(): array
{
    return Amazon_SES_Mail_Provider::getConfigurationFields();
}
```

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
- cURL extension
