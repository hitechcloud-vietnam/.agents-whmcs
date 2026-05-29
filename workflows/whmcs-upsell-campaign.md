# WHMCS Upsell Campaign Workflow

## Overview
Automated upsell workflow for suggesting premium upgrades to existing customers.

## Prerequisites
- WHMCS v8.0+
- Product configuration

## Step-by-Step Guide

### Step 1: Upsell Rules Engine
```php
<?php
namespace Vendor\Module;

class UpsellEngine
{
    protected array $rules = [
        ['source' => 'basic_plan', 'target' => 'pro_plan', 'trigger' => '30_days_active'],
        ['source' => 'single_domain', 'target' => 'unlimited_domains', 'trigger' => '3_domains'],
    ];

    public function findUpsellOpportunities(int $clientId): array
    {
        $services = $this->getClientServices($clientId);
        $opportunities = [];

        foreach ($services as $service) {
            foreach ($this->rules as $rule) {
                if ($this->matchesRule($service, $rule)) {
                    $opportunities[] = [
                        'service_id' => $service->id,
                        'current_product' => $service->product->name,
                        'recommended_product' => $this->getRecommendedProduct($rule['target']),
                        'savings' => $this->calculateSavings($rule),
                    ];
                }
            }
        }

        return $opportunities;
    }

    protected function calculateSavings(array $rule): float
    {
        $currentPrice = 9.99;
        $upgradePrice = 19.99;
        return round(($currentPrice - $upgradePrice) * 0.2, 2);
    }
}
```

### Step 2: Send Upsell Offers
```php
add_hook('AfterCronJob', 1, function() {
    $upsell = new \Vendor\Module\UpsellEngine();
    
    $activeClients = \WHMCS\User\Client::where('status', 'Active')->get();
    
    foreach ($activeClients as $client) {
        $opportunities = $upsell->findUpsellOpportunities($client->id);
        
        if (!empty($opportunities)) {
            send_email($client->email, 'Upgrade Your Service', [
                'client_name' => $client->fullName,
                'opportunities' => $opportunities,
            ]);
        }
    }
});
```

## Checklist
- Upsell rules defined
- Recommendations generated
- Offers sent automatically
- Conversion tracked
