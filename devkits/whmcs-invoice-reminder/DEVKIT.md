# WHMCS Invoice Reminder Module

```php
<?php
/**
 * WHMCS Invoice Reminder Module
 * 
 * Automated invoice reminder scheduler with customizable
 * reminder templates and delivery tracking.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function invoicereminder_MetaData() {
    return array('DisplayName' => 'Invoice Reminder', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function invoicereminder_ConfigArray() {
    return array('FriendlyName' => array('Type' => 'System', 'Value' => 'Invoice Reminder'),
        'EnableReminders' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable invoice reminders'),
        'EnableOverdue' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable overdue notices'),
        'FirstReminderDays' => array('Type' => 'text', 'Size' => '10', 'Default' => '7', 'Description' => 'Days before due'),
        'SecondReminderDays' => array('Type' => 'text', 'Size' => '10', 'Default' => '3', 'Description' => 'Days before due'),
        'OverdueNoticeDays' => array('Type' => 'text', 'Size' => '10', 'Default' => '1,3,7', 'Description' => 'Days after overdue (comma-separated)'),
        'MaxReminders' => array('Type' => 'text', 'Size' => '10', 'Default' => '5', 'Description' => 'Max reminders per invoice'));
}

function invoicereminder_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_invoicereminder_templates', "
            CREATE TABLE `mod_invoicereminder_templates` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `template_key` VARCHAR(100) UNIQUE NOT NULL,
                `name` VARCHAR(255) NOT NULL,
                `reminder_type` ENUM('reminder', 'overdue', 'final') NOT NULL,
                `days_offset` INT NOT NULL,
                `email_subject` VARCHAR(255) NOT NULL,
                `email_body` TEXT NOT NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_invoicereminder_sent', "
            CREATE TABLE `mod_invoicereminder_sent` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `invoice_id` INT NOT NULL,
                `template_id` INT NOT NULL,
                `sent_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `status` ENUM('sent', 'failed', 'skipped') DEFAULT 'sent',
                `error` TEXT NULL,
                INDEX `idx_invoice_id` (`invoice_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Invoice Reminder module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function invoicereminder_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function invoicereminder_CreateTemplate($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $key = 'tmpl-' . substr(md5(uniqid()), 0, 10);
        Capsule::table('mod_invoicereminder_templates')->insert(array('template_key' => $key, 'name' => $data['name'], 'reminder_type' => $data['reminder_type'], 'days_offset' => $data['days_offset'], 'email_subject' => $data['subject'], 'email_body' => $data['body']));
        return array('success' => true, 'template_key' => $key);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function invoicereminder_ProcessReminders() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $templates = Capsule::table('mod_invoicereminder_templates')->where('is_active', 1)->orderBy('days_offset', 'asc')->get();
    $processed = 0;
    foreach ($templates as $template) {
        $invoices = invoicereminder_GetInvoicesForTemplate($template);
        foreach ($invoices as $invoice) {
            if (invoicereminder_ShouldSend($invoice->id, $template->id)) {
                invoicereminder_SendReminder($invoice, $template);
                $processed++;
            }
        }
    }
    return $processed;
}

function invoicereminder_GetInvoicesForTemplate($template) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $now = date('Y-m-d');
    $targetDate = date('Y-m-d', strtotime("+{$template->days_offset} days"));
    if ($template->reminder_type === 'overdue') {
        return Capsule::table('tblinvoices')->where('status', 'Unpaid')->where('duedate', '<', $now)->where('duedate', '>=', date('Y-m-d', strtotime("-{$template->days_offset} days")))->get();
    }
    return Capsule::table('tblinvoices')->where('status', 'Unpaid')->where('duedate', $targetDate)->get();
}

function invoicereminder_ShouldSend($invoiceId, $templateId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $existing = Capsule::table('mod_invoicereminder_sent')->where('invoice_id', $invoiceId)->where('template_id', $templateId)->first();
    return !$existing;
}

function invoicereminder_SendReminder($invoice, $template) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $client = Capsule::table('tblclients')->where('id', $invoice->userid)->first();
        $subject = str_replace(array('{invoice_num}', '{due_date}', '{amount}'), array($invoice->id, $invoice->duedate, $invoice->total), $template->email_subject);
        $body = str_replace(array('{first_name}', '{last_name}', '{invoice_num}', '{due_date}', '{amount}'), array($client->firstname ?? '', $client->lastname ?? '', $invoice->id, $invoice->duedate, $invoice->total), $template->email_body);
        sendEmail($client->email, $subject, $body);
        Capsule::table('mod_invoicereminder_sent')->insert(array('invoice_id' => $invoice->id, 'template_id' => $template->id, 'status' => 'sent'));
        return true;
    } catch (\Exception $e) {
        Capsule::table('mod_invoicereminder_sent')->insert(array('invoice_id' => $invoice->id, 'template_id' => $template->id, 'status' => 'failed', 'error' => $e->getMessage()));
        return false;
    }
}

function invoicereminder_GetSentReminders($invoiceId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_invoicereminder_sent')->where('invoice_id', $invoiceId)->get();
}

function invoicereminder_GetStatistics($days = 30) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $since = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    $stats = Capsule::table('mod_invoicereminder_sent')->where('sent_at', '>=', $since)->selectRaw("COUNT(*) as total, SUM(CASE WHEN status='sent' THEN 1 ELSE 0 END) as sent, SUM(CASE WHEN status='failed' THEN 1 ELSE 0 END) as failed")->first();
    return array('total' => (int)$stats->total, 'sent' => (int)$stats->sent, 'failed' => (int)$stats->failed);
}
