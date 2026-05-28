# WHMCS Referral System Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Implement multi-level referral system with commissions and tracking.

## Database Schema

```php
<?php
// modules/addons/referral_system/referral_system.php

use WHMCS\Database\Capsule;

function referral_system_config(): array {
    return [
        'name' => 'Referral System',
        'description' => 'Multi-level referral tracking and commissions',
        'version' => '1.0',
    ];
}

function referral_system_activate(): array {
    Capsule::schema()->create('mod_referral_referrers', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned()->unique();
        $t->string('referral_code', 20)->unique();
        $t->integer('parent_referrer_id')->unsigned()->nullable();
        $t->integer('referral_count')->unsigned()->default(0);
        $t->decimal('total_commission', 12, 2)->default(0);
        $t->decimal('pending_commission', 12, 2)->default(0);
        $t->timestamp('created_at')->useCurrent();
    });

    Capsule::schema()->create('mod_referral_referrals', function($t) {
        $t->increments('id');
        $t->integer('referrer_id')->unsigned();
        $t->integer('user_id')->unsigned();
        $t->string('referral_code_used', 20);
        $t->string('status', 20)->default('pending');
        $t->decimal('commission_amount', 10, 2)->default(0);
        $t->integer('order_id')->unsigned()->nullable();
        $t->timestamp('converted_at')->nullable();
        $t->timestamp('created_at')->useCurrent();
    });

    Capsule::schema()->create('mod_referral_tiers', function($t) {
        $t->increments('id');
        $t->string('name', 50);
        $t->integer('level')->unsigned()->unique();
        $t->string('commission_type', 20)->default('percentage');
        $t->decimal('commission_value', 10, 2);
        $t->decimal('fixed_bonus', 10, 2)->default(0);
        $t->integer('min_referrals')->unsigned()->default(0);
        $t->boolean('is_active')->default(true);
    });

    Capsule::schema()->create('mod_referral_payments', function($t) {
        $t->increments('id');
        $t->integer('referrer_id')->unsigned();
        $t->decimal('amount', 10, 2);
        $t->string('payment_method', 50);
        $t->string('status', 20)->default('pending');
        $t->string('transaction_id', 100)->nullable();
        $t->timestamp('paid_at')->nullable();
        $t->timestamp('created_at')->useCurrent();
    });

    Capsule::schema()->create('mod_referral_leaderboard', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->string('period', 20);
        $t->integer('referral_count')->unsigned()->default(0);
        $t->decimal('total_commission', 12, 2)->default(0);
        $t->integer('rank')->unsigned();
        $t->timestamps();
    });

    // Insert default tiers
    $tiers = [
        ['name' => 'Starter', 'level' => 1, 'commission_value' => 5, 'min_referrals' => 0],
        ['name' => 'Bronze', 'level' => 2, 'commission_value' => 7.5, 'min_referrals' => 5],
        ['name' => 'Silver', 'level' => 3, 'commission_value' => 10, 'min_referrals' => 15],
        ['name' => 'Gold', 'level' => 4, 'commission_value' => 12.5, 'min_referrals' => 30],
        ['name' => 'Platinum', 'level' => 5, 'commission_value' => 15, 'min_referrals' => 50],
    ];

    foreach ($tiers as $tier) {
        Capsule::table('mod_referral_tiers')->insert($tier);
    }

    return ['status' => 'success'];
}

function referral_system_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_referral_leaderboard');
    Capsule::schema()->dropIfExists('mod_referral_payments');
    Capsule::schema()->dropIfExists('mod_referral_referrals');
    Capsule::schema()->dropIfExists('mod_referral_tiers');
    Capsule::schema()->dropIfExists('mod_referral_referrers');
    return ['status' => 'success'];
}
```

## Referral Manager

```php
<?php
class ReferralManager {
    public function createReferrer(int $userId, ?int $parentId = null): array {
        $exists = Capsule::table('mod_referral_referrers')
            ->where('user_id', $userId)
            ->first();

        if ($exists) {
            return ['success' => true, 'referrer_id' => $exists->id, 'referral_code' => $exists->referral_code];
        }

        $referralCode = $this->generateReferralCode();

        $referrerId = Capsule::table('mod_referral_referrers')->insertGetId([
            'user_id' => $userId,
            'referral_code' => $referralCode,
            'parent_referrer_id' => $parentId,
        ]);

        return [
            'success' => true,
            'referrer_id' => $referrerId,
            'referral_code' => $referralCode,
        ];
    }

    private function generateReferralCode(): string {
        do {
            $code = strtoupper(substr(md5(random_bytes(16)), 0, 8));

            $exists = Capsule::table('mod_referral_referrers')
                ->where('referral_code', $code)
                ->exists();
        } while ($exists);

        return $code;
    }

    public function processReferral(string $referralCode, int $newUserId): array {
        $referrer = Capsule::table('mod_referral_referrers')
            ->where('referral_code', $referralCode)
            ->first();

        if (!$referrer) {
            return ['success' => false, 'error' => 'Invalid referral code'];
        }

        if ($referrer->user_id === $newUserId) {
            return ['success' => false, 'error' => 'Cannot refer yourself'];
        }

        $existingReferral = Capsule::table('mod_referral_referrals')
            ->where('referrer_id', $referrer->id)
            ->where('user_id', $newUserId)
            ->first();

        if ($existingReferral) {
            return ['success' => false, 'error' => 'Already referred'];
        }

        $referralId = Capsule::table('mod_referral_referrals')->insertGetId([
            'referrer_id' => $referrer->id,
            'user_id' => $newUserId,
            'referral_code_used' => $referralCode,
            'status' => 'pending',
        ]);

        Capsule::table('mod_referral_referrers')
            ->where('id', $referrer->id)
            ->increment('referral_count');

        // Check for tier upgrade
        $this->checkTierUpgrade($referrer->user_id);

        return [
            'success' => true,
            'referral_id' => $referralId,
            'message' => 'Referral recorded successfully',
        ];
    }

    public function convertReferral(int $referralId, int $orderId, float $orderAmount): array {
        $referral = Capsule::table('mod_referral_referrals')->find($referralId);

        if (!$referral) {
            return ['success' => false, 'error' => 'Referral not found'];
        }

        $referrer = Capsule::table('mod_referral_referrers')->find($referral->referrer_id);
        $commission = $this->calculateCommission($referrer->user_id, $orderAmount);

        Capsule::table('mod_referral_referrals')->where('id', $referralId)->update([
            'status' => 'converted',
            'commission_amount' => $commission,
            'order_id' => $orderId,
            'converted_at' => date('Y-m-d H:i:s'),
        ]);

        Capsule::table('mod_referral_referrers')
            ->where('id', $referrer->id)
            ->increment('pending_commission', $commission);

        // Process multi-level commissions
        $this->processMultiLevelCommission($referrer->id, $orderAmount, $orderId);

        return [
            'success' => true,
            'commission' => $commission,
        ];
    }

    private function calculateCommission(int $userId, float $orderAmount): float {
        $referrer = Capsule::table('mod_referral_referrers')
            ->where('user_id', $userId)
            ->first();

        $tier = Capsule::table('mod_referral_tiers')
            ->where('id', $referrer->tier_id ?? 1)
            ->first();

        if (!$tier) {
            $tier = (object)['commission_type' => 'percentage', 'commission_value' => 5];
        }

        if ($tier->commission_type === 'percentage') {
            return $orderAmount * ($tier->commission_value / 100);
        }

        return $tier->commission_value;
    }

    private function processMultiLevelCommission(int $referrerId, float $orderAmount, int $orderId): void {
        $referrer = Capsule::table('mod_referral_referrers')->find($referrerId);

        if (!$referrer->parent_referrer_id) return;

        $parentReferrer = Capsule::table('mod_referral_referrers')->find($referrer->parent_referrer_id);

        if (!$parentReferrer) return;

        // Parent gets 10% of child's commission
        $parentCommission = $this->calculateCommission($parentReferrer->user_id, $orderAmount) * 0.1;

        Capsule::table('mod_referral_referrers')
            ->where('id', $parentReferrer->id)
            ->increment('pending_commission', $parentCommission);

        // Record multi-level referral
        Capsule::table('mod_referral_referrals')->insert([
            'referrer_id' => $parentReferrer->id,
            'user_id' => $referrer->user_id,
            'referral_code_used' => 'MLT',
            'status' => 'mlt_bonus',
            'commission_amount' => $parentCommission,
            'order_id' => $orderId,
        ]);
    }

    private function checkTierUpgrade(int $userId): void {
        $referrer = Capsule::table('mod_referral_referrers')
            ->where('user_id', $userId)
            ->first();

        if (!$referrer) return;

        $nextTier = Capsule::table('mod_referral_tiers')
            ->where('min_referrals', '<=', $referrer->referral_count)
            ->where('level', '>', Capsule::raw("(SELECT level FROM mod_referral_tiers WHERE id = " . ($referrer->tier_id ?? 1) . ")"))
            ->orderBy('level', 'desc')
            ->first();

        if ($nextTier) {
            Capsule::table('mod_referral_referrers')
                ->where('id', $referrer->id)
                ->update(['tier_id' => $nextTier->id]);

            $user = Capsule::table('tblclients')->find($userId);
            sendTplEmail($user->email, 'referral_tier_upgrade', [
                'tier_name' => $nextTier->name,
                'new_commission_rate' => $nextTier->commission_value,
            ]);
        }
    }

    public function requestPayout(int $referrerId, string $method, float $amount): array {
        $referrer = Capsule::table('mod_referral_referrers')->find($referrerId);

        if ($amount > $referrer->pending_commission) {
            return ['success' => false, 'error' => 'Amount exceeds pending commission'];
        }

        $minPayout = Capsule::table('tblconfig')->where('setting', 'referral_min_payout')->first();
        $minPayout = $minPayout ? $minPayout->value : 50;

        if ($amount < $minPayout) {
            return ['success' => false, 'error' => "Minimum payout is $minPayout"];
        }

        $paymentId = Capsule::table('mod_referral_payments')->insertGetId([
            'referrer_id' => $referrerId,
            'amount' => $amount,
            'payment_method' => $method,
            'status' => 'pending',
        ]);

        Capsule::table('mod_referral_referrers')
            ->where('id', $referrerId)
            ->update([
                'pending_commission' => $referrer->pending_commission - $amount,
                'total_commission' => $referrer->total_commission + $amount,
            ]);

        return ['success' => true, 'payment_id' => $paymentId];
    }

    public function getReferrerStats(int $userId): array {
        $referrer = Capsule::table('mod_referral_referrers')
            ->where('user_id', $userId)
            ->first();

        if (!$referrer) {
            return [
                'referral_code' => null,
                'referral_count' => 0,
                'total_commission' => 0,
                'pending_commission' => 0,
            ];
        }

        $tier = Capsule::table('mod_referral_tiers')
            ->where('id', $referrer->tier_id ?? 1)
            ->first();

        return [
            'referral_code' => $referrer->referral_code,
            'referral_count' => $referrer->referral_count,
            'total_commission' => $referrer->total_commission,
            'pending_commission' => $referrer->pending_commission,
            'tier' => $tier,
            'next_tier' => $this->getNextTier($tier->level),
        ];
    }

    private function getNextTier(int $currentLevel): ?object {
        return Capsule::table('mod_referral_tiers')
            ->where('level', $currentLevel + 1)
            ->first();
    }

    public function getLeaderboard(string $period = 'monthly'): array {
        return Capsule::table('mod_referral_leaderboard')
            ->join('tblclients', 'mod_referral_leaderboard.user_id', '=', 'tblclients.id')
            ->where('period', $period)
            ->orderBy('rank')
            ->limit(10)
            ->get(['mod_referral_leaderboard.*', 'tblclients.firstname', 'tblclients.lastname']);
    }
}
```

## Registration Hook

```php
<?php
add_hook('ClientAdd', 1, function($vars) {
    $userId = $vars['userid'];
    $code = $_COOKIE['referral_code'] ?? $_SESSION['referral_code'] ?? null;

    if ($code) {
        $referralManager = new ReferralManager();
        $result = $referralManager->processReferral($code, $userId);

        if ($result['success']) {
            logActivity("Referral processed for user {$userId} with code {$code}");
        }
    }
});

add_hook('OrderPaid', 1, function($vars) {
    $orderId = $vars['orderid'];
    $order = Capsule::table('tblorders')->find($orderId);

    $referralManager = new ReferralManager();

    $referral = Capsule::table('mod_referral_referrals')
        ->where('user_id', $order->userid)
        ->where('status', 'pending')
        ->first();

    if ($referral) {
        $referralManager->convertReferral($referral->id, $orderId, $order->amount);
    }
});
```

## Client Area

```php
<?php
function referral_system_clientarea(array $vars): array {
    $userId = $_SESSION['uid'];
    $manager = new ReferralManager();

    $stats = $manager->getReferrerStats($userId);

    if (!$stats['referral_code']) {
        $manager->createReferrer($userId);
        $stats = $manager->getReferrerStats($userId);
    }

    $referrals = Capsule::table('mod_referral_referrals')
        ->join('tblclients', 'mod_referral_referrals.user_id', '=', 'tblclients.id')
        ->where('mod_referral_referrals.referrer_id', Capsule::table('mod_referral_referrers')
            ->where('user_id', $userId)->first()->id ?? 0)
        ->get(['mod_referral_referrals.*', 'tblclients.email']);

    $leaderboard = $manager->getLeaderboard();

    return [
        'pagetitle' => 'Referral Program',
        'templatefile' => 'referral',
        'vars' => [
            'referral_code' => $stats['referral_code'],
            'referral_count' => $stats['referral_count'],
            'total_commission' => $stats['total_commission'],
            'pending_commission' => $stats['pending_commission'],
            'tier' => $stats['tier'],
            'referrals' => $referrals,
            'leaderboard' => $leaderboard,
        ],
    ];
}
```

---

**Related Skills:**
- whmcs-loyalty-program
- whmcs-affiliate-module
- whmcs-store-credit