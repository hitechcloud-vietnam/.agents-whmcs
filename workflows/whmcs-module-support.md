# WHMCS Module Support Integration Workflow

## Description
Integrate support ticketing into WHMCS modules.

## Steps

### Step 1: Create Support Service
```php
<?php
/**
 * Module Support Service
 */

class CLICodesSupport
{
    private $deptId = 1; // Support department ID
    
    /**
     * Open support ticket
     */
    public function openTicket($clientId, $subject, $message, $priority = 'Medium')
    {
        $result = localAPI('OpenTicket', [
            'clientid' => $clientId,
            'deptid' => $this->deptId,
            'subject' => $subject,
            'message' => $message,
            'priority' => $priority,
        ]);
        
        return $result;
    }
    
    /**
     * Get ticket replies
     */
    public function getTicketReplies($ticketId)
    {
        $result = localAPI('GetTicket', [
            'ticketid' => $ticketId,
        ]);
        
        return $result['replies']['reply'] ?? [];
    }
    
    /**
     * Add reply to ticket
     */
    public function addReply($ticketId, $message)
    {
        $result = localAPI('AddTicketReply', [
            'ticketid' => $ticketId,
            'message' => $message,
        ]);
        
        return $result;
    }
    
    /**
     * Log support request in module
     */
    public function logSupportRequest($clientId, $type, $details)
    {
        Capsule::table('mod_clicodes_support')->insert([
            'client_id' => $clientId,
            'type' => $type,
            'details' => json_encode($details),
            'status' => 'open',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

### Step 2: Add Support Features
```php
<?php
// In module output

function clicodes_example_support($vars)
{
    $support = new CLICodesSupport();
    $clientId = $vars['clientsdetails']['id'];
    
    if ($_POST['action'] === 'contact_support') {
        $support->openTicket(
            $clientId,
            $_POST['subject'],
            $_POST['message'],
            $_POST['priority'] ?? 'Medium'
        );
        
        return ['success' => true];
    }
}
```

## Support Features
- Knowledge base
- FAQ section
- Direct ticket creation
- Support status tracking

## Tags
- support
- ticketing
- customer-service