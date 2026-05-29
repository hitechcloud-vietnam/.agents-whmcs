# WHMCS Cancellation Flow Workflow

## Overview
This workflow automates the service cancellation process with retention options.

## Prerequisites
- WHMCS installation
- Cancellation workflow configured

## Step-by-Step Guide

### Step 1: Create Cancellation Hook
```php
add_hook('ServiceCancellationRequest', 1, function($vars) {
    $serviceId = $vars['serviceId'];
    $reason = $vars['reason'];
    
    // Store cancellation request
    \WHMCS\Database\Capsule::table('mod_yourmodule_cancellations')->insert([
        'service_id' => $serviceId,
        'reason' => $reason,
        'requested_at' => date('Y-m-d H:i:s'),
        'status' => 'pending',
    ]);
    
    // Send confirmation email
    $client = get_client_from_service($serviceId);
    send_cancellation_confirmation($client, $serviceId);
});

add_hook('AfterCronJob', 1, function() {
    // Process pending cancellations
    $pending = \WHMCS\Database\Capsule::table('mod_yourmodule_cancellations')
        ->where('status', 'pending')
        ->where('requested_at', '<', date('Y-m-d H:i:s', strtotime('-24 hours')))
        ->get();
    
    foreach ($pending as $cancellation) {
        process_service_cancellation($cancellation->service_id);
        
        \WHMCS\Database\Capsule::table('mod_yourmodule_cancellations')
            ->where('id', $cancellation->id)
            ->update(['status' => 'processed']);
    }
});

function process_service_cancellation(int $serviceId): void
{
    // Terminate in external system
    $api = new YourModuleAPI();
    $api->terminate($serviceId);
    
    // Update WHMCS
    \WHMCS\Database\Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->update(['domainstatus' => 'Terminated']);
    
    // Send final invoice
    create_termination_invoice($serviceId);
}
```

## Cancellation Flow Checklist

### Request
- [ ] Reason captured
- [ ] Confirmation sent
- [ ] Retention offered

### Processing
- [ ] External service terminated
- [ ] WHMCS updated
- [ ] Final invoice sent

### Completion
- [ ] Data exported
- [ ] Account archived
- [ ] Satisfaction survey sent
