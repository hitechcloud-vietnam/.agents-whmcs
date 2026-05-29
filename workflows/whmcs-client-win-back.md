# WHMCS Client Win-Back Workflow

## Description
Re-engage dormant or churned customers.

## Steps

### Step 1: Identify Win-Back Candidates
```php
<?php
function getWinBackCandidates($daysInactive = 90)
{
    return Capsule::table('tblclients')
        ->where('status', 'Active')
        ->where('lastlogin', '<', date('Y-m-d', strtotime("-$daysInactive days")))
        ->whereNotIn('id', function($q) {
            $q->from('tblorders')
              ->select('userid')
              ->where('status', 'Active')
              ->where('date', '>=', date('Y-m-d', strtotime("-{$daysInactive} days")));
        })
        ->get();
}
```

### Step 2: Create Win-Back Campaign
```php
<?php
function sendWinBackCampaign($clientId)
{
    // Check if already in win-back
    $existing = Capsule::table('mod_winback_campaigns')
        ->where('client_id', $clientId)
        ->whereIn('status', ['sent', 'responded'])
        ->first();
    
    if ($existing) {
        return false;
    }
    
    // Create campaign record
    Capsule::table('mod_winback_campaigns')->insert([
        'client_id' => $clientId,
        'template' => 'win_back_special',
        'sent_at' => date('Y-m-d H:i:s'),
        'status' => 'sent',
    ]);
    
    // Send offer
    $offer = generateWinBackOffer($clientId);
    
    sendEmail('WinBackCampaign', $clientId, [
        'offer_code' => $offer['code'],
        'discount' => $offer['discount'],
    ]);
    
    return true;
}

function generateWinBackOffer($clientId)
{
    $lifetimeValue = getClientLifetimeValue($clientId);
    
    if ($lifetimeValue > 1000) {
        return ['code' => 'WELCOME_BACK30', 'discount' => '30%'];
    } else {
        return ['code' => 'WELCOME_BACK15', 'discount' => '15%'];
    }
}
```

### Step 3: Track Win-Back Response
```php
<?php
add_hook('OrderPaid', 1, function($vars) {
    $campaign = Capsule::table('mod_winback_campaigns')
        ->where('client_id', $vars['userId'])
        ->where('status', 'sent')
        ->first();
    
    if ($campaign) {
        Capsule::table('mod_winback_campaigns')
            ->where('id', $campaign->id)
            ->update([
                'status' => 'responded',
                'responded_at' => date('Y-m-d H:i:s'),
                'order_id' => $vars['orderId'],
            ]);
        
        logActivity("Win-back campaign successful for client {$vars['userId']}");
    }
});
```

## Tags
- win-back
- re-engagement
- churn-prevention
- campaign