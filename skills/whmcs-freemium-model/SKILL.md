# WHMCS Freemium Model Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Implement freemium business model with free tier and upgrade paths.

## Database Schema

```php
<?php
// modules/addons/freemium_model/freemium_model.php

use WHMCS\Database\Capsule;

function freemium_model_config(): array {
    return [
        'name' => 'Freemium Model',
        'description' => 'Freemium business model implementation',
        'version' => '1.0',
    ];
}

function freemium_model_activate(): array {
    Capsule::schema()->create('mod_freemium_plans', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('tier', 20)->default('free');
        $t->text('description')->nullable();
        $t->boolean('is_default')->default(false);
        $t->integer('sort_order')->default(0);
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_freemium_features', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->text('description')->nullable();
        $t->string('feature_key', 50)->unique();
        $t->string('feature_type', 30)->default('boolean');
        $t->decimal('limit_value', 12, 2)->default(0);
        $t->string('unit', 30)->nullable();
        $t->boolean('is_active')->default(true);
    });

    Capsule::schema()->create('mod_freemium_plan_features', function($t) {
        $t->increments('id');
        $t->integer('plan_id')->unsigned();
        $t->integer('feature_id')->unsigned();
        $t->decimal('value', 12, 2)->default(1);
        $t->boolean('is_included')->default(true);
    });

    Capsule::schema()->create('mod_freemium_user_plans', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned()->unique();
        $t->integer('plan_id')->unsigned();
        $t->string('status', 20)->default('active');
        $t->date('started_at')->nullable();
        $t->date('upgraded_at')->nullable();
        $t->date('downgraded_at')->nullable();
        $t->timestamp('created_at')->useCurrent();
    });

    Capsule::schema()->create('mod_freemium_upgrade_paths', function($t) {
        $t->increments('id');
        $t->integer('from_plan_id')->unsigned();
        $t->integer('to_plan_id')->unsigned();
        $t->string('trigger_type', 30)->default('manual');
        $t->text('upgrade_bonus')->nullable();
        $t->boolean('is_active')->default(true);
    });

    Capsule::schema()->create('mod_freemium_usage', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->string('feature_key', 50);
        $t->decimal('usage_value', 12, 2)->default(0);
        $t->string('period', 20)->default('monthly');
        $t->timestamp('period_start')->nullable();
        $t->timestamp('updated_at')->useCurrent();

        $t->unique(['user_id', 'feature_key', 'period']);
    });

    // Insert default plans
    $plans = [
        ['name' => 'Free', 'tier' => 'free', 'is_default' => true, 'sort_order' => 0],
        ['name' => 'Starter', 'tier' => 'starter', 'sort_order' => 1],
        ['name' => 'Professional', 'tier' => 'professional', 'sort_order' => 2],
        ['name' => 'Enterprise', 'tier' => 'enterprise', 'sort_order' => 3],
    ];

    foreach ($plans as $plan) {
        Capsule::table('mod_freemium_plans')->insert($plan);
    }

    return ['status' => 'success'];
}

function freemium_model_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_freemium_usage');
    Capsule::schema()->dropIfExists('mod_freemium_upgrade_paths');
    Capsule::schema()->dropIfExists('mod_freemium_user_plans');
    Capsule::schema()->dropIfExists('mod_freemium_plan_features');
    Capsule::schema()->dropIfExists('mod_freemium_features');
    Capsule::schema()->dropIfExists('mod_freemium_plans');
    return ['status' => 'success'];
}
```

## Freemium Manager

```php
<?php
class FreemiumManager {
    public function assignPlan(int $userId, int $planId, string $status = 'active'): array {
        $userPlan = Capsule::table('mod_freemium_user_plans')
            ->where('user_id', $userId)
            ->first();

        if ($userPlan) {
            Capsule::table('mod_freemium_user_plans')
                ->where('user_id', $userId)
                ->update([
                    'plan_id' => $planId,
                    'status' => $status,
                    'upgraded_at' => $status === 'active' ? date('Y-m-d') : null,
                ]);
        } else {
            Capsule::table('mod_freemium_user_plans')->insert([
                'user_id' => $userId,
                'plan_id' => $planId,
                'status' => $status,
                'started_at' => date('Y-m-d'),
            ]);
        }

        return ['success' => true];
    }

    public function getUserPlan(int $userId): ?object {
        $userPlan = Capsule::table('mod_freemium_user_plans')
            ->where('user_id', $userId)
            ->where('status', 'active')
            ->first();

        if (!$userPlan) {
            // Assign default free plan
            $defaultPlan = Capsule::table('mod_freemium_plans')
                ->where('is_default', 1)
                ->first();

            if ($defaultPlan) {
                $this->assignPlan($userId, $defaultPlan->id);
                $userPlan = Capsule::table('mod_freemium_user_plans')
                    ->where('user_id', $userId)
                    ->first();
            }
        }

        if ($userPlan) {
            $userPlan->plan = Capsule::table('mod_freemium_plans')->find($userPlan->plan_id);
            $userPlan->features = $this->getPlanFeatures($userPlan->plan_id);
        }

        return $userPlan;
    }

    public function getPlanFeatures(int $planId): array {
        return Capsule::table('mod_freemium_plan_features')
            ->join('mod_freemium_features', 'mod_freemium_plan_features.feature_id', '=', 'mod_freemium_features.id')
            ->where('mod_freemium_plan_features.plan_id', $planId)
            ->get();
    }

    public function checkFeatureAccess(int $userId, string $featureKey): array {
        $userPlan = $this->getUserPlan($userId);

        if (!$userPlan) {
            return ['has_access' => false, 'reason' => 'No plan assigned'];
        }

        $feature = Capsule::table('mod_freemium_features')
            ->where('feature_key', $featureKey)
            ->first();

        if (!$feature) {
            return ['has_access' => false, 'reason' => 'Feature not found'];
        }

        $planFeature = Capsule::table('mod_freemium_plan_features')
            ->where('plan_id', $userPlan->plan_id)
            ->where('feature_id', $feature->id)
            ->first();

        if (!$planFeature || !$planFeature->is_included) {
            // Check if it's a limit-based feature
            if ($feature->feature_type === 'limited') {
                $usage = $this->getFeatureUsage($userId, $featureKey);
                $limit = $planFeature ? $planFeature->value : 0;

                if ($usage >= $limit) {
                    return [
                        'has_access' => false,
                        'feature_type' => 'limited',
                        'limit' => $limit,
                        'usage' => $usage,
                        'upgrade_required' => true,
                    ];
                }

                return [
                    'has_access' => true,
                    'feature_type' => 'limited',
                    'limit' => $limit,
                    'usage' => $usage,
                    'remaining' => $limit - $usage,
                ];
            }

            return [
                'has_access' => false,
                'reason' => 'Feature not included in plan',
                'upgrade_required' => true,
            ];
        }

        if ($planFeature->value == 1) {
            return ['has_access' => true, 'feature_type' => 'boolean'];
        }

        return [
            'has_access' => true,
            'feature_type' => 'limited',
            'limit' => $planFeature->value,
            'usage' => $this->getFeatureUsage($userId, $featureKey),
        ];
    }

    public function trackUsage(int $userId, string $featureKey, float $value = 1): void {
        $period = date('Y-m');
        $periodStart = date('Y-m-01 00:00:00');

        $existing = Capsule::table('mod_freemium_usage')
            ->where('user_id', $userId)
            ->where('feature_key', $featureKey)
            ->where('period', $period)
            ->first();

        if ($existing) {
            Capsule::table('mod_freemium_usage')
                ->where('id', $existing->id)
                ->update([
                    'usage_value' => $existing->usage_value + $value,
                ]);
        } else {
            Capsule::table('mod_freemium_usage')->insert([
                'user_id' => $userId,
                'feature_key' => $featureKey,
                'usage_value' => $value,
                'period' => $period,
                'period_start' => $periodStart,
            ]);
        }
    }

    private function getFeatureUsage(int $userId, string $featureKey): float {
        $period = date('Y-m');

        $usage = Capsule::table('mod_freemium_usage')
            ->where('user_id', $userId)
            ->where('feature_key', $featureKey)
            ->where('period', $period)
            ->first();

        return $usage ? (float)$usage->usage_value : 0;
    }

    public function upgradePlan(int $userId, int $newPlanId): array {
        $userPlan = $this->getUserPlan($userId);

        if (!$userPlan) {
            return ['success' => false, 'error' => 'User has no current plan'];
        }

        $newPlan = Capsule::table('mod_freemium_plans')->find($newPlanId);

        if (!$newPlan || !$newPlan->is_active) {
            return ['success' => false, 'error' => 'Invalid plan'];
        }

        if ($newPlan->sort_order <= $userPlan->plan->sort_order) {
            return ['success' => false, 'error' => 'Invalid upgrade path'];
        }

        // Check for upgrade path bonus
        $upgradePath = Capsule::table('mod_freemium_upgrade_paths')
            ->where('from_plan_id', $userPlan->plan_id)
            ->where('to_plan_id', $newPlanId)
            ->where('is_active', 1)
            ->first();

        $bonusApplied = null;
        if ($upgradePath && $upgradePath->upgrade_bonus) {
            $bonusApplied = json_decode($upgradePath->upgrade_bonus, true);
        }

        Capsule::table('mod_freemium_user_plans')
            ->where('user_id', $userId)
            ->update([
                'status' => 'inactive',
            ]);

        Capsule::table('mod_freemium_user_plans')->insert([
            'user_id' => $userId,
            'plan_id' => $newPlanId,
            'status' => 'active',
            'started_at' => date('Y-m-d'),
            'upgraded_at' => date('Y-m-d'),
        ]);

        // Apply bonus credits if any
        if ($bonusApplied && isset($bonusApplied['credits'])) {
            $this->applyBonusCredits($userId, $bonusApplied['credits']);
        }

        return [
            'success' => true,
            'previous_plan' => $userPlan->plan->name,
            'new_plan' => $newPlan->name,
            'bonus' => $bonusApplied,
        ];
    }

    public function downgradePlan(int $userId, int $newPlanId): array {
        $userPlan = $this->getUserPlan($userId);

        if (!$userPlan) {
            return ['success' => false, 'error' => 'No current plan'];
        }

        $newPlan = Capsule::table('mod_freemium_plans')->find($newPlanId);

        if ($newPlan->sort_order >= $userPlan->plan->sort_order) {
            return ['success' => false, 'error' => 'Invalid downgrade path'];
        }

        Capsule::table('mod_freemium_user_plans')
            ->where('user_id', $userId)
            ->update(['status' => 'inactive']);

        Capsule::table('mod_freemium_user_plans')->insert([
            'user_id' => $userId,
            'plan_id' => $newPlanId,
            'status' => 'active',
            'started_at' => date('Y-m-d'),
            'downgraded_at' => date('Y-m-d'),
        ]);

        return [
            'success' => true,
            'new_plan' => $newPlan->name,
        ];
    }

    private function applyBonusCredits(int $userId, float $credits): void {
        // Add to wallet or store credit
        Capsule::table('tblcredit')
            ->insert([
                'userid' => $userId,
                'description' => 'Freemium upgrade bonus',
                'amount' => $credits,
                'date' => date('Y-m-d'),
            ]);
    }

    public function getAvailableUpgrades(int $userId): array {
        $userPlan = $this->getUserPlan($userId);

        if (!$userPlan) return [];

        return Capsule::table('mod_freemium_plans')
            ->where('is_active', 1)
            ->where('sort_order', '>', $userPlan->plan->sort_order)
            ->orderBy('sort_order')
            ->get();
    }

    public function getUpgradePath(int $fromPlanId, int $toPlanId): ?object {
        return Capsule::table('mod_freemium_upgrade_paths')
            ->where('from_plan_id', $fromPlanId)
            ->where('to_plan_id', $toPlanId)
            ->where('is_active', 1)
            ->first();
    }

    public function getConversionAnalytics(): array {
        $plans = Capsule::table('mod_freemium_plans')->get();
        $analytics = [];

        foreach ($plans as $plan) {
            $users = Capsule::table('mod_freemium_user_plans')
                ->where('plan_id', $plan->id)
                ->count();

            $upgrades = Capsule::table('mod_freemium_user_plans')
                ->where('plan_id', $plan->id)
                ->whereNotNull('upgraded_at')
                ->count();

            $analytics[] = [
                'plan' => $plan,
                'total_users' => $users,
                'upgrades' => $upgrades,
                'conversion_rate' => $users > 0 ? ($upgrades / $users) * 100 : 0,
            ];
        }

        return $analytics;
    }
}
```

## Feature Access Middleware

```php
<?php
class FreemiumFeatureMiddleware {
    public function checkAccess(int $userId, string $feature, ?callable $onDenied = null): bool {
        $manager = new FreemiumManager();
        $access = $manager->checkFeatureAccess($userId, $feature);

        if (!$access['has_access']) {
            if ($onDenied) {
                return $onDenied($access);
            }
            return false;
        }

        // Track usage
        $manager->trackUsage($userId, $feature);

        return true;
    }

    public function getRemainingQuota(int $userId, string $feature): array {
        $manager = new FreemiumManager();
        $access = $manager->checkFeatureAccess($userId, $feature);

        if ($access['feature_type'] === 'limited') {
            return [
                'limit' => $access['limit'],
                'usage' => $access['usage'],
                'remaining' => $access['remaining'] ?? 0,
            ];
        }

        return ['unlimited' => true];
    }
}
```

## Client Area

```php
<?php
function freemium_model_clientarea(array $vars): array {
    $userId = $_SESSION['uid'];
    $manager = new FreemiumManager();

    $userPlan = $manager->getUserPlan($userId);
    $upgrades = $manager->getAvailableUpgrades($userId);
    $analytics = $manager->getConversionAnalytics();

    $allPlans = Capsule::table('mod_freemium_plans')
        ->where('is_active', 1)
        ->orderBy('sort_order')
        ->get();

    foreach ($allPlans as $plan) {
        $plan->features = $manager->getPlanFeatures($plan->id);
    }

    return [
        'pagetitle' => 'My Plan',
        'templatefile' => 'freemium',
        'vars' => [
            'current_plan' => $userPlan->plan ?? null,
            'current_features' => $userPlan->features ?? [],
            'available_plans' => $allPlans,
            'upgrades' => $upgrades,
            'analytics' => $analytics,
        ],
    ];
}
```

---

**Related Skills:**
- whmcs-trial-management
- whmcs-tiered-pricing
- whmcs-usage-tracking