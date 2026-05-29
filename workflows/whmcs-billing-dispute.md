# WHMCS Billing Dispute Workflow

## Overview
This workflow automates billing dispute handling.

## Prerequisites
- WHMCS installation
- Dispute resolution process

## Step-by-Step Guide

### Step 1: Create Dispute Handler
```php
add_hook('TicketOpen', 1, function($vars) {
    $ticketId = $vars['ticketid'];
    $subject = $vars['subject'];
    
    // Check if billing-related
    if (is_billing_dispute($subject)) {
        $ticket = \WHMCS\Support\Ticket::find($ticketId);
        
        // Create dispute record
        $dispute = \WHMCS\Database\Capsule::table('mod_yourmodule_disputes')->insert([
            'ticket_id' => $ticketId,
            'client_id' => $ticket->clientId,
            'invoice_id' => extract_invoice_id($subject),
            'status' => 'investigating',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        // Auto-respond
        send_dispute_acknowledgment($ticketId);
        
        // Assign to billing department
        assign_ticket($ticketId, 'billing_team');
    }
});
```

### Step 2: Process Dispute
```php
function resolve_dispute(int $disputeId, string $resolution, float $adjustment = 0): bool
{
    $dispute = \WHMCS\Database\Capsule::table('mod_yourmodule_disputes')
        ->where('id', $disputeId)
        ->first();
    
    if ($adjustment > 0) {
        // Create credit
        add_credit($dispute->client_id, $adjustment, "Dispute resolution: $resolution");
    }
    
    // Update dispute
    \WHMCS\Database\Capsule::table('mod_yourmodule_disputes')
        ->where('id', $disputeId)
        ->update([
            'status' => 'resolved',
            'resolution' => $resolution,
            'resolved_at' => date('Y-m-d H:i:s'),
        ]);
    
    // Close ticket
    close_ticket($dispute->ticket_id, $resolution);
    
    return true;
}
```

## Billing Dispute Checklist

### Intake
- [ ] Ticket identified as dispute
- [ ] Dispute record created
- [ ] Acknowledgment sent

### Investigation
- [ ] Invoice reviewed
- [ ] Client contacted
- [ ] Evidence gathered

### Resolution
- [ ] Decision made
- [ ] Client notified
- [ ] Credit/adjustment applied
