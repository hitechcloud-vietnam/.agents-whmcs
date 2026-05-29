# WHMCS Dunning Automation

## Concept Explanation

Dunning automation manages the collection process for overdue invoices through systematic reminder sequences, escalation policies, and recovery actions. Effective dunning reduces bad debt, improves cash flow, and maintains customer relationships through professional communication.

### Dunning Stages

1. **Friendly Reminder**: Initial overdue notification
2. **Urgent Notice**: Second reminder with late fee warning
3. **Final Notice**: Critical reminder before service impact
4. **Suspension Warning**: Service suspension imminent
5. **Suspension Action**: Services suspended
6. **Collection Action**: External collection consideration

## Code Patterns

### Dunning Configuration

```php
<?php
// includes/DunningAutomation.php

class DunningAutomation {
    
    const STAGE_FRIENDLY = 1;
    const STAGE_URGENT = 2;
    const STAGE_FINAL = 3;
    const STAGE_SUSPENSION_WARNING = 4;
    const STAGE_SUSPENDED = 5;
    
    public static function processDunningCycle() {
        $overdueInvoices = self::getOverdueInvoices();
        
        foreach ($overdueInvoices as $invoice) {
            $currentStage = self::getCurrentDunningStage($invoice['id']);
            $daysOverdue = $invoice['days_overdue'];
            
            $nextAction = self::determineNextAction($daysOverdue, $currentStage);
            
            if ($nextAction['execute']) {
                self::executeDunningAction($invoice, $nextAction);
            }
        }
    }
    
    public static function getOverdueInvoices() {
        $query = "SELECT i.*, 
            DATEDIFF(NOW(), i.duedate) as days_overdue,
            c.firstname, c.lastname, c.email
            FROM tblinvoices i
            JOIN tblclients c ON i.userid = c.id
            WHERE i.status = 'Unpaid'
            AND i.duedate < CURDATE()
            AND i.billingcycle != 'Free'
            AND i.billingcycle != 'One Time'";
        return full_query($query);
    }
    
    public static function getCurrentDunningStage($invoiceId) {
        $result = full_query("SELECT MAX(stage) as stage FROM tbl_dunning_log WHERE invoice_id = " . (int)$invoiceId);
        $row = mysql_fetch_array($result);
        return (int)($row['stage'] ?? 0);
    }
    
    public static function determineNextAction($daysOverdue, $currentStage) {
        $schedule = [
            ['days' => 1, 'stage' => 1, 'action' => 'friendly_reminder', 'email' => 'invoice_overdue_reminder_1'],
            ['days' => 5, 'stage' => 2, 'action' => 'urgent_notice', 'email' => 'invoice_overdue_reminder_2', 'late_fee' => 5],
            ['days' => 10, 'stage' => 3, 'action' => 'final_notice', 'email' => 'invoice_overdue_reminder_3'],
            ['days' => 15, 'stage' => 4, 'action' => 'suspension_warning', 'email' => 'service_suspension_warning'],
            ['days' => 20, 'stage' => 5, 'action' => 'suspend_service', 'email' => 'service_suspended']
        ];
        
        foreach ($schedule as $item) {
            if ($daysOverdue >= $item['days'] && $currentStage < $item['stage']) {
                return ['execute' => true, 'action' => $item['action'], 'email' => $item['email'], 'stage' => $item['stage']];
            }
        }
        
        return ['execute' => false];
    }
    
    public static function executeDunningAction($invoice, $action) {
        insert_query('tbl_dunning_log', [
            'invoice_id' => $invoice['id'],
            'stage' => $action['stage'],
            'action' => $action['action'],
            'executed_at' => date('Y-m-d H:i:s')
        ]);
        
        send_email($action['email'], $invoice['userid'], ['invoice_id' => $invoice['id']]);
        
        if ($action['action'] === 'suspend_service') {
            self::suspendOverdueServices($invoice['userid']);
        }
        
        return true;
    }
    
    public static function suspendOverdueServices($clientId) {
        $query = "SELECT id FROM tblhosting WHERE userid = ? AND domain != '' AND regdate != '0000-00-00'";
        $services = full_query($query, [$clientId]);
        
        while ($service = mysql_fetch_array($services)) {
            localAPI('ModuleSuspend', ['serviceid' => $service['id']]);
        }
    }
}

// Cron hook for daily dunning processing
add_hook('DailyCronJob', 1, function() {
    \DunningAutomation::processDunningCycle();
});
```

## Implementation Checklist

- [ ] Create dunning log table
- [ ] Configure dunning schedule
- [ ] Create email templates for each stage
- [ ] Implement service suspension logic
- [ ] Test dunning workflow
