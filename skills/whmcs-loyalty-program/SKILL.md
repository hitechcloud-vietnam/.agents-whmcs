# WHMCS Loyalty Program Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Implement customer loyalty program with points, rewards, and tiered membership.

## Database Schema

```php
<?php
// modules/addons/loyalty_program/loyalty_program.php

use WHMCS\Database\Capsule;

function loyalty_program_config(): array {
    return [
        'name' => 'Loyalty Program',
        'description' => 'Customer rewards and loyalty system',
        'version' => '1.0',
    ];
}

function loyalty_program_activate(): array {
    Capsule::schema()->create('mod_loyalty_tiers', function($t) {
        $t->increments('id');
        $t->string('name', 50);
        $t->string('color', 7);
        $t->integer('min_points')->unsigned()->default(0);
        $t->decimal('point_multiplier', 5, 2)->default(1.00);
        $t->decimal('discount_percentage', 5, 2)->default(0);
        $t->text('benefits')->nullable();
        $t->integer('sort_order')->default(0);
    });

    Capsule::schema()->create('mod_loyalty_points', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->integer('tier_id')->unsigned()->nullable();
        $t->integer('balance')->default(0);
        $t->integer('lifetime_points')->default(0);
        $t->timestamp('tier_updated_at')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_loyalty_transactions', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->string('type', 30);
        $t->integer('points');
        $t->string('description', 255);
        $t->integer('order_id')->unsigned()->nullable();
        $t->string('status', 20)->default('completed');
        $t->timestamp('created_at')->useCurrent();
    });

    Capsule::schema()->create('mod_loyalty_rewards', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->text('description')->nullable();
        $t->integer('points_required');
        $t->string('reward_type', 30);
        $t->decimal('reward_value', 10, 2)->default(0);
        $t->string('applies_to', 50)->default('all');
        $t->text('product_ids')->nullable();
        $t->integer('max_redemptions')->unsigned()->nullable();
        $t->integer('redemption_count')->unsigned()->default(0);
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_loyalty_redemptions', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->integer('reward_id')->unsigned();
        $t->integer('points_spent');
        $t->string('coupon_code', 50)->nullable();
        $t->timestamp('redeemed_at')->useCurrent();
    });

    // Insert default tiers
    $tiers = [
        ['name' => 'Bronze', 'color' => '#CD7F32', 'min_points' => 0, 'point_multiplier' => 1.0, 'discount_percentage' => 0],
        ['name' => 'Silver', 'color' => '#C0C0C0', 'min_points' => 1000, 'point_multiplier' => 1.25, 'discount_percentage' => 5],
        ['name' => 'Gold', 'color' => '#FFD700', 'min_points' => 5000, 'point_multiplier' => 1.5, 'discount_percentage' => 10],
        ['name' => 'Platinum', 'color' => '#E5E4E2', 'min_points' => 10000, 'point_multiplier' => 2.0, 'discount_percentage' => 15],
    ];

    foreach ($tiers as $index => $tier) {
        Capsule::table('mod_loyalty_tiers')->insert(array_merge($tier, ['sort_order' => $index]));
    }

    return ['status' => 'success'];
}

function loyalty_program_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_loyalty_redemptions');
    Capsule::schema()->dropIfExists('mod_loyalty_rewards');
    Capsule::schema()->dropIfExists('mod_loyalty_transactions');
    Capsule::schema()->dropIfExists('mod_loyalty_points');
    Capsule::schema()->dropIfExists('mod_loyalty_tiers');
    return ['status' => 'success'];
}
```

## Loyalty Manager

```php
<?php
class LoyaltyManager {
    private int $pointsPerDollar = 10;

    public function getCustomerPoints(int $userId): object {
        $points = Capsule::table('mod_loyalty_points')
            ->where('user_id', $userId)
            ->first();

        if (!$points) {
            $points = (object)[
                'balance' => 0,
                'lifetime_points' => 0,
                'tier_id' => 1,
            ];

            Capsule::table('mod_loyalty_points')->insert([
                'user_id' => $userId,
                'tier_id' => 1,
            ]);
        }

        return $points;
    }

    public function getCustomerTier(int $userId): ?object {
        $points = $this->getCustomerPoints($userId);

        return Capsule::table('mod_loyalty_tiers')
            ->where('id', $points->tier_id)
            ->first();
    }

    public function awardPoints(int $userId, int $points, string $type, ?int $orderId = null, string $description = ''): array {
        $customerPoints = $this->getCustomerPoints($userId);
        $tier = $this->getCustomerTier($userId);

        // Apply tier multiplier
        $multipliedPoints = floor($points * $tier->point_multiplier);

        // Update balance
        Capsule::table('mod_loyalty_points')
            ->where('user_id', $userId)
            ->update([
                'balance' => $customerPoints->balance + $multipliedPoints,
                'lifetime_points' => $customerPoints->lifetime_points + $multipliedPoints,
            ]);

        // Record transaction
        Capsule::table('mod_loyalty_transactions')->insert([
            'user_id' => $userId,
            'type' => $type,
            'points' => $multipliedPoints,
            'description' => $description ?: "Points earned from {$type}",
            'order_id' => $orderId,
        ]);

        // Check for tier upgrade
        $this->checkTierUpgrade($userId);

        return [
            'success' => true,
            'points_awarded' => $multipliedPoints,
            'new_balance' => $customerPoints->balance + $multipliedPoints,
        ];
    }

    public function redeemPoints(int $userId, int $rewardId): array {
        $reward = Capsule::table('mod_loyalty_rewards')->find($rewardId);

        if (!$reward || !$reward->is_active) {
            return ['success' => false, 'error' => 'Reward not available'];
        }

        if ($reward->max_redemptions && $reward->redemption_count >= $reward->max_redemptions) {
            return ['success' => false, 'error' => 'Reward fully redeemed'];
        }

        $customerPoints = $this->getCustomerPoints($userId);

        if ($customerPoints->balance < $reward->points_required) {
            return ['success' => false, 'error' => 'Insufficient points'];
        }

        // Deduct points
        Capsule::table('mod_loyalty_points')
            ->where('user_id', $userId)
            ->decrement('balance', $reward->points_required);

        // Record redemption
        Capsule::table('mod_loyalty_redemptions')->insert([
            'user_id' => $userId,
            'reward_id' => $rewardId,
            'points_spent' => $reward->points_required,
        ]);

        Capsule::table('mod_loyalty_rewards')
            ->where('id', $rewardId)
            ->increment('redemption_count');

        // Record transaction
        Capsule::table('mod_loyalty_transactions')->insert([
            'user_id' => $userId,
            'type' => 'redemption',
            'points' => -$reward->points_required,
            'description' => "Redeemed: {$reward->name}",
        ]);

        // Generate coupon if needed
        if ($reward->reward_type === 'coupon') {
            $coupon = $this->generateRewardCoupon($userId, $reward);
            return [
                'success' => true,
                'coupon_code' => $coupon,
                'reward_name' => $reward->name,
            ];
        }

        return ['success' => true, 'reward_name' => $reward->name];
    }

    private function generateRewardCoupon(int $userId, object $reward): string {
        $code = 'LOYALTY' . strtoupper(substr(md5($userId . $reward->id . time()), 0, 8));

        $appliesTo = $reward->applies_to === 'all' ? 'all' : 'product';
        $productIds = $reward->applies_to !== 'all' ? json_encode(explode(',', $reward->product_ids)) : null;

        Capsule::table('mod_promo_codes')->insert([
            'code' => $code,
            'type' => 'fixed',
            'value' => $reward->reward_value,
            'max_uses' => 1,
            'max_uses_per_user' => 1,
            'user_id' => $userId,
            'applies_to' => $appliesTo,
            'product_ids' => $productIds,
            'valid_from' => date('Y-m-d'),
            'valid_until' => date('Y-m-d', strtotime('+30 days')),
            'is_active' => 1,
        ]);

        return $code;
    }

    private function checkTierUpgrade(int $userId): void {
        $points = $this->getCustomerPoints($userId);

        $nextTier = Capsule::table('mod_loyalty_tiers')
            ->where('min_points', '>', $points->lifetime_points)
            ->where('id', '!=', $points->tier_id)
            ->orderBy('min_points')
            ->first();

        if ($nextTier && $points->lifetime_points >= $nextTier->min_points) {
            Capsule::table('mod_loyalty_points')
                ->where('user_id', $userId)
                ->update([
                    'tier_id' => $nextTier->id,
                    'tier_updated_at' => date('Y-m-d H:i:s'),
                ]);

            // Notify customer
            $user = Capsule::table('tblclients')->find($userId);
            sendTplEmail($user->email, 'loyalty_tier_upgrade', [
                'tier_name' => $nextTier->name,
                'benefits' => $nextTier->benefits,
            ]);
        }
    }

    public function getAvailableRewards(int $userId = null): array {
        $points = $userId ? $this->getCustomerPoints($userId)->balance : 0;

        $rewards = Capsule::table('mod_loyalty_rewards')
            ->where('is_active', 1)
            ->where(function($q) {
                $q->whereNull('max_redemptions')
                  ->orWhereRaw('redemption_count < max_redemptions');
            })
            ->get();

        return array_map(function($reward) use ($points) {
            $reward->affordable = $points >= $reward->points_required;
            return $reward;
        }, $rewards->toArray());
    }

    public function getTransactionHistory(int $userId, int $limit = 50): array {
        return Capsule::table('mod_loyalty_transactions')
            ->where('user_id', $userId)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get();
    }
}
```

## Order Integration

```php
<?php
add_hook('AfterOrderPaid', 1, function($vars) {
    $orderId = $vars['orderid'];
    $order = Capsule::table('tblorders')->find($orderId);

    $loyaltyManager = new LoyaltyManager();
    $manager->awardPoints(
        $order->userid,
        floor($order->amount * $loyaltyManager->pointsPerDollar),
        'purchase',
        $orderId,
        "Points earned from order #{$orderId}"
    );
});
```

## Client Area

```php
<?php
function loyalty_program_clientarea(array $vars): array {
    $userId = $_SESSION['uid'];
    $loyaltyManager = new LoyaltyManager();

    $points = $loyaltyManager->getCustomerPoints($userId);
    $tier = $loyaltyManager->getCustomerTier($userId);
    $rewards = $loyaltyManager->getAvailableRewards($userId);
    $history = $loyaltyManager->getTransactionHistory($userId, 20);

    // Calculate progress to next tier
    $nextTier = Capsule::table('mod_loyalty_tiers')
        ->where('min_points', '>', $tier->min_points)
        ->orderBy('min_points')
        ->first();

    $progress = 0;
    $pointsNeeded = 0;

    if ($nextTier) {
        $tierRange = $nextTier->min_points - $tier->min_points;
        $currentProgress = $points->lifetime_points - $tier->min_points;
        $progress = min(100, ($currentProgress / $tierRange) * 100);
        $pointsNeeded = $nextTier->min_points - $points->lifetime_points;
    }

    return [
        'pagetitle' => 'Loyalty Program',
        'templatefile' => 'loyalty',
        'vars' => [
            'points' => $points->balance,
            'lifetime_points' => $points->lifetime_points,
            'tier' => $tier,
            'next_tier' => $nextTier,
            'progress' => $progress,
            'points_needed' => $pointsNeeded,
            'rewards' => $rewards,
            'history' => $history,
        ],
    ];
}
```

## Admin Dashboard

```php
<?php
function loyalty_program_output(array $vars): void {
    $action = $_GET['action'] ?? 'dashboard';

    $manager = new LoyaltyManager();

    if ($action === 'dashboard') {
        $stats = [
            'total_members' => Capsule::table('mod_loyalty_points')->count(),
            'total_points_issued' => Capsule::table('mod_loyalty_transactions')
                ->where('points', '>', 0)
                ->sum('points'),
            'total_points_redeemed' => abs(Capsule::table('mod_loyalty_transactions')
                ->where('points', '<', 0)
                ->sum('points')),
        ];

        $tierStats = [];
        $tiers = Capsule::table('mod_loyalty_tiers')->get();
        foreach ($tiers as $tier) {
            $tierStats[] = [
                'tier' => $tier,
                'members' => Capsule::table('mod_loyalty_points')
                    ->where('tier_id', $tier->id)
                    ->count(),
            ];
        }

        echo '<div class="loyalty-dashboard">';
        echo '<h2>Loyalty Program Dashboard</h2>';
        echo '<div class="stats-grid">';
        echo '<div class="stat-box"><h3>' . number_format($stats['total_members']) . '</h3><p>Total Members</p></div>';
        echo '<div class="stat-box"><h3>' . number_format($stats['total_points_issued']) . '</h3><p>Points Issued</p></div>';
        echo '<div class="stat-box"><h3>' . number_format($stats['total_points_redeemed']) . '</h3><p>Points Redeemed</p></div>';
        echo '</div>';

        echo '<h3>Tier Distribution</h3>';
        echo '<table class="datatable"><thead><tr><th>Tier</th><th>Members</th><th>Benefits</th></tr></thead><tbody>';
        foreach ($tierStats as $stat) {
            echo '<tr>';
            echo "<td style='color:{$stat['tier']->color}'>{$stat['tier']->name}</td>";
            echo "<td>{$stat['members']}</td>";
            echo "<td>{$stat['tier']->discount_percentage}% discount, {$stat['tier']->point_multiplier}x points</td>";
            echo '</tr>';
        }
        echo '</tbody></table>';
        echo '</div>';
    }
}
```

---

**Related Skills:**
- whmcs-customer-lifecycle
- whmcs-promotional-codes
- whmcs-store-credit