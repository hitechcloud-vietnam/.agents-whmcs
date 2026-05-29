# WHMCS SMTP Connector Module

## Overview
Generic SMTP connector module for WHMCS email delivery.

## Module File: smtp.php

```php
<?php
/**
 * WHMCS SMTP Connector Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Mail\Contract\ProviderInterface;
use WHMCS\Mail\Log as MailLog;

class SMTP_Connector_Provider implements ProviderInterface
{
    protected $config;

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    public function getName(): string
    {
        return 'SMTP Connector';
    }

    public function getUniqueId(): string
    {
        return 'smtp_connector';
    }

    public function isConfigured(): bool
    {
        return !empty($this->config['smtp_host']) && !empty($this->config['smtp_username']);
    }

    public function send(MailLog $mail): array
    {
        try {
            require_once __DIR__ . '/../PHPMailer/PHPMailer.php';
            require_once __DIR__ . '/../PHPMailer/SMTP.php';
            require_once __DIR__ . '/../PHPMailer/Exception.php';

            $mailer = new \PHPMailer\PHPMailer\PHPMailer(true);
            $mailer->isSMTP();
            $mailer->Host = $this->config['smtp_host'];
            $mailer->SMTPAuth = true;
            $mailer->Username = $this->config['smtp_username'];
            $mailer->Password = $this->config['smtp_password'];
            $mailer->SMTPSecure = $this->config['smtp_encryption'] ?? 'tls';
            $mailer->Port = $this->config['smtp_port'] ?? 587;
            $mailer->CharSet = 'UTF-8';

            $mailer->setFrom(
                $this->config['smtp_from_email'] ?? $this->config['smtp_username'],
                $this->config['smtp_from_name'] ?? ''
            );

            $mailer->addAddress($mail->recipient);
            $mailer->Subject = $mail->subject;
            $mailer->Body = $mail->body;
            $mailer->isHTML(true);

            if ($mail->cc) {
                $ccAddresses = explode(',', $mail->cc);
                foreach ($ccAddresses as $cc) {
                    $mailer->addCC(trim($cc));
                }
            }

            if ($mail->bcc) {
                $bccAddresses = explode(',', $mail->bcc);
                foreach ($bccAddresses as $bcc) {
                    $mailer->addBCC(trim($bcc));
                }
            }

            if (!empty($mail->attachments)) {
                foreach ($mail->attachments as $attachment) {
                    $mailer->addAttachment($attachment['path'], $attachment['name'] ?? '');
                }
            }

            $mailer->send();

            return [
                'status' => 'success',
                'message_id' => 'smtp-' . uniqid(),
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
            'smtp_host' => [
                'FriendlyName' => 'SMTP Host',
                'Type' => 'text',
                'Size' => '50',
            ],
            'smtp_port' => [
                'FriendlyName' => 'SMTP Port',
                'Type' => 'text',
                'Size' => '10',
                'Default' => '587',
            ],
            'smtp_username' => [
                'FriendlyName' => 'Username',
                'Type' => 'text',
                'Size' => '50',
            ],
            'smtp_password' => [
                'FriendlyName' => 'Password',
                'Type' => 'password',
                'Size' => '50',
            ],
            'smtp_encryption' => [
                'FriendlyName' => 'Encryption',
                'Type' => 'dropdown',
                'Options' => 'tls,ssl',
                'Default' => 'tls',
            ],
            'smtp_from_email' => [
                'FriendlyName' => 'From Email',
                'Type' => 'text',
                'Size' => '50',
            ],
            'smtp_from_name' => [
                'FriendlyName' => 'From Name',
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

function whmcs_smtp_connector_activate(): array
{
    return ['status' => 'success', 'description' => 'SMTP Connector module activated'];
}

function whmcs_smtp_connector_deactivate(): array
{
    return ['status' => 'success', 'description' => 'SMTP Connector module deactivated'];
}

function whmcs_smtp_connector_config(): array
{
    return SMTP_Connector_Provider::getConfigurationFields();
}
```

## Configuration File: config.php

```php
<?php
return [
    'smtp_host' => '',
    'smtp_port' => '587',
    'smtp_username' => '',
    'smtp_password' => '',
    'smtp_encryption' => 'tls',
    'smtp_from_email' => '',
    'smtp_from_name' => '',
];
```

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
- PHPMailer library
