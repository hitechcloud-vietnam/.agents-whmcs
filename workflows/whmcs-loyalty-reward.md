# WHMCS Loyalty Reward Workflow

## Overview
Loyalty program with points accumulation, tier management, and reward redemption.

## Prerequisites
- WHMCS v8.0+
- Points tracking system

## Step-by-Step Guide

### Step 1: Loyalty Database
```sql
CREATE TABLE `mod_yourmodule_loyalty_points` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `client_id` INT NOT NULL,
    `points` INT DEFAULT 0,
    `lifetime_points` INT DEFAULT 0,
    `tier` VARCHAR(50) DEFAULT 'bronze',
    `updated_at` DATETIME,
    FOREIGN KEY (`client_id`) REFERENCES `tblclients`(`id`)
);

CREATE TABLE `mod_yourmodule_points_history` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `client_id` INT NOT NULL,
    `points` INT NOT NULL,
    `type` VARCHAR(50),
    `reference` VARCHAR(100),
    `created_at` DATETIME
);
```

### Step 2: Points Service
```php
<?php
namespace Vendor\Module;

class LoyaltyService
{
    protected array $tiers = [
        'bronze' => ['min_points' => 0, 'multiplier' => 1.0],
        'silver' => ['min_points' => 1000, 'multiplier' => 1.25],
        'gold' => ['min_points' => 5000, 'multiplier' => 1.5],
        'platinum' => ['min_points' => 10000, 'multiplier' => 2.0],
    ];

    public function addPoints(int $clientId, int $points, string $type, string $reference): void
    {
        $currentTier = $this->getTier($clientId);
        $multiplier = $this->tiers[$currentTier]['multiplier'];
        $actualPoints = (int)($points * $multiplier);

        \WHMCS\Database\Capsule::table('mod_yourmodule_loyalty_points')
            ->where('client_id', $clientId)
            ->update([
                'points' => \WHMCS\Database\Capsule::raw("points + $actualPoints"),
                'lifetime_points' => \WHMCS\Database\Capsule::raw("lifetime_points + $actualPoints"),
                'tier' => $this->calculateTier($clientId),
            ]);

        $this->logPoints($clientId, $actualPoints, $type, $reference);
    }

    protected function calculateTier(int $clientId): string
    {
        $lifetime = \WHMCS\Database\Capsule::table('mod_yourmodule_loyalty_points')
            ->where('client_id', $clientId)->value('lifetime_points');

        foreach ($this->tiers as $tier => $config) {
            if ($lifetime >= $config['min_points']) {
                return $tier;
            }
        }
        return 'bronze';
    }
}
```

### Step 3: Award Points on Purchase
```php
add_hook('OrderPaid', 1, function($vars) {
    $orderId = $vars['orderId'];
    $clientId = $vars['userId'];
    
    $order = \WHMCS\Billing\Order\Order::find($orderId);
    $points = (int)($order->total * 10); // 10 points per dollar

    $loyalty = new \Vendor\Module\LoyaltyService();
    $loyalty->addPoints($clientId, $points, 'order', "Order #$orderId");
});
```

## Checklist
- Points tracking enabled
- Tier system configured
- Points awarded on purchases
- Redemption workflow functional
