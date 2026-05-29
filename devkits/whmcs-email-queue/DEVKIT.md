# WHMCS Email Queue Module - DEVKIT

## Module Information
- **Name**: Email Queue
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Queue and batch email sending for better performance

## Installation
1. Copy to `/modules/addons/email_queue/`
2. Activate via WHMCS Admin

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('DailyCronJob', 1, function($vars) {
    EmailQueue::processQueue();
});
```

### includes/EmailQueue.php
```php
<?php
if (!defined("WHMCS")) {
    die("this file cannot be accessed directly");
}

class EmailQueue
{
    private static $table = 'mod_email_queue';
    
    public static function add($to, $subject, $body, $attachments = [], $priority = 'normal')
    {
        $data = json_encode([
            'to' => $to,
            'subject' => $subject,
            'body' => $body,
            'attachments' => $attachments
        ]);
        
        $priorityValues = ['low' => 0, 'normal' => 1, 'high' => 2];
        
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$table . "
            (email_data, priority, status, created_at, scheduled_for)
            VALUES (
                '" . db_escape_string($data) . "',
                " . ($priorityValues[$priority] ?? 1) . ",
                'queued',
                NOW(),
                NOW()
            )
        ");
        
        return mysql_insert_id();
    }
    
    public static function processQueue()
    {
        $batchSize = 50;
        
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$table . "
            WHERE status = 'queued'
            AND scheduled_for <= NOW()
            ORDER BY priority DESC, created_at ASC
            LIMIT " . (int)$batchSize
        ");
        
        while ($email = mysql_fetch_array($result)) {
            self::sendEmail($email);
            
            full_query("
                UPDATE " . TABLE_PREFIX . self::$table . "
                SET status = 'sent', sent_at = NOW()
                WHERE id = " . (int)$email['id']
            );
        }
        
        // Clean old sent emails
        full_query("
            DELETE FROM " . TABLE_PREFIX . self::$table . "
            WHERE status = 'sent'
            AND sent_at < DATE_SUB(NOW(), INTERVAL 7 DAY)
        ");
    }
    
    private static function sendEmail($emailData)
    {
        $data = json_decode($emailData['email_data'], true);
        
        $to = $data['to'];
        $subject = $data['subject'];
        $body = $data['body'];
        $attachments = $data['attachments'] ?? [];
        
        mail($to, $subject, $body, "From: " . get_config('Email') . "\r\n");
    }
    
    public static function schedule($to, $subject, $body, $scheduledAt, $attachments = [])
    {
        $data = json_encode([
            'to' => $to,
            'subject' => $subject,
            'body' => $body,
            'attachments' => $attachments
        ]);
        
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$table . "
            (email_data, priority, status, created_at, scheduled_for)
            VALUES (
                '" . db_escape_string($data) . "',
                1,
                'queued',
                NOW(),
                '" . db_escape_string($scheduledAt) . "'
            )
        ");
    }
    
    public static function cancel($emailId)
    {
        full_query("
            UPDATE " . TABLE_PREFIX . self::$table . "
            SET status = 'cancelled'
            WHERE id = " . (int)$emailId . "
            AND status = 'queued'
        ");
    }
    
    public static function getStats()
    {
        $result = full_query("
            SELECT 
                COUNT(*) as total,
                SUM(CASE WHEN status = 'queued' THEN 1 ELSE 0 END) as queued,
                SUM(CASE WHEN status = 'sent' THEN 1 ELSE 0 END) as sent,
                SUM(CASE WHEN status = 'failed' THEN 1 ELSE 0 END) as failed
            FROM " . TABLE_PREFIX . self::$table
        );
        
        return mysql_fetch_array($result);
    }
}

function email_queue_activate()
{
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_email_queue (
            id INT AUTO_INCREMENT PRIMARY KEY,
            email_data TEXT NOT NULL,
            priority INT DEFAULT 1,
            status VARCHAR(20) DEFAULT 'queued',
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            scheduled_for DATETIME,
            sent_at DATETIME,
            error_message TEXT
        )
    ");
    
    full_query("CREATE INDEX idx_status ON " . TABLE_PREFIX . "mod_email_queue(status)");
    full_query("CREATE INDEX idx_scheduled ON " . TABLE_PREFIX . "mod_email_queue(scheduled_for)");
    
    return ['status' => 'success', 'description' => 'Email Queue activated'];
}

function email_queue_deactivate()
{
    return ['status' => 'success', 'description' => 'Email Queue deactivated'];
}

function email_queue_config()
{
    return [
        'name' => 'Email Queue',
        'description' => 'Queue and batch email sending',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'batch_size' => ['Type' => 'text', 'FriendlyName' => 'Batch Size', 'Default' => '50'],
            'max_retries' => ['Type' => 'text', 'FriendlyName' => 'Max Retries', 'Default' => '3']
        ]
    ];
}
```