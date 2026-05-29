# WHMCS VIP Tier Upgrade Workflow

## Overview
Automated VIP tier system based on customer lifetime value and engagement.

## Prerequisites
- WHMCS v8.0+
- Customer analytics

## Step-by-Step Guide

### Step 1: Tier Configuration
```php
<?php
namespace Vendor\Module;

class TierService
{
    protected array $tiers = [
        'standard' => ['min_value' => 0, 'discount' => 0, 'benefits' => ['basic_support']],
        'silver' => ['min_value' => 500, 'discount' => 5, 'benefits' => ['priority_support', 'early_access']],
        'gold' => ['min_value' => 2000, 'discount' => 10, 'benefits' => ['dedicated_support', 'early_access', 'free_setup']],
        'platinum' => ['min_value' => 5000, 'discount' => 15, 'benefits' => ['dedicated_manager', '24_7_support', 'custom_deals']],
    ];

    public function calculateLifetimeValue(int $clientId): float
    {
        $totalPaid = \WHMCS\Database\Capsule::table('tblaccounts')
            ->join('tblinvoices', 'tblaccounts.invoiceid', '=', 'tblinvoices.id')
            ->where('tblinvoices.userid', $clientId)
            ->where('tblinvoices.status', 'Paid')
            ->sum('tblaccounts.amount');

        return (float) $totalPaid;
    }

    public function updateClientTier(int $clientId): string
    {
        $lifetimeValue = $this->calculateLifetimeValue($clientId);
        $newTier = 'standard';

        foreach ($this->tiers as $tier => $config) {
            if ($lifetimeValue >= $config['min_value']) {
                $newTier = $tier;
            }
        }

        $currentTier = \WHMCS\Database\Capsule::table('mod_yourmodule_tiers')
            ->where('client_id', $clientId)->value('tier');

        \WHMCS\Database\Capsule::table('mod_yourmodule_tiers')
            ->where('client_id', $clientId)
            ->update([
                'tier' => $newTier,
                'lifetime_value' => $lifetimeValue,
                'updated_at' => date('Y-m-d H:i:s'),
            ]);

        if ($newTier !== $currentTier) {
            $this->notifyTierChange($clientId, $currentTier, $newTier);
        }

        return $newTier;
    }
}
```

### Step 2: Apply Tier Discount
```php
add_hook('ShoppingCartValidateCheckout', 1, function($vars) {
    $clientId = \Auth::id();
    $tier = \WHMCS\Database\Capsule::table('mod_yourmodule_tiers')
        ->where('client_id', $clientId)->first();

    if ($tier && $tier->tier !== 'standard') {
        $discount = \Vendor\Module\TierService::getTierDiscount($tier->tier);
        // Apply discount to cart
    }
});
```

## Checklist
- Tier system configured
- Lifetime value calculated
- Tier upgrades automated
- Discounts applied automatically
- Customers notified of upgrades
