# WHMCS Cross-Sell Campaign Workflow

## Overview
Comprehensive workflow for automating cross-sell recommendations based on purchase history and customer behavior.

## Prerequisites
- WHMCS v8.0+
- Purchase history data

## Step-by-Step Guide

### Step 1: Cross-Sell Rules Engine
```php
<?php
class CrossSellEngine
{
    private array $rules = [];

    public function addRule(array $rule): void
    {
        $this->rules[] = $rule;
    }

    public function getRecommendations(int $clientId, int $limit = 5): array
    {
        $purchasedProducts = $this->getPurchasedProducts($clientId);
        $recommendations = [];

        foreach ($this->rules as $rule) {
            if ($this->matchesRule($rule, $purchasedProducts)) {
                $products = $this->getProductsByCategory($rule['target_category']);
                
                foreach ($products as $product) {
                    if (!in_array($product->id, $purchasedProducts)) {
                        $recommendations[] = [
                            'product_id' => $product->id,
                            'product_name' => $product->name,
                            'price' => $product->pricing()->first()->monthly ?? 0,
                            'reason' => $rule['reason'],
                            'match_score' => $rule['score'],
                        ];
                    }
                }
            }
        }

        // Sort by match score
        usort($recommendations, fn($a, $b) => $b['match_score'] - $a['match_score']);

        return array_slice($recommendations, 0, $limit);
    }

    private function matchesRule(array $rule, array $purchasedProducts): bool
    {
        $ownedCategories = \WHMCS\Database\Capsule::table('tblhosting')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->where('tblhosting.userid', \Auth::id())
            ->whereIn('tblproducts.gid', $rule['source_categories'])
            ->distinct()
            ->pluck('tblproducts.gid')
            ->toArray();

        return !empty(array_intersect($ownedCategories, $rule['source_categories']));
    }
}
```

### Step 2: Email Recommendations
```php
<?php
class CrossSellEmail
{
    public function sendRecommendations(int $clientId): void
    {
        $client = \WHMCS\User\Client::find($clientId);
        $engine = new CrossSellEngine();
        
        // Define rules
        $engine->addRule([
            'source_categories' => [1], // Web Hosting
            'target_category' => 3, // SSL Certificates
            'reason' => 'Complete your hosting security',
            'score' => 90,
        ]);

        $recommendations = $engine->getRecommendations($clientId, 3);

        if (!empty($recommendations)) {
            send_email($client->email, 'Cross-Sell Offer', [
                'client_name' => $client->fullName,
                'recommendations' => $recommendations,
                'personal_note' => 'Based on your recent purchases...',
            ]);
        }
    }
}
```

### Step 3: Cart Integration
```php
<?php
add_hook('ShoppingCartCompletePage', 1, function($vars) {
    $clientId = \Auth::id();
    $engine = new CrossSellEngine();
    
    $recommendations = $engine->getRecommendations($clientId, 3);
    
    return [
        'crosssell_products' => $recommendations,
    ];
});
```

## Automation Steps
1. Analyze customer purchase history
2. Apply cross-sell rules
3. Generate personalized recommendations
4. Display on checkout page
5. Send follow-up emails
6. Track conversion rates
