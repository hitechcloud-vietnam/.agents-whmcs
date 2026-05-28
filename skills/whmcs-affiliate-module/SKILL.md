# WHMCS Affiliate Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building affiliate tracking and commission management modules.

## When to Use

- Creating affiliate program modules
- Building referral tracking systems
- Managing commission calculations

## Affiliate Module Patterns

```php
<?php
// modules/addons/{affiliatemodule}/{affiliatemodule}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {affiliatemodule}_config(): array {
    return [
        'name' => 'Affiliate Manager',
        'description' => 'Advanced affiliate program management',
        'version' => '1.0',
        'author' => 'Author',
        'commission_type' => ['FriendlyName' => 'Commission Type', 'Type' => 'dropdown',
            'Options' => 'percentage,fixed,both'],
        'default_commission' => ['FriendlyName' => 'Default Commission (%)', 'Type' => 'text', 'Default' => '10'],
        'second_tier_enabled' => ['FriendlyName' => 'Enable Second Tier', 'Type' => 'yesno'],
    ];
}

function {affiliatemodule}_activate(): array {
    Capsule::schema()->create('mod_affiliate_referrals', function($t) {
        $t->increments('id');
        $t->integer('affiliate_id')->unsigned();
        $t->integer('referred_user_id')->unsigned();
        $t->integer('order_id')->unsigned();
        $t->decimal('order_amount', 10, 2);
        $t->decimal('commission', 10, 2);
        $t->string('commission_type', 20);
        $t->string('status', 20)->default('pending');
        $t->timestamp('created_at');
        $t->timestamp('paid_at')->nullable();
    });

    Capsule::schema()->create('mod_affiliate_payouts', function($t) {
        $t->increments('id');
        $t->integer('affiliate_id')->unsigned();
        $t->decimal('amount', 10, 2);
        $t->string('method', 50);
        $t->string('status', 20)->default('pending');
        $t->string('transaction_id', 100)->nullable();
        $t->text('notes')->nullable();
        $t->timestamp('created_at');
        $t->timestamp('processed_at')->nullable();
    });

    Capsule::schema()->create('mod_affiliate_tiers', function($t) {
        $t->increments('id');
        $t->integer('parent_affiliate_id')->unsigned();
        $t->integer('child_affiliate_id')->unsigned();
        $t->integer('level')->unsigned();
        $t->timestamp('created_at');
    });

    Capsule::schema()->create('mod_affiliate_commissions', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('type', 50);
        $t->decimal('rate', 10, 2)->default(0);
        $t->decimal('fixed_amount', 10, 2)->default(0);
        $t->boolean('active')->default(true);
        $t->timestamps();
    });

    // Insert default commission rules
    Capsule::table('mod_affiliate_commissions')->insert([
        ['name' => 'New Customer', 'type' => 'percentage', 'rate' => 10],
        ['name' => 'Product Addon', 'type' => 'percentage', 'rate' => 5],
        ['name' => 'Renewal', 'type' => 'percentage', 'rate' => 3],
    ]);

    return ['status' => 'success', 'description' => 'Affiliate module activated'];
}

function {affiliatemodule}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_affiliate_referrals');
    Capsule::schema()->dropIfExists('mod_affiliate_payouts');
    Capsule::schema()->dropIfExists('mod_affiliate_tiers');
    Capsule::schema()->dropIfExists('mod_affiliate_Commissions');
    return ['status' => 'success', 'description' => 'Affiliate module deactivated'];
}
```

### Affiliate Registration

```php
function registerAffiliate(string $userId): int {
    // Check if already registered
    $existing = Capsule::table('mod_affiliate_referrals')
        ->where('affiliate_id', $userId)
        ->where('referred_user_id', $userId)
        ->first();

    if ($existing) {
        return $userId;
    }

    return $userId; // Return user ID as affiliate ID
}

function generateAffiliateLink(int $affiliateId, string $url): string {
    return $url . '?ref=' . base64_encode($affiliateId);
}

function trackReferral(int $affiliateId, int $orderId): void {
    $order = Capsule::table('tblorders')->where('id', $orderId)->first();

    // Check if referral already tracked for this order
    $existing = Capsule::table('mod_affiliate_referrals')
        ->where('order_id', $orderId)
        ->first();

    if ($existing) {
        return;
    }

    // Calculate commission
    $commission = calculateCommission($affiliateId, $order->totalamount);

    Capsule::table('mod_affiliate_referrals')->insert([
        'affiliate_id' => $affiliateId,
        'referred_user_id' => $order->userid,
        'order_id' => $orderId,
        'order_amount' => $order->totalamount,
        'commission' => $commission,
        'commission_type' => 'percentage',
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    // Second tier commission
    $secondTierEnabled = Capsule::table('mod_configuration')
        ->where('setting', 'second_tier_enabled')
        ->value('value');

    if ($secondTierEnabled) {
        processSecondTierCommission($affiliateId, $orderId, $commission);
    }
}

function calculateCommission(int $affiliateId, float $orderAmount): float {
    $defaultRate = (float) Capsule::table('mod_configuration')
        ->where('setting', 'default_commission')
        ->value('value') ?? 10;

    return ($orderAmount * $defaultRate) / 100;
}
```

### Commission Processing

```php
function processSecondTierCommission(int $affiliateId, int $orderId, float $commission): void {
    $parentTier = Capsule::table('mod_affiliate_tiers')
        ->where('child_affiliate_id', $affiliateId)
        ->where('level', 1)
        ->first();

    if ($parentTier) {
        $secondTierCommission = $commission * 0.05; // 5% of first tier commission

        Capsule::table('mod_affiliate_referrals')->insert([
            'affiliate_id' => $parentTier->parent_affiliate_id,
            'referred_user_id' => $affiliateId,
            'order_id' => $orderId,
            'order_amount' => 0,
            'commission' => $secondTierCommission,
            'commission_type' => 'tier',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

function getAffiliateStats(int $affiliateId): array {
    $referrals = Capsule::table('mod_affiliate_referrals')
        ->where('affiliate_id', $affiliateId);

    return [
        'total_referrals' => $referrals->count(),
        'total_referrals_pending' => (clone $referrals)->where('status', 'pending')->count(),
        'total_referrals_converted' => (clone $referrals)->where('status', 'paid')->count(),
        'pending_commission' => (clone $referrals)->where('status', 'pending')->sum('commission'),
        'total_commission' => (clone $referrals)->where('status', 'paid')->sum('commission'),
        'total_sales' => (clone $referrals)->sum('order_amount'),
    ];
}

function processPayout(int $affiliateId, float $amount, string $method): int {
    return Capsule::table('mod_affiliate_payouts')->insertGetId([
        'affiliate_id' => $affiliateId,
        'amount' => $amount,
        'method' => $method,
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

function markCommissionsAsPaid(int $affiliateId, int $payoutId): void {
    Capsule::table('mod_affiliate_referrals')
        ->where('affiliate_id', $affiliateId)
        ->where('status', 'pending')
        ->update([
            'status' => 'paid',
            'paid_at' => date('Y-m-d H:i:s'),
        ]);

    Capsule::table('mod_affiliate_payouts')
        ->where('id', $payoutId)
        ->update([
            'status' => 'completed',
            'processed_at' => date('Y-m-d H:i:s'),
        ]);
}
```

### Hook Integration

```php
add_hook('OrderPaid', 1, function($vars) {
    // Check if order came from affiliate referral
    $order = Capsule::table('tblorders')->where('id', $vars['order_id'])->first();

    // Track referral via cookie or session
    $affiliateId = $_COOKIE['affiliate_ref'] ?? null;

    if ($affiliateId) {
        trackReferral((int)$affiliateId, $vars['order_id']);
    }
});

add_hook('ClientAdd', 1, function($vars) {
    // Link new client to referred affiliate if applicable
    $userId = $vars['userid'];

    // Update referred_user_id if this is a new referred customer
    $existing = Capsule::table('mod_affiliate_referrals')
        ->where('referred_user_id', $userId)
        ->where('order_id', 0)
        ->first();

    if (!$existing) {
        // Check if client came from affiliate link
        $affiliateSession = $_SESSION['affiliate_ref'] ?? null;
        if ($affiliateSession) {
            $orderCount = Capsule::table('tblorders')
                ->where('userid', $userId)
                ->count();

            if ($orderCount == 1) {
                $firstOrder = Capsule::table('tblorders')
                    ->where('userid', $userId)
                    ->orderBy('id', 'asc')
                    ->first();

                Capsule::table('mod_affiliate_referrals')
                    ->where('referred_user_id', $userId)
                    ->where('order_id', $firstOrder->id)
                    ->update(['referred_user_id' => $userId]);
            }
        }
    }
});
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-reporting
- whmcs-cron-automation
