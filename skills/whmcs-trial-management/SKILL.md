# WHMCS Trial Management Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Manage trial periods for products and services with automatic conversion.

## Database Schema

```php
<?php
// modules/addons/trial_management/trial_management.php

use WHMCS\Database\Capsule;

function trial_management_config(): array {
    return [
        'name' => 'Trial Management',
        'description' => 'Manage trial periods and conversions',
        'version' => '1.0',
    ];
}

function trial_management_activate(): array {
    Capsule::schema()->create('mod_trial_products', function($t) {
        $t->increments('id');
        $t->integer('product_id')->unsigned();
        $t->integer('trial_days')->unsigned()->default(14);
        $t->decimal('trial_price', 10, 2)->default(0);
        $t->string('billing_cycle_after_trial', 30)->default('monthly');
        $t->boolean('require_payment_method')->default(true);
        $t->text('trial_terms')->nullable();
        $t->integer('max_trials_per_user')->default(1);
        $t->boolean('auto_convert')->default(true);
        $t->integer('convert_grace_days')->default(3);
        $t->boolean('is_active')->default(true);
    });

    Capsule::schema()->create('mod_trial_instances', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->integer('product_id')->unsigned();
        $t->integer('service_id')->unsigned()->nullable();
        $t->integer('trial_config_id')->unsigned();
        $t->date('start_date');
        $t->date('end_date');
        $t->string('status', 20)->default('active');
        $t->date('converted_at')->nullable();
        $t->integer('billing_cycle_id')->nullable();
        $t->decimal('setup_fee_waived', 10, 2)->default(0);
        $t->text('conversion_notes')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_trial_activities', function($t) {
        $t->increments('id');
        $t->integer('trial_id')->unsigned();
        $t->string('activity_type', 50);
        $t->text('description')->nullable();
        $t->json('metadata')->nullable();
        $t->timestamp('created_at')->useCurrent();
    });

    Capsule::schema()->create('mod_trial_limits', function($t) {
        $t->increments('id');
        $t->integer('trial_config_id')->unsigned();
        $t->string('limit_type', 50);
        $t->string('resource', 100);
        $t->decimal('limit_value', 12, 2)->default(0);
        $t->boolean('hard_limit')->default(false);
    });

    Capsule::schema()->create('mod_trial_usage', function($t) {
        $t->increments('id');
        $t->integer('trial_id')->unsigned();
        $t->integer('limit_id')->unsigned();
        $t->decimal('current_usage', 12, 2)->default(0);
        $t->timestamp('updated_at')->useCurrent();
    });

    return ['status' => 'success'];
}

function trial_management_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_trial_usage');
    Capsule::schema()->dropIfExists('mod_trial_limits');
    Capsule::schema()->dropIfExists('mod_trial_activities');
    Capsule::schema()->dropIfExists('mod_trial_instances');
    Capsule::schema()->dropIfExists('mod_trial_products');
    return ['status' => 'success'];
}
```

## Trial Manager

```php
<?php
class TrialManager {
    public function configureTrialProduct(int $productId, array $config): int {
        $existing = Capsule::table('mod_trial_products')
            ->where('product_id', $productId)
            ->first();

        if ($existing) {
            Capsule::table('mod_trial_products')
                ->where('product_id', $productId)
                ->update([
                    'trial_days' => $config['trial_days'] ?? 14,
                    'trial_price' => $config['trial_price'] ?? 0,
                    'billing_cycle_after_trial' => $config['billing_cycle'] ?? 'monthly',
                    'require_payment_method' => $config['require_payment'] ?? true,
                    'auto_convert' => $config['auto_convert'] ?? true,
                    'convert_grace_days' => $config['grace_days'] ?? 3,
                ]);

            return $existing->id;
        }

        return Capsule::table('mod_trial_products')->insertGetId([
            'product_id' => $productId,
            'trial_days' => $config['trial_days'] ?? 14,
            'trial_price' => $config['trial_price'] ?? 0,
            'billing_cycle_after_trial' => $config['billing_cycle'] ?? 'monthly',
            'require_payment_method' => $config['require_payment'] ?? true,
            'auto_convert' => $config['auto_convert'] ?? true,
            'convert_grace_days' => $config['grace_days'] ?? 3,
        ]);
    }

    public function startTrial(int $userId, int $productId, array $options = []): array {
        $config = Capsule::table('mod_trial_products')
            ->where('product_id', $productId)
            ->where('is_active', 1)
            ->first();

        if (!$config) {
            return ['success' => false, 'error' => 'Trial not available for this product'];
        }

        // Check user trial limit
        $userTrialCount = Capsule::table('mod_trial_instances')
            ->where('user_id', $userId)
            ->where('product_id', $productId)
            ->count();

        if ($userTrialCount >= $config->max_trials_per_user) {
            return ['success' => false, 'error' => 'Maximum trials exceeded'];
        }

        // Check for active trial
        $activeTrial = $this->getActiveTrial($userId, $productId);
        if ($activeTrial) {
            return ['success' => false, 'error' => 'Active trial already exists'];
        }

        $startDate = $options['start_date'] ?? date('Y-m-d');
        $endDate = date('Y-m-d', strtotime($startDate . ' + ' . $config->trial_days . ' days'));

        // Create service with trial
        $serviceParams = [
            'clientid' => $userId,
            'pid' => $productId,
            'billingcycle' => 'onetime',
            'regdate' => $startDate,
            'nextduedate' => $endDate,
        ];

        $orderResult = localAPI('AddOrder', $serviceParams);

        if ($orderResult['result'] !== 'success') {
            return ['success' => false, 'error' => 'Failed to create trial service'];
        }

        $serviceId = $orderResult['serviceids'][0] ?? null;

        // Create trial instance
        $trialId = Capsule::table('mod_trial_instances')->insertGetId([
            'user_id' => $userId,
            'product_id' => $productId,
            'service_id' => $serviceId,
            'trial_config_id' => $config->id,
            'start_date' => $startDate,
            'end_date' => $endDate,
            'status' => 'active',
        ]);

        // Initialize limits
        $this->initializeTrialLimits($trialId, $config->id);

        // Record activity
        $this->logActivity($trialId, 'trial_started', 'Trial period started');

        return [
            'success' => true,
            'trial_id' => $trialId,
            'service_id' => $serviceId,
            'end_date' => $endDate,
        ];
    }

    public function convertTrial(int $trialId, string $billingCycle = 'monthly'): array {
        $trial = Capsule::table('mod_trial_instances')->find($trialId);

        if (!$trial || $trial->status !== 'active') {
            return ['success' => false, 'error' => 'Invalid trial'];
        }

        $config = Capsule::table('mod_trial_products')->find($trial->trial_config_id);

        // Update service billing cycle
        if ($trial->service_id) {
            Capsule::table('tblhosting')
                ->where('id', $trial->service_id)
                ->update([
                    'billingcycle' => $billingCycle,
                    'nextduedate' => date('Y-m-d'),
                ]);
        }

        // Create first paid invoice if trial had cost
        $invoiceCreated = false;
        if ($config->trial_price > 0) {
            $invoiceResult = $this->createConversionInvoice($trial, $config, $billingCycle);
            $invoiceCreated = $invoiceResult['success'];
        }

        // Update trial status
        Capsule::table('mod_trial_instances')
            ->where('id', $trialId)
            ->update([
                'status' => 'converted',
                'converted_at' => date('Y-m-d H:i:s'),
                'billing_cycle_id' => $this->getBillingCycleId($billingCycle),
            ]);

        $this->logActivity($trialId, 'trial_converted', "Converted to {$billingCycle} billing");

        return [
            'success' => true,
            'trial_id' => $trialId,
            'invoice_created' => $invoiceCreated,
        ];
    }

    private function createConversionInvoice($trial, $config, string $billingCycle): array {
        $product = Capsule::table('tblproducts')->find($trial->product_id);
        $pricing = Capsule::table('tblpricing')
            ->where('type', 'product')
            ->where('relid', $trial->product_id)
            ->first();

        $amount = $pricing->{$billingCycle . 'ly'} ?? $pricing->monthly ?? 0;

        $params = [
            'userid' => $trial->user_id,
            'date' => date('Y-m-d'),
            'duedate' => date('Y-m-d'),
            'itemdescription' => [
                "{$product->name} - {$billingCycle} (After Trial)",
            ],
            'itemamount' => [$amount],
        ];

        $result = localAPI('CreateInvoice', $params);

        if ($result['result'] === 'success') {
            Capsule::table('mod_trial_instances')
                ->where('id', $trial->id)
                ->update(['conversion_notes' => "Invoice #{$result['invoiceid']} created"]);
        }

        return $result;
    }

    public function cancelTrial(int $trialId, string $reason = ''): array {
        $trial = Capsule::table('mod_trial_instances')->find($trialId);

        if (!$trial) {
            return ['success' => false, 'error' => 'Trial not found'];
        }

        // Terminate service
        if ($trial->service_id) {
            localAPI('CancelService', [
                'serviceid' => $trial->service_id,
                'immediate' => true,
            ]);
        }

        Capsule::table('mod_trial_instances')
            ->where('id', $trialId)
            ->update([
                'status' => 'cancelled',
                'conversion_notes' => $reason,
            ]);

        $this->logActivity($trialId, 'trial_cancelled', $reason);

        return ['success' => true];
    }

    private function initializeTrialLimits(int $trialId, int $configId): void {
        $limits = Capsule::table('mod_trial_limits')
            ->where('trial_config_id', $configId)
            ->get();

        foreach ($limits as $limit) {
            Capsule::table('mod_trial_usage')->insert([
                'trial_id' => $trialId,
                'limit_id' => $limit->id,
                'current_usage' => 0,
            ]);
        }
    }

    public function checkLimit(int $trialId, string $resource): array {
        $limit = Capsule::table('mod_trial_limits')
            ->join('mod_trial_products', 'mod_trial_limits.trial_config_id', '=', 'mod_trial_products.id')
            ->where('mod_trial_products.product_id', function($q) {
                $q->select('product_id')
                  ->from('mod_trial_instances')
                  ->where('id', request()->get('trial_id'));
            })
            ->where('mod_trial_limits.resource', $resource)
            ->first();

        if (!$limit) {
            return ['limited' => false];
        }

        $usage = Capsule::table('mod_trial_usage')
            ->where('trial_id', $trialId)
            ->where('limit_id', $limit->id)
            ->first();

        $currentUsage = $usage->current_usage ?? 0;

        if ($limit->hard_limit && $currentUsage >= $limit->limit_value) {
            return [
                'limited' => true,
                'blocked' => true,
                'resource' => $resource,
                'limit' => $limit->limit_value,
                'current' => $currentUsage,
            ];
        }

        return [
            'limited' => true,
            'blocked' => false,
            'resource' => $resource,
            'limit' => $limit->limit_value,
            'current' => $currentUsage,
            'remaining' => $limit->limit_value - $currentUsage,
        ];
    }

    public function getActiveTrial(int $userId, int $productId): ?object {
        return Capsule::table('mod_trial_instances')
            ->where('user_id', $userId)
            ->where('product_id', $productId)
            ->where('status', 'active')
            ->first();
    }

    public function getTrialAnalytics(): array {
        $stats = [
            'active_trials' => Capsule::table('mod_trial_instances')
                ->where('status', 'active')
                ->count(),
            'converted_trials' => Capsule::table('mod_trial_instances')
                ->where('status', 'converted')
                ->count(),
            'cancelled_trials' => Capsule::table('mod_trial_instances')
                ->where('status', 'cancelled')
                ->count(),
        ];

        $stats['conversion_rate'] = $stats['active_trials'] > 0
            ? ($stats['converted_trials'] / ($stats['converted_trials'] + $stats['cancelled_trials'] + 1)) * 100
            : 0;

        return $stats;
    }

    private function logActivity(int $trialId, string $type, string $description): void {
        Capsule::table('mod_trial_activities')->insert([
            'trial_id' => $trialId,
            'activity_type' => $type,
            'description' => $description,
        ]);
    }

    private function getBillingCycleId(string $cycle): int {
        return match($cycle) {
            'monthly' => 1,
            'quarterly' => 2,
            'annually' => 3,
            'biennially' => 4,
            default => 1,
        };
    }
}
```

## Cron Processing

```php
<?php
add_hook('DailyCronJob', 1, function($vars) {
    $trialManager = new TrialManager();

    // Convert expired trials
    $expiredTrials = Capsule::table('mod_trial_instances')
        ->where('status', 'active')
        ->where('end_date', '<', date('Y-m-d'))
        ->get();

    foreach ($expiredTrials as $trial) {
        $config = Capsule::table('mod_trial_products')->find($trial->trial_config_id);

        if ($config->auto_convert) {
            $trialManager->convertTrial($trial->id, $config->billing_cycle_after_trial);
        } else {
            Capsule::table('mod_trial_instances')
                ->where('id', $trial->id)
                ->update(['status' => 'expired_pending']);
        }
    }

    // Grace period expired - convert or cancel
    $graceExpired = Capsule::table('mod_trial_instances')
        ->where('status', 'expired_pending')
        ->where('end_date', '<', date('Y-m-d', strtotime('-3 days')))
        ->get();

    foreach ($graceExpired as $trial) {
        $trialManager->cancelTrial($trial->id, 'Grace period expired without conversion');
    }
});
```

---

**Related Skills:**
- whmcs-freemium-model
- whmcs-pricing-strategy
- whmcs-billing-dashboard