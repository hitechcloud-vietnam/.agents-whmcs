# WHMCS Client Reactivation Workflow

## Description
Reactivate suspended client accounts in WHMCS.

## Steps

### Step 1: Reactivate Account
```php
<?php
function reactivateClient($clientId, $reason = '')
{
    // Update client status
    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update(['status' => 'Active']);
    
    // Log reactivation
    Capsule::table('mod_reactivation_log')->insert([
        'client_id' => $clientId,
        'reason' => $reason,
        'reactivated_by' => $_SESSION['adminid'],
        'reactivated_at' => date('Y-m-d H:i:s'),
    ]);
    
    // Send notification
    sendEmail('AccountReactivated', $clientId);
    
    logActivity("Client reactivated: ID $clientId");
}
```

### Step 2: Reactivate Services
```php
<?php
add_hook('InvoicePaid', 1, function($vars) {
    $invoice = Capsule::table('tblinvoices')->find($vars['invoiceid']);
    
    // Check if any suspended services should be reactivated
    $services = Capsule::table('tblhosting')
        ->where('userid', $invoice->userid)
        ->where('domainstatus', 'Suspended')
        ->get();
    
    foreach ($services as $service) {
        Capsule::table('tblhosting')
            ->where('id', $service->id)
            ->update(['domainstatus' => 'Active']);
        
        sendEmail('ServiceReactivated', $service->userid, ['service_id' => $service->id]);
    }
});
```

## Tags
- reactivation
- restoration
- account-recovery
- unsuspend