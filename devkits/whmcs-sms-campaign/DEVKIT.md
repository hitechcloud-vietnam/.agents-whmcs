# WHMCS SMS Campaign DevKit

## Overview

SMS campaign management system for WHMCS enabling targeted SMS marketing, transactional SMS, automated notifications, and campaign analytics.

## Features

- SMS campaign creation
- Contact management
- Template system
- Scheduling
- Two-way messaging
- Delivery tracking
- Opt-out management
- Campaign analytics
- A/B testing
- Integration with SMS providers

## Module Files

```php
<?php
/**
 * WHMCS SMS Campaign Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/SmsCampaignManager.php';

function whmcs_sms_campaign_activate() {
    $manager = new SmsCampaignManager();
    return $manager->activate();
}

function whmcs_sms_campaign_send($userId, $message, $type = 'transactional') {
    $manager = new SmsCampaignManager();
    return $manager->sendSms($userId, $message, $type);
}

function whmcs_sms_campaign_create_campaign($data) {
    $manager = new SmsCampaignManager();
    return $manager->createCampaign($data);
}

function whmcs_sms_campaign_schedule($campaignId, $scheduleTime) {
    $manager = new SmsCampaignManager();
    return $manager->scheduleCampaign($campaignId, $scheduleTime);
}

function whmcs_sms_campaign_get_campaigns($status = null) {
    $manager = new SmsCampaignManager();
    return $manager->getCampaigns($status);
}

add_hook('DailyCronJob', 1, function() {
    $manager = new SmsCampaignManager();
    $manager->processScheduledCampaigns();
    $manager->processDeliveryReports();
});

add_hook('ServiceCreate', 1, function($params) {
    $manager = new SmsCampaignManager();
    $manager->sendSms($params['user_id'], 'Welcome to our service!', 'welcome');
});
```

### lib/SmsCampaignManager.php

```php
<?php
namespace WHMCS\Module\SmsCampaign;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class SmsCampaignManager {
    
    protected $provider = 'twilio';
    
    public function activate() {
        try {
            $this->createTables();
            $this->createDefaultTemplates();
            return ['success' => true, 'msg' => 'SMS Campaign module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_sms_campaigns` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `campaign_name` VARCHAR(255) NOT NULL,
                `campaign_code` VARCHAR(50) NOT NULL,
                `message_template` TEXT NOT NULL,
                `target_segment` JSON NULL,
                `status` ENUM('draft', 'scheduled', 'sending', 'completed', 'paused', 'cancelled') NOT NULL DEFAULT 'draft',
                `scheduled_at` DATETIME NULL,
                `sent_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `delivered_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `failed_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_sms_messages` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `campaign_id` INT UNSIGNED NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `phone_number` VARCHAR(20) NOT NULL,
                `message` TEXT NOT NULL,
                `message_type` ENUM('campaign', 'transactional', 'notification', 'alert') NOT NULL DEFAULT 'transactional',
                `status` ENUM('pending', 'sent', 'delivered', 'failed', 'undelivered') NOT NULL DEFAULT 'pending',
                `provider_message_id` VARCHAR(255) NULL,
                `sent_at` DATETIME NULL,
                `delivered_at` DATETIME NULL,
                `error_code` VARCHAR(50) NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_sms_templates` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `template_name` VARCHAR(255) NOT NULL,
                `template_code` VARCHAR(50) NOT NULL,
                `message_content` TEXT NOT NULL,
                `variables` JSON NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_sms_optouts` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NULL,
                `phone_number` VARCHAR(20) NOT NULL,
                `optout_reason` VARCHAR(255) NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_phone` (`phone_number`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function createDefaultTemplates() {
        $templates = [
            ['name' => 'Welcome Message', 'code' => 'WELCOME', 'content' => 'Welcome {firstname}! Thank you for joining {company}'],
            ['name' => 'Payment Reminder', 'code' => 'PAYMENT_REMINDER', 'content' => 'Reminder: Invoice #{invoice_num} for {amount} is due on {due_date}'],
            ['name' => 'Service Activated', 'code' => 'SERVICE_ACTIVE', 'content' => 'Great news! Your {service_name} service is now active'],
        ];
        
        foreach ($templates as $template) {
            if (!Capsule::table('mod_sms_templates')->where('template_code', $template['code'])->exists()) {
                Capsule::table('mod_sms_templates')->insert($template);
            }
        }
    }
    
    public function sendSms($userId, $message, $type = 'transactional') {
        $client = Capsule::table('tblclients')->where('id', $userId)->first();
        
        if (!$client || empty($client->phonenumber)) {
            return ['success' => false, 'msg' => 'Invalid phone number'];
        }
        
        // Check opt-out
        if ($this->isOptedOut($client->phonenumber)) {
            return ['success' => false, 'msg' => 'Phone number opted out'];
        }
        
        $messageId = Capsule::table('mod_sms_messages')->insertGetId([
            'user_id' => $userId,
            'phone_number' => $client->phonenumber,
            'message' => $message,
            'message_type' => $type,
            'status' => 'pending',
        ]);
        
        $result = $this->sendToProvider($client->phonenumber, $message);
        
        if ($result['success']) {
            Capsule::table('mod_sms_messages')
                ->where('id', $messageId)
                ->update([
                    'status' => 'sent',
                    'provider_message_id' => $result['message_id'],
                    'sent_at' => Carbon::now(),
                ]);
        } else {
            Capsule::table('mod_sms_messages')
                ->where('id', $messageId)
                ->update([
                    'status' => 'failed',
                    'error_code' => $result['error'],
                ]);
        }
        
        return $result;
    }
    
    protected function sendToProvider($phoneNumber, $message) {
        // Integrate with SMS provider (Twilio, etc.)
        // This is a placeholder for actual implementation
        return [
            'success' => true,
            'message_id' => 'MSG-' . uniqid(),
        ];
    }
    
    protected function isOptedOut($phoneNumber) {
        return Capsule::table('mod_sms_optouts')
            ->where('phone_number', $phoneNumber)
            ->exists();
    }
    
    public function createCampaign($data) {
        $campaignId = Capsule::table('mod_sms_campaigns')->insertGetId([
            'campaign_name' => $data['name'],
            'campaign_code' => 'CAMP-' . strtoupper(substr(uniqid(), -6)),
            'message_template' => $data['message'],
            'target_segment' => json_encode($data['segment'] ?? []),
            'status' => 'draft',
        ]);
        
        return ['success' => true, 'campaign_id' => $campaignId];
    }
    
    public function scheduleCampaign($campaignId, $scheduleTime) {
        Capsule::table('mod_sms_campaigns')
            ->where('id', $campaignId)
            ->update([
                'status' => 'scheduled',
                'scheduled_at' => $scheduleTime,
            ]);
        
        return ['success' => true];
    }
    
    public function processScheduledCampaigns() {
        $campaigns = Capsule::table('mod_sms_campaigns')
            ->where('status', 'scheduled')
            ->where('scheduled_at', '<=', Carbon::now())
            ->get();
        
        foreach ($campaigns as $campaign) {
            $this->sendCampaign($campaign->id);
        }
    }
    
    public function sendCampaign($campaignId) {
        $campaign = Capsule::table('mod_sms_campaigns')->where('id', $campaignId)->first();
        
        Capsule::table('mod_sms_campaigns')
            ->where('id', $campaignId)
            ->update(['status' => 'sending']);
        
        $recipients = $this->getCampaignRecipients($campaign);
        
        foreach ($recipients as $recipient) {
            $this->sendSms($recipient->id, $campaign->message_template, 'campaign');
            Capsule::table('mod_sms_campaigns')
                ->where('id', $campaignId)
                ->increment('sent_count');
        }
        
        Capsule::table('mod_sms_campaigns')
            ->where('id', $campaignId)
            ->update(['status' => 'completed']);
        
        return ['success' => true, 'sent' => count($recipients)];
    }
    
    protected function getCampaignRecipients($campaign) {
        $segment = json_decode($campaign->target_segment, true) ?? [];
        
        $query = Capsule::table('tblclients')
            ->whereNotNull('phonenumber')
            ->where('phonenumber', '!=', '');
        
        if (!empty($segment['min_spend'])) {
            $query->where('total_amount', '>=', $segment['min_spend']);
        }
        
        if (!empty($segment['max_spend'])) {
            $query->where('total_amount', '<=', $segment['max_spend']);
        }
        
        if (!empty($segment['status'])) {
            $query->where('status', $segment['status']);
        }
        
        return $query->get();
    }
    
    public function getCampaigns($status = null) {
        $query = Capsule::table('mod_sms_campaigns');
        
        if ($status) {
            $query->where('status', $status);
        }
        
        return $query->orderBy('created_at', 'desc')->get();
    }
    
    public function handleOptOut($phoneNumber, $reason = null) {
        Capsule::table('mod_sms_optouts')->insert([
            'phone_number' => $phoneNumber,
            'optout_reason' => $reason,
        ]);
        
        return ['success' => true];
    }
}
```

## API Endpoints

```
POST /api/v1/sms/send                  - Send SMS
POST /api/v1/sms/campaign              - Create campaign
POST /api/v1/sms/campaign/{id}/send     - Send campaign
POST /api/v1/sms/campaign/{id}/schedule - Schedule campaign
GET  /api/v1/sms/campaigns              - List campaigns
GET  /api/v1/sms/campaigns/{id}         - Get campaign stats
POST /api/v1/sms/optout                - Opt-out from SMS
GET  /api/v1/sms/templates             - List templates
```
