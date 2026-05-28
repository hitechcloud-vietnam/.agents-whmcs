# WHMCS Email Automation Module

```php
<?php
/**
 * WHMCS Email Automation Module
 * 
 * Creates automated email sequences triggered by events,
 * with scheduling, personalization, and tracking.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function emailautomation_MetaData() {
    return array('DisplayName' => 'Email Automation', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function emailautomation_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Email Automation'),
        'EnableSequences' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable email sequences'),
        'MaxEmailsPerDay' => array('Type' => 'text', 'Size' => '10', 'Default' => '100', 'Description' => 'Max emails per day per client'),
        'EnableTracking' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Track email opens and clicks'),
        'RetryFailed' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Retry failed emails'),
        'MaxRetries' => array('Type' => 'text', 'Size' => '10', 'Default' => '3', 'Description' => 'Max retry attempts'),
    );
}

function emailautomation_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        
        createTable('mod_emailautomation_sequences', "
            CREATE TABLE `mod_emailautomation_sequences` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `sequence_key` VARCHAR(100) UNIQUE NOT NULL,
                `name` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `trigger_event` VARCHAR(100) NOT NULL,
                `emails` JSON NOT NULL,
                `conditions` JSON NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        
        createTable('mod_emailautomation_enrollments', "
            CREATE TABLE `mod_emailautomation_enrollments` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `sequence_id` INT NOT NULL,
                `user_id` INT NOT NULL,
                `rel_id` INT NULL,
                `status` ENUM('active', 'completed', 'paused', 'cancelled') DEFAULT 'active',
                `current_step` INT DEFAULT 0,
                `started_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `completed_at` DATETIME NULL,
                `unsubscribed` TINYINT(1) DEFAULT 0,
                UNIQUE KEY `unique_enrollment` (`sequence_id`, `user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        
        createTable('mod_emailautomation_sent', "
            CREATE TABLE `mod_emailautomation_sent` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `enrollment_id` INT NOT NULL,
                `email_index` INT NOT NULL,
                `subject` VARCHAR(255) NOT NULL,
                `sent_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `opened_at` DATETIME NULL,
                `clicked_at` DATETIME NULL,
                `status` ENUM('pending', 'sent', 'opened', 'clicked', 'failed', 'bounced') DEFAULT 'pending',
                `attempts` INT DEFAULT 0,
                `error` TEXT NULL
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        
        return array('status' => 'success', 'description' => 'Email Automation module activated.');
    } catch (\Exception $e) {
        return array('status' => 'error', 'description' => 'Failed to activate: ' . $e->getMessage());
    }
}

function emailautomation_deactivate() {
    return array('status' => 'success', 'description' => 'Module deactivated.');
}

function emailautomation_CreateSequence($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $key = 'seq-' . substr(md5(uniqid()), 0, 12);
        Capsule::table('mod_emailautomation_sequences')->insert(array(
            'sequence_key' => $key, 'name' => $data['name'], 'description' => $data['description'] ?? '',
            'trigger_event' => $data['trigger_event'], 'emails' => json_encode($data['emails']),
            'conditions' => isset($data['conditions']) ? json_encode($data['conditions']) : null
        ));
        return array('success' => true, 'sequence_key' => $key);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function emailautomation_GetSequence($key) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $seq = Capsule::table('mod_emailautomation_sequences')->where('sequence_key', $key)->first();
    if ($seq) { $seq->emails = json_decode($seq->emails, true); $seq->conditions = json_decode($seq->conditions, true); }
    return $seq;
}

function emailautomation_UpdateSequence($key, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $update = array_filter(array('name' => $data['name'] ?? null, 'description' => $data['description'] ?? null,
            'emails' => isset($data['emails']) ? json_encode($data['emails']) : null,
            'conditions' => isset($data['conditions']) ? json_encode($data['conditions']) : null,
            'is_active' => isset($data['is_active']) ? $data['is_active'] : null), function($v) { return $v !== null; });
        Capsule::table('mod_emailautomation_sequences')->where('sequence_key', $key)->update($update);
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function emailautomation_EnrollUser($sequenceKey, $userId, $relId = null, $customData = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $sequence = emailautomation_GetSequence($sequenceKey);
        if (!$sequence || !$sequence->is_active) { return array('success' => false, 'error' => 'Sequence not found or inactive'); }
        
        Capsule::table('mod_emailautomation_enrollments')->insert(array(
            'sequence_id' => $sequence->id, 'user_id' => $userId, 'rel_id' => $relId, 'status' => 'active'
        ));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function emailautomation_ProcessSequences() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $enrollments = Capsule::table('mod_emailautomation_enrollments')->where('status', 'active')->where('unsubscribed', 0)->get();
    $processed = 0;
    foreach ($enrollments as $enrollment) {
        $sequence = Capsule::table('mod_emailautomation_sequences')->where('id', $enrollment->sequence_id)->first();
        if (!$sequence || !$sequence->is_active) continue;
        $emails = json_decode($sequence->emails, true);
        if ($enrollment->current_step >= count($emails)) {
            Capsule::table('mod_emailautomation_enrollments')->where('id', $enrollment->id)->update(array('status' => 'completed', 'completed_at' => date('Y-m-d H:i:s')));
            continue;
        }
        $currentEmail = $emails[$enrollment->current_step];
        $delayHours = $currentEmail['delay_hours'] ?? 0;
        $enrolledAt = new DateTime($enrollment->started_at);
        $shouldSendAt = $enrolledAt->modify("+{$delayHours} hours");
        if (new DateTime() >= $shouldSendAt) {
            emailautomation_SendEmail($enrollment, $currentEmail);
            Capsule::table('mod_emailautomation_enrollments')->where('id', $enrollment->id)->update(array('current_step' => $enrollment->current_step + 1));
            $processed++;
        }
    }
    return $processed;
}

function emailautomation_SendEmail($enrollment, $emailData) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $client = Capsule::table('tblclients')->where('id', $enrollment->user_id)->first();
    if (!$client) return;
    $subject = $emailData['subject'];
    $body = $emailData['body'];
    $subject = str_replace(array('{first_name}', '{last_name}', '{email}'), array($client->firstname, $client->lastname, $client->email), $subject);
    $body = str_replace(array('{first_name}', '{last_name}', '{email}'), array($client->firstname, $client->lastname, $client->email), $body);
    Capsule::table('mod_emailautomation_sent')->insert(array(
        'enrollment_id' => $enrollment->id, 'email_index' => $enrollment->current_step,
        'subject' => $subject, 'status' => 'pending'
    ));
    sendEmail($client->email, $subject, $body);
}

function emailautomation_GetEnrollments($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_emailautomation_enrollments')->where('user_id', $userId)->get();
}

function emailautomation_Unenroll($sequenceKey, $userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $sequence = emailautomation_GetSequence($sequenceKey);
    if ($sequence) {
        Capsule::table('mod_emailautomation_enrollments')->where('sequence_id', $sequence->id)->where('user_id', $userId)->update(array('unsubscribed' => 1, 'status' => 'cancelled'));
    }
}

function emailautomation_TriggerEvent($eventType, $userId, $relId = null, $data = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $sequences = Capsule::table('mod_emailautomation_sequences')->where('trigger_event', $eventType)->where('is_active', 1)->get();
    foreach ($sequences as $sequence) {
        emailautomation_EnrollUser($sequence->sequence_key, $userId, $relId, $data);
    }
}
