# WHMCS Email Tracking Mail Module

## Overview
Email open/click tracking module for WHMCS.

## Module File: email_tracking.php

```php
<?php
/**
 * WHMCS Email Tracking Mail Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Mail\Contract\ProviderInterface;
use WHMCS\Mail\Log as MailLog;

class Email_Tracking_Mail_Provider implements ProviderInterface
{
    protected $config;

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    public function getName(): string
    {
        return 'Email with Tracking';
    }

    public function getUniqueId(): string
    {
        return 'email_tracking';
    }

    public function isConfigured(): bool
    {
        return true; // Uses default mailer with tracking
    }

    public function send(MailLog $mail): array
    {
        try {
            // Inject tracking pixels and links
            $modifiedBody = $this->injectTracking($mail);
            
            // Store tracking record
            $trackingId = $this->createTrackingRecord($mail);

            // Replace body with modified version
            $mail->body = $modifiedBody;

            // Send via default mailer
            $result = $this->sendViaDefault($mail);

            return $result;

        } catch (\Exception $e) {
            return ['status' => 'error', 'error' => $e->getMessage()];
        }
    }

    protected function injectTracking(MailLog $mail): string
    {
        $trackingId = uniqid('t_');
        $baseUrl = $this->config['tracking_base_url'] ?? rtrim(\App::getSystemURL(), '/');
        
        // Add tracking pixel
        $pixel = '<img src="' . $baseUrl . '/modules/addons/email_tracking/pixel.php?id=' . $trackingId . '" width="1" height="1" style="display:none" />';
        
        // Replace URLs with tracked URLs
        $pattern = '/href=["\']([^"\']+)["\']/i';
        $modifiedBody = preg_replace_callback($pattern, function($matches) use ($baseUrl, $trackingId) {
            $url = $matches[1];
            if (strpos($url, '://') !== false || strpos($url, '/') === 0) {
                $encodedUrl = urlencode($url);
                return 'href="' . $baseUrl . '/modules/addons/email_tracking/click.php?id=' . $trackingId . '&url=' . $encodedUrl . '"';
            }
            return $matches[0];
        }, $mail->body);

        return $pixel . $modifiedBody;
    }

    protected function createTrackingRecord(MailLog $mail): string
    {
        $trackingId = 't_' . uniqid();
        $now = date('Y-m-d H:i:s');

        Capsule::table('mod_email_tracking')->insert([
            'tracking_id' => $trackingId,
            'recipient' => $mail->recipient,
            'subject' => $mail->subject,
            'sent_at' => $now,
            'opens' => 0,
            'clicks' => 0,
            'created_at' => $now,
        ]);

        return $trackingId;
    }

    protected function sendViaDefault(MailLog $mail): array
    {
        // Use default WHMCS mailer
        $mailer = new WHMCS\Mail\Mailer();
        $result = $mailer->send(
            new WHMCS\Mail\Log($mail->recipient, $mail->subject, $mail->body)
        );

        return [
            'status' => 'success',
            'message_id' => 'tracked-' . uniqid(),
        ];
    }

    public static function getConfigurationFields(): array
    {
        return [
            'tracking_base_url' => [
                'FriendlyName' => 'Tracking Base URL',
                'Type' => 'text',
                'Size' => '50',
                'Description' => 'Base URL for tracking pixel and click redirects',
            ],
            'enable_open_tracking' => [
                'FriendlyName' => 'Track Opens',
                'Type' => 'yesno',
                'Description' => 'Track when emails are opened',
            ],
            'enable_click_tracking' => [
                'FriendlyName' => 'Track Clicks',
                'Type' => 'yesno',
                'Description' => 'Track when links are clicked',
            ],
        ];
    }
}

// Database schema for tracking
function whmcs_email_tracking_mail_activate(): array
{
    try {
        if (!Capsule::schema()->hasTable('mod_email_tracking')) {
            Capsule::schema()->create('mod_email_tracking', function ($table) {
                $table->increments('id');
                $table->string('tracking_id', 50)->unique();
                $table->string('recipient', 255);
                $table->string('subject', 500);
                $table->string('message_id', 255)->nullable();
                $table->integer('opens')->default(0);
                $table->integer('clicks')->default(0);
                $table->timestamp('sent_at');
                $table->timestamp('last_open_at')->nullable();
                $table->timestamp('last_click_at')->nullable();
                $table->timestamp('created_at')->useCurrent();
            });
        }

        if (!Capsule::schema()->hasTable('mod_email_tracking_events')) {
            Capsule::schema()->create('mod_email_tracking_events', function ($table) {
                $table->increments('id');
                $table->string('tracking_id', 50);
                $table->enum('event_type', ['open', 'click']);
                $table->string('ip_address', 45)->nullable();
                $table->string('user_agent', 500)->nullable();
                $table->string('click_url', 500)->nullable();
                $table->timestamp('occurred_at');
                
                $table->index('tracking_id');
            });
        }

        return ['status' => 'success', 'description' => 'Email Tracking module activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_email_tracking_mail_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Email Tracking Mail module deactivated'];
}

function whmcs_email_tracking_mail_config(): array
{
    return Email_Tracking_Mail_Provider::getConfigurationFields();
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
    'tracking_base_url' => '',
    'enable_open_tracking' => true,
    'enable_click_tracking' => true,
];
```
