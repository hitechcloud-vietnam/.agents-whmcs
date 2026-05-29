# WHMCS Client Notes Automation Workflow

## Description
Automatically add notes to client accounts based on events.

## Steps

### Step 1: Create Note Automation Rules
```php
<?php
add_hook('OrderPaid', 1, function($vars) {
    $clientId = $vars['userId'];
    $orderId = $vars['orderId'];
    
    addClientNote($clientId, "Order #$orderId completed", 'system');
});

add_hook('InvoicePaid', 1, function($vars) {
    $clientId = Capsule::table('tblinvoices')
        ->where('id', $vars['invoiceid'])
        ->value('userid');
    
    addClientNote($clientId, "Invoice #{$vars['invoiceid']} paid", 'system');
});

add_hook('TicketOpen', 1, function($vars) {
    addClientNote($vars['userid'], "Support ticket #{$vars['ticketid']} opened", 'system');
});
```

### Step 2: Note Helper Function
```php
<?php
function addClientNote($clientId, $note, $type = 'admin')
{
    Capsule::table('tblnotes')->insert([
        'userid' => $clientId,
        'note' => $note,
        'type' => $type,
        'created_by' => $type === 'admin' ? $_SESSION['adminid'] : 0,
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}
```

## Tags
- notes
- automation
- tracking
- activity-log