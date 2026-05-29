# WHMCS Ticket Workflow

## Concept
Automated ticket routing, assignment, and resolution workflows.

## Code
```php
<?php
class TicketWorkflow {
    public static function processNewTicket($ticketId) {
        $ticket = getTicket($ticketId);
        
        // Auto-assign based on department
        $deptRules = self::getDepartmentRules($ticket["deptid"]);
        if ($deptRules["auto_assign"]) {
            $assignedTo = self::getNextAgent($deptRules["team_id"]);
            updateTicket($ticketId, ["assignedto" => $assignedTo]);
        }
        
        // Apply SLA if configured
        if ($deptRules["sla_id"]) {
            self::applySLA($ticketId, $deptRules["sla_id"]);
        }
    }
    
    public static function getNextAgent($teamId) {
        $result = full_query("SELECT userid FROM tbladmins WHERE roleid IN (SELECT roleid FROM tbladminroles WHERE module = "support") ORDER BY ticket_count ASC LIMIT 1");
        return mysql_fetch_array($result)["userid"] ?? 0;
    }
}

add_hook("TicketOpen", 1, function($vars) { TicketWorkflow::processNewTicket($vars["ticketid"]); });
```
