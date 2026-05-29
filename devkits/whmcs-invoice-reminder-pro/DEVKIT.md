# WHMCS Invoice Reminder Pro Module - DEVKIT

## Module Information
- **Name**: Invoice Reminder Pro
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Advanced invoice reminder system with customizable templates

## Installation
1. Copy to `/modules/addons/invoice_reminder_pro/`
2. Activate via WHMCS Admin

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('DailyCronJob', 1, function($vars) {
    InvoiceReminderPro::processReminders();
});

add_hook('InvoiceCreated', 1, function($vars) {
    InvoiceReminderPro::scheduleReminders($vars['invoiceid']);
});
```

### includes/InvoiceReminderPro.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class InvoiceReminderPro
{
    private static $table = 'mod_invoice_reminder_pro';
    
    public static function scheduleReminders($invoiceId)
    {
        $invoice = full_query("
            SELECT i.*, c.email, c.firstname, c.lastname
            FROM " . TABLE_PREFIX . "tblinvoices i
            JOIN " . TABLE_PREFIX . "tblclients c ON i.userid = c.id
            WHERE i.id = " . (int)$invoiceId
        );
        $data = mysql_fetch_array($invoice);
        
        if (!$data) return;
        
        $dueDate = strtotime($data['duedate']);
        $now = time();
        $daysUntilDue = ceil(($dueDate - $now) / 86400);
        
        // Schedule reminders based on days until due
        $reminders = self::getConfiguredReminders();
        
        foreach ($reminders as $reminder) {
            $sendDays = $reminder['days_before'] ?? 0;
            $sendTime = $dueDate - ($sendDays * 86400);
            
            if ($sendTime > $now) {
                self::addToQueue($invoiceId, $reminder['template'], $sendTime);
            }
        }
    }
    
    public static function processReminders()
    {
        $pending = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$table . "
            WHERE status = 'pending'
            AND scheduled_for <= NOW()
        ");
        
        while ($reminder = mysql_fetch_array($pending)) {
            self::sendReminder($reminder);
            
            full_query("
                UPDATE " . TABLE_PREFIX . self::$table . "
                SET status = 'sent', sent_at = NOW()
                WHERE id = " . (int)$reminder['id']
            );
        }
    }
    
    private static function sendReminder($reminder)
    {
        $invoice = full_query("
            SELECT i.*, c.email, c.firstname, c.lastname, c.companyname
            FROM " . TABLE_PREFIX . "tblinvoices i
            JOIN " . TABLE_PREFIX . "tblclients c ON i.userid = c.id
            WHERE i.id = " . (int)$reminder['invoice_id']
        );
        $data = mysql_fetch_array($invoice);
        
        if (!$data || !$data['email']) return;
        
        $vars = [
            'client_name' => $data['firstname'] . ' ' . $data['lastname'],
            'company_name' => $data['companyname'],
            'invoice_id' => $data['id'],
            'invoice_number' => $data['invoicenum'],
            'amount' => format_currency($data['total']),
            'due_date' => fromMySQLDate($data['duedate']),
            'balance' => format_currency($data['total'] - $data['amountpaid']),
            'invoice_url' => $data['systemurl'] . '/viewinvoice.php?id=' . $data['id']
        ];
        
        send_email($data['email'], $reminder['template'], $vars);
    }
    
    private static function addToQueue($invoiceId, $template, $scheduledFor)
    {
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$table . "
            (invoice_id, template, scheduled_for, status, created_at)
            VALUES (
                " . (int)$invoiceId . ",
                '" . db_escape_string($template) . "',
                FROM_UNIXTIME(" . (int)$scheduledFor . "),
                'pending',
                NOW()
            )
        ");
    }
    
    private static function getConfiguredReminders()
    {
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . "mod_invoice_reminder_pro_config
            WHERE enabled = 1
            ORDER BY days_before DESC
        ");
        
        $reminders = [];
        while ($row = mysql_fetch_array($result)) {
            $reminders[] = $row;
        }
        
        return $reminders;
    }
    
    public static function getStats($invoiceId)
    {
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$table . "
            WHERE invoice_id = " . (int)$invoiceId . "
            ORDER BY scheduled_for ASC
        ");
        
        $stats = [];
        while ($row = mysql_fetch_array($result)) {
            $stats[] = $row;
        }
        
        return $stats;
    }
    
    public static function cancelReminders($invoiceId)
    {
        full_query("
            UPDATE " . TABLE_PREFIX . self::$table . "
            SET status = 'cancelled'
            WHERE invoice_id = " . (int)$invoiceId . "
            AND status = 'pending'
        ");
    }
}

function invoice_reminder_pro_activate()
{
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_invoice_reminder_pro (
            id INT AUTO_INCREMENT PRIMARY KEY,
            invoice_id INT NOT NULL,
            template VARCHAR(100),
            scheduled_for DATETIME,
            status VARCHAR(20) DEFAULT 'pending',
            sent_at DATETIME,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_invoice_reminder_pro_config (
            id INT AUTO_INCREMENT PRIMARY KEY,
            days_before INT,
            template VARCHAR(100),
            enabled TINYINT(1) DEFAULT 1
        )
    ");
    
    // Insert default configuration
    $defaults = [
        ['days_before' => 7, 'template' => 'invoice_reminder_7day'],
        ['days_before' => 3, 'template' => 'invoice_reminder_3day'],
        ['days_before' => 1, 'template' => 'invoice_reminder_1day'],
        ['days_before' => 0, 'template' => 'invoice_overdue']
    ];
    
    foreach ($defaults as $config) {
        full_query("
            INSERT INTO " . TABLE_PREFIX . "mod_invoice_reminder_pro_config (days_before, template, enabled)
            VALUES (" . (int)$config['days_before'] . ", '" . db_escape_string($config['template']) . "', 1)
        ");
    }
    
    return ['status' => 'success', 'description' => 'Invoice Reminder Pro activated'];
}

function invoice_reminder_pro_deactivate()
{
    return ['status' => 'success', 'description' => 'Invoice Reminder Pro deactivated'];
}

function invoice_reminder_pro_config()
{
    return [
        'name' => 'Invoice Reminder Pro',
        'description' => 'Advanced invoice reminder system',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'auto_schedule' => [
                'Type' => 'yesno',
                'FriendlyName' => 'Auto Schedule Reminders',
                'Default' => '1'
            ],
            'max_reminders' => [
                'Type' => 'text',
                'FriendlyName' => 'Max Reminders per Invoice',
                'Default' => '4'
            ]
        ]
    ];
}
```