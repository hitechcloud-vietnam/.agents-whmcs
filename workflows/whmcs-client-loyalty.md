# WHMCS Client Loyalty Program Workflow

## Description
Implement a customer loyalty and rewards program.

## Steps

### Step 1: Loyalty Points System
```php
<?php
/**
 * Loyalty Points Configuration
 */

$loyaltyConfig = [
    'points_per_dollar' => 1, // 1 point per $1 spent
    'redemption_rate' => 0.01, // $0.01 per point
    'expiry_months' => 12, // Points expire after 12 months
    'tier_thresholds' => [
        'bronze' => 0,
        'silver' => 1000,
        'gold' => 5000,
        'platinum' => 15000,
    ],
    'tier_bonuses' => [
        'bronze' => 0,
        'silver' => 0.1, // 10% bonus points
        'gold' => 0.25,
        'platinum' => 0.5,
    ],
];
```

### Step 2: Award Points
```php
<?php
add_hook('InvoicePaid', 1, function($vars) {
    $invoice = Capsule::table('tblinvoices')->find($vars['invoiceid']);
    
    $points = floor($invoice->total * $loyaltyConfig['points_per_dollar']);
    
    // Apply tier bonus
    $tier = getClientTier($invoice->userid);
    $bonusMultiplier = $loyaltyConfig['tier_bonuses'][$tier] ?? 0;
    $totalPoints = floor($points * (1 + $bonusMultiplier));
    
    Capsule::table('mod_loyalty_points')->insert([
        'client_id' => $invoice->userid,
        'invoice_id' => $invoice->id,
        'points' => $totalPoints,
        'type' => 'earned',
        'expires_at' => date('Y-m-d H:i:s', strtotime('+' . $loyaltyConfig['expiry_months'] . ' months')),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    // Update tier
    updateClientLoyaltyTier($invoice->userid);
});
```

### Step 3: Points Redemption
```php
<?php
function redeemLoyaltyPoints($clientId, $points)
{
    $balance = getPointsBalance($clientId);
    
    if ($points > $balance) {
        return ['error' => 'Insufficient points'];
    }
    
    $credit = $points * $loyaltyConfig['redemption_rate'];
    
    Capsule::table('mod_loyalty_points')->insert([
        'client_id' => $clientId,
        'points' => -$points,
        'type' => 'redeemed',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    // Apply credit to account
    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update(['credit' => Capsule::raw('credit + ' . $credit)]);
    
    return [
        'success' => true,
        'points_redeemed' => $points,
        'credit_earned' => $credit,
    ];
}
```

### Step 4: Tier Management
```php
<?php
function getClientTier($clientId)
{
    $totalPoints = Capsule::table('mod_loyalty_points')
        ->where('client_id', $clientId)
        ->where('expires_at', '>', date('Y-m-d H:i:s'))
        ->sum('points');
    
    $tiers = $loyaltyConfig['tier_thresholds'];
    arsort($tiers);
    
    foreach ($tiers as $tier => $threshold) {
        if ($totalPoints >= $threshold) {
            return $tier;
        }
    }
    
    return 'bronze';
}
```

## Tags
- loyalty
- rewards
- points
- tier-system
- retention