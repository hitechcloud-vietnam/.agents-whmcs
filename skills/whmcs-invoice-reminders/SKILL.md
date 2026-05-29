# WHMCS Invoice Reminders

## Concept Explanation
Automated invoice reminders improve payment collection rates through timely notifications.

### Reminder Schedule
- **Due Soon**: 3 days before due date
- **Overdue 1**: 1 day after due date
- **Overdue 2**: 5 days after due date
- **Final Notice**: 15 days after due date

## Code Patterns

```php
<?php
add_hook('DailyCronJob', 1, function() {
    $invoices = full_query("SELECT * FROM tblinvoices WHERE status = 'Unpaid'");
    
    while ($invoice = mysql_fetch_array($invoices)) {
        $daysUntilDue = daysUntil($invoice['duedate']);
        $reminder = determineReminder($daysUntilDue);
        
        if ($reminder) sendInvoiceReminder($invoice, $reminder);
    }
});

function determineReminder($daysUntilDue) {
    $schedule = [
        -3 => 'invoice_reminder_due_soon',
        0 => 'invoice_reminder_today',
        1 => 'invoice_overdue_1',
        5 => 'invoice_overdue_2',
        10 => 'invoice_overdue_3',
        15 => 'invoice_final_notice'
    ];
    
    return isset($schedule[$daysUntilDue]) ? $schedule[$daysUntilDue] : null;
}
```
