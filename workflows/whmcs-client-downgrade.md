# WHMCS Client Downgrade Workflow

## Description
Manage client account downgrade procedures in WHMCS.

## Steps

### Step 1: Configure Downgrade Policy
```php
<?php
$downgradePolicy = [
    'allow_immediate' => false,
    'effective_date' => 'end_of_billing_cycle',
    'refund_type' => 'credit',
    'notice_required_days' => 30,
];
```

### Step 2: Downgrade Request Handler
```php
<?php
add_hook('ClientAreaPageProductDetails', 1, function($vars) {
    if (isset($_POST['downgrade_package'])) {
        return processDowngradeRequest($vars['service']['id'], $_POST['new_package']);
    }
});

function processDowngradeRequest($serviceId, $newPackageId)
{
    $service = Capsule::table('tblhosting')->find($serviceId);
    
    // Validate downgrade
    if (!$this->isValidDowngrade($service->packageid, $newPackageId)) {
        return ['error' => 'Downgrade not allowed'];
    }
    
    // Schedule for end of cycle
    Capsule::table('mod_downgrade_scheduled')->insert([
        'service_id' => $serviceId,
        'new_package_id' => $newPackageId,
        'effective_date' => $service->nextduedate,
        'status' => 'scheduled',
    ]);
    
    return [
        'success' => true,
        'message' => 'Downgrade scheduled for ' . $service->nextduedate,
    ];
}
```

### Step 3: Cron for Scheduled Downgrades
```php
<?php
add_hook('DailyCronJob', 1, function() {
    // Process scheduled downgrades
    $scheduled = Capsule::table('mod_downgrade_scheduled')
        ->where('effective_date', '<=', date('Y-m-d'))
        ->where('status', 'scheduled')
        ->get();
    
    foreach ($scheduled as $downgrade) {
        executeDowngrade($downgrade);
    }
});

function executeDowngrade($downgrade)
{
    Capsule::table('tblhosting')
        ->where('id', $downgrade->service_id)
        ->update(['packageid' => $downgrade->new_package_id]);
    
    Capsule::table('mod_downgrade_scheduled')
        ->where('id', $downgrade->id)
        ->update(['status' => 'completed']);
    
    sendEmail('ServiceDowngradeComplete', $downgrade->service_id);
}
```

## Tags
- downgrade
- account-change
- billing
- cancellation