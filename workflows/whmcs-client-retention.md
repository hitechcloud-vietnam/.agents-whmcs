# WHMCS Client Retention Workflow

## Description
Strategies and automation for customer retention.

## Steps

### Step 1: Identify At-Risk Clients
```php
<?php
function getAtRiskClients()
{
    return Capsule::table('tblclients')
        ->where('status', 'Active')
        ->whereExists(function($q) {
            $q->from('tblinvoices')
              ->whereRaw('userid = tblclients.id')
              ->whereIn('status', ['Unpaid', 'Overdue']);
        })
        ->get();
}
```

### Step 2: Retention Actions
```php
<?php
add_hook('DailyCronJob', 1, function() {
    // Check for at-risk clients
    $atRisk = getAtRiskClients();
    
    foreach ($atRisk as $client) {
        // Send proactive support message
        sendEmail('ProactiveSupport', $client->id);
        
        // Add to retention tracking
        Capsule::table('mod_retention_tracking')->insert([
            'client_id' => $client->id,
            'status' => 'at_risk',
            'detected_at' => date('Y-m-d H:i:s'),
        ]);
    }
});
```

### Step 3: Loyalty Rewards
```php
<?php
function applyLoyaltyDiscount($clientId)
{
    $years = getClientTenure($clientId);
    
    if ($years >= 3) {
        $discount = 10; // 10% for 3+ years
    } elseif ($years >= 2) {
        $discount = 5; // 5% for 2+ years
    } else {
        $discount = 0;
    }
    
    if ($discount > 0) {
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update(['loyalty_discount' => $discount]);
        
        logActivity("Applied $discount% loyalty discount to client $clientId");
    }
}
```

## Tags
- retention
- loyalty
- customer-success
- engagement