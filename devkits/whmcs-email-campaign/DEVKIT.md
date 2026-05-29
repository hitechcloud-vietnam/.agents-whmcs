# WHMCS Email Campaign DevKit

## Overview

Email campaign management system for WHMCS with drag-drop email builder, contact management, campaign automation, delivery tracking, and analytics.

## Features

- Email campaign builder
- Template management
- Contact segmentation
- Automated workflows
- A/B testing
- Delivery tracking
- Unsubscribe management
- Analytics dashboard
- Personalization tokens
- ISP deliverability

## Module Files

```php
<?php
/**
 * WHMCS Email Campaign Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/EmailCampaignManager.php';

function whmcs_email_campaign_activate() {
    $manager = new EmailCampaignManager();
    return $manager->activate();
}

function whmcs_email_campaign_create_campaign($data) {
    $manager = new EmailCampaignManager();
    return $manager->createCampaign($data);
}

function whmcs_email_campaign_send($campaignId) {
    $manager = new EmailCampaignManager();
    return $manager->sendCampaign($campaignId);
}

function whmcs_email_campaign_schedule($campaignId, $scheduleTime) {
    $manager = new EmailCampaignManager();
    return $manager->scheduleCampaign($campaignId, $scheduleTime);
}

function whmcs_email_campaign_get_analytics($campaignId) {
    $manager = new EmailCampaignManager();
    return $manager->getAnalytics($campaignId);
}

add_hook('DailyCronJob', 1, function() {
    $manager = new EmailCampaignManager();
    $manager->processScheduledCampaigns();
    $manager->processBounces();
    $manager->updateAnalytics();
});
```

### lib/EmailCampaignManager.php

```php
<?php
namespace WHMCS\Module\EmailCampaign;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class EmailCampaignManager {
    
    public function activate() {
        try {
            $this->createTables();
            $this->createDefaultTemplates();
            return ['success' => true, 'msg' => 'Email Campaign module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_email_campaigns` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `campaign_name` VARCHAR(255) NOT NULL,
                `campaign_code` VARCHAR(50) NOT NULL,
                `subject` VARCHAR(255) NOT NULL,
                `preheader` VARCHAR(255) NULL,
                `content` LONGTEXT NOT NULL,
                `target_segment` JSON NULL,
                `status` ENUM('draft', 'scheduled', 'sending', 'sent', 'paused', 'cancelled') NOT NULL DEFAULT 'draft',
                `scheduled_at` DATETIME NULL,
                `sent_at` DATETIME NULL,
                `total_recipients` INT UNSIGNED NOT NULL DEFAULT 0,
                `sent_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `delivered_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `opened_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `clicked_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `bounced_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `unsubscribed_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_email_recipients` (
                `id` BIGINT/UNSIGNED NOT NULL AUTO_INCREMENT,
                `campaign_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `email` VARCHAR(255) NOT NULL,
                `status` ENUM('pending', 'sent', 'delivered', 'opened', 'clicked', 'bounced', 'unsubscribed') NOT NULL DEFAULT 'pending',
                `sent_at` DATETIME NULL,
                `opened_at` DATETIME NULL,
                `clicked_at` DATETIME NULL,
                `bounced_at` DATETIME NULL,
                `provider_message_id` VARCHAR(255) NULL,
                PRIMARY KEY (`id`),
                INDEX `idx_campaign_status` (`campaign_id`, `status`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_email_templates` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `template_name` VARCHAR(255) NOT NULL,
                `template_code` VARCHAR(50) NOT NULL,
                `subject` VARCHAR(255) NOT NULL,
                `content` LONGTEXT NOT NULL,
                `variables` JSON NULL,
                `category` VARCHAR(100) NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_email_unsubscribes` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NULL,
                `email` VARCHAR(255) NOT NULL,
                `campaign_id` INT UNSIGNED NULL,
                `reason` VARCHAR(255) NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function createDefaultTemplates() {
        $templates = [
            [
                'name' => 'Newsletter Standard',
                'code' => 'NEWSLETTER_STD',
                'subject' => '{company_name} Newsletter - {month}',
                'content' => '<h1>Hello {firstname}</h1><p>Welcome to our newsletter...</p>',
            ],
            [
                'name' => 'Promotional Offer',
                'code' => 'PROMO',
                'subject' => 'Special Offer for You, {firstname}!',
                'content' => '<h1>Great News!</h1><p>Get {discount}% off...</p>',
            ],
        ];
        
        foreach ($templates as $template) {
            if (!Capsule::table('mod_email_templates')->where('template_code', $template['code'])->exists()) {
                Capsule::table('mod_email_templates')->insert($template);
            }
        }
    }
    
    public function createCampaign($data) {
        $campaignId = Capsule::table('mod_email_campaigns')->insertGetId([
            'campaign_name' => $data['name'],
            'campaign_code' => 'ECAMP-' . strtoupper(substr(uniqid(), -6)),
            'subject' => $data['subject'],
            'preheader' => $data['preheader'] ?? null,
            'content' => $data['content'],
            'target_segment' => json_encode($data['segment'] ?? []),
            'status' => 'draft',
        ]);
        
        return ['success' => true, 'campaign_id' => $campaignId];
    }
    
    public function scheduleCampaign($campaignId, $scheduleTime) {
        Capsule::table('mod_email_campaigns')
            ->where('id', $campaignId)
            ->update([
                'status' => 'scheduled',
                'scheduled_at' => $scheduleTime,
            ]);
        
        return ['success' => true];
    }
    
    public function sendCampaign($campaignId) {
        $campaign = Capsule::table('mod_email_campaigns')->where('id', $campaignId)->first();
        
        Capsule::table('mod_email_campaigns')
            ->where('id', $campaignId)
            ->update(['status' => 'sending', 'sent_at' => Carbon::now()]);
        
        $recipients = $this->prepareRecipients($campaign);
        
        foreach ($recipients as $recipient) {
            $this->sendEmail($campaign, $recipient);
        }
        
        Capsule::table('mod_email_campaigns')
            ->where('id', $campaignId)
            ->increment('sent_count', count($recipients));
        
        return ['success' => true, 'recipients' => count($recipients)];
    }
    
    protected function prepareRecipients($campaign) {
        $segment = json_decode($campaign->target_segment, true) ?? [];
        
        $query = Capsule::table('tblclients')
            ->where('email', '!=', '')
            ->where(function($q) {
                $q->whereNotExists(function($sq) {
                    $sq->select('id')->from('mod_email_unsubscribes');
                });
            });
        
        if (!empty($segment['min_spend'])) {
            $query->where('total_amount', '>=', $segment['min_spend']);
        }
        
        return $query->get();
    }
    
    protected function sendEmail($campaign, $recipient) {
        $personalizedContent = $this->personalizeContent($campaign->content, $recipient);
        $personalizedSubject = $this->personalizeContent($campaign->subject, $recipient);
        
        $messageId = Capsule::table('mod_email_recipients')->insertGetId([
            'campaign_id' => $campaign->id,
            'user_id' => $recipient->id,
            'email' => $recipient->email,
            'status' => 'pending',
        ]);
        
        // Send email via WHMCS mail system
        sendEmail($campaign->campaign_code, $recipient->email, [
            'subject' => $personalizedSubject,
            'message' => $personalizedContent,
        ]);
        
        Capsule::table('mod_email_recipients')
            ->where('id', $messageId)
            ->update(['status' => 'sent', 'sent_at' => Carbon::now()]);
    }
    
    protected function personalizeContent($content, $recipient) {
        $replacements = [
            '{firstname}' => $recipient->firstname ?? '',
            '{lastname}' => $recipient->lastname ?? '',
            '{email}' => $recipient->email ?? '',
            '{company}' =>Capsule::table('tblconfiguration')->where('setting', 'CompanyName')->value('value') ?? '',
            '{month}' => Carbon::now()->format('F Y'),
        ];
        
        return str_replace(array_keys($replacements), array_values($replacements), $content);
    }
    
    public function processScheduledCampaigns() {
        $campaigns = Capsule::table('mod_email_campaigns')
            ->where('status', 'scheduled')
            ->where('scheduled_at', '<=', Carbon::now())
            ->get();
        
        foreach ($campaigns as $campaign) {
            $this->sendCampaign($campaign->id);
        }
    }
    
    public function getAnalytics($campaignId) {
        $campaign = Capsule::table('mod_email_campaigns')->where('id', $campaignId)->first();
        
        if (!$campaign) {
            return null;
        }
        
        $total = $campaign->sent_count;
        
        return [
            'campaign' => $campaign->campaign_name,
            'sent' => $campaign->sent_count,
            'delivered' => $campaign->delivered_count,
            'opened' => $campaign->opened_count,
            'clicked' => $campaign->clicked_count,
            'bounced' => $campaign->bounced_count,
            'unsubscribed' => $campaign->unsubscribed_count,
            'delivery_rate' => $total > 0 ? ($campaign->delivered_count / $total) * 100 : 0,
            'open_rate' => $total > 0 ? ($campaign->opened_count / $campaign->delivered_count) * 100 : 0,
            'click_rate' => $total > 0 ? ($campaign->clicked_count / $campaign->opened_count) * 100 : 0,
        ];
    }
    
    public function handleUnsubscribe($userId, $campaignId = null) {
        $client = Capsule::table('tblclients')->where('id', $userId)->first();
        
        if ($client) {
            Capsule::table('mod_email_unsubscribes')->insert([
                'user_id' => $userId,
                'email' => $client->email,
                'campaign_id' => $campaignId,
            ]);
        }
        
        return ['success' => true];
    }
}
```

## API Endpoints

```
POST /api/v1/email-campaign/campaign     - Create campaign
POST /api/v1/email-campaign/{id}/send   - Send campaign
POST /api/v1/email-campaign/{id}/schedule - Schedule campaign
GET  /api/v1/email-campaign/campaigns   - List campaigns
GET  /api/v1/email-campaign/{id}/analytics - Get analytics
GET  /api/v1/email-campaign/templates   - List templates
POST /api/v1/email-campaign/unsubscribe  - Handle unsubscribe
```
