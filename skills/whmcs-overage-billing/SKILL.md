# WHMCS Overage Billing Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Handle overage charges when usage exceeds plan limits.

## Database Schema

```php
<?php
// modules/addons/overage_billing/overage_billing.php

use WHMCS\Database\Capsule;

function overage_billing_config(): array {
    return [
        'name' => 'Overage Billing',
        'description' => 'Overage charges for exceeded limits',
        'version' => '1.0',
    ];
}

function overage_billing_activate(): array {
    Capsule::schema()->create('mod_overage_plans', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->text('description')->nullable();
        $t->decimal('base_price', 10, 2);
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_overage_included', function($t) {
        $t->increments('id');
        $t->integer('plan_id')->unsigned();
        $t->string('resource', 100);
        $t->decimal('included_quantity', 12, 2)->default(0);
        $t->string('unit', 30)->nullable();
    });

    Capsule::schema()->create('mod_overage_rates', function($t) {
        $t->increments('id');
        $t->integer('plan_id')->unsigned();
        $t->string('resource', 100);
        $t->string('rate_type', 20)->default('per_unit');
        $t->decimal('rate', 10, 4);
        $t->text('tier_rates')->nullable();
        $t->decimal('cap_amount', 10, 2)->nullable();
    });

    Capsule::schema()->create('mod_overage_usage', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->integer('service_id')->unsigned();
        $t->string('resource', 100);
        $t->string('period', 20);
        $t->decimal('included', 12, 2)->default(0);
        $t->decimal('used', 12, 2)->default(0);
        $t->decimal('overage', 12, 2)->default(0);
        $t->decimal('overage_charge', 10, 2)->default(0);
        $t->timestamp('calculated_at')->nullable();
    });

    Capsule::schema()->create('mod_overage_breaks', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('resource', 100);
        $t->decimal('break_point', 12, 2);
        $t->decimal('discount_percentage', 5, 2)->default(0);
        $t->boolean('is_active')->default(true);
    });

    return ['status' => 'success'];
}

function overage_billing_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_overage_breaks');
    Capsule::schema()->dropIfExists('mod_overage_usage');
    Capsule::schema()->dropIfExists('mod_overage_rates');
    Capsule::schema()->dropIfExists('mod_overage_included');
    Capsule::schema()->dropIfExists('mod_overage_plans');
    return ['status' => 'success'];
}
```

## Overage Billing Manager

```php
<?php
class OverageBillingManager {
    public function calculateOverage(int $userId, int $serviceId, string $resource, float $usage, string $period): array {
        $plan = $this->getUserPlan($userId);

        if (!$plan) {
            return ['success' => false, 'error' => 'No plan found'];
        }

        $included = $this->getIncludedAmount($plan->id, $resource);
        $overage = max(0, $usage - $included);
        $rate = $this->getOverageRate($plan->id, $resource);

        $charge = 0;

        if ($overage > 0) {
            $charge = $this->calculateOverageCharge($plan->id, $resource, $overage, $rate);
        }

        // Record usage
        $this->recordOverage($userId, $serviceId, $resource, $period, $included, $usage, $overage, $charge);

        return [
            'success' => true,
            'included' => $included,
            'used' => $usage,
            'overage' => $overage,
            'rate' => $rate,
            'charge' => $charge,
        ];
    }

    private function getUserPlan(int $userId): ?object {
        $service = Capsule::table('tblhosting')
            ->where('userid', $userId)
            ->orderBy('id', 'desc')
            ->first();

        if (!$service) return null;

        return Capsule::table('mod_overage_plans')
            ->join('mod_overage_included', 'mod_overage_plans.id', '=', 'mod_overage_included.plan_id')
            ->where('mod_overage_plans.is_active', 1)
            ->first();
    }

    private function getIncludedAmount(int $planId, string $resource): float {
        $included = Capsule::table('mod_overage_included')
            ->where('plan_id', $planId)
            ->where('resource', $resource)
            ->first();

        return $included ? (float)$included->included_quantity : 0;
    }

    private function getOverageRate(int $planId, string $resource): float {
        $rate = Capsule::table('mod_overage_rates')
            ->where('plan_id', $planId)
            ->where('resource', $resource)
            ->first();

        return $rate ? (float)$rate->rate : 0;
    }

    private function calculateOverageCharge(int $planId, string $resource, float $overage, float $baseRate): float {
        $rateConfig = Capsule::table('mod_overage_rates')
            ->where('plan_id', $planId)
            ->where('resource', $resource)
            ->first();

        if (!$rateConfig || empty($rateConfig->tier_rates)) {
            return $overage * $baseRate;
        }

        // Tiered overage rates
        $tiers = json_decode($rateConfig->tier_rates, true);
        $charge = 0;
        $remaining = $overage;
        $prevBreak = 0;

        usort($tiers, fn($a, $b) => $a['break_point'] - $b['break_point']);

        foreach ($tiers as $tier) {
            if ($overage > $tier['break_point']) {
                $unitsInTier = $tier['break_point'] - $prevBreak;
                $charge += $unitsInTier * $tier['rate'];
                $remaining -= $unitsInTier;
                $prevBreak = $tier['break_point'];
            }
        }

        // Remaining units at last tier rate
        if ($remaining > 0 && !empty($tiers)) {
            $lastTier = end($tiers);
            $charge += $remaining * $lastTier['rate'];
        } elseif (empty($tiers)) {
            $charge = $overage * $baseRate;
        }

        // Apply cap if configured
        if ($rateConfig->cap_amount && $charge > $rateConfig->cap_amount) {
            $charge = $rateConfig->cap_amount;
        }

        // Apply break discounts
        $charge = $this->applyBreakDiscounts($resource, $charge);

        return $charge;
    }

    private function applyBreakDiscounts(string $resource, float $charge): float {
        $breaks = Capsule::table('mod_overage_breaks')
            ->where('resource', $resource)
            ->where('is_active', 1)
            ->orderBy('break_point', 'desc')
            ->get();

        foreach ($breaks as $break) {
            if ($charge >= $break->break_point) {
                $charge = $charge * (1 - $break->discount_percentage / 100);
                break;
            }
        }

        return $charge;
    }

    private function recordOverage(
        int $userId,
        int $serviceId,
        string $resource,
        string $period,
        float $included,
        float $used,
        float $overage,
        float $charge
    ): void {
        $existing = Capsule::table('mod_overage_usage')
            ->where('user_id', $userId)
            ->where('service_id', $serviceId)
            ->where('resource', $resource)
            ->where('period', $period)
            ->first();

        if ($existing) {
            Capsule::table('mod_overage_usage')
                ->where('id', $existing->id)
                ->update([
                    'used' => $used,
                    'overage' => $overage,
                    'overage_charge' => $charge,
                    'calculated_at' => date('Y-m-d H:i:s'),
                ]);
        } else {
            Capsule::table('mod_overage_usage')->insert([
                'user_id' => $userId,
                'service_id' => $serviceId,
                'resource' => $resource,
                'period' => $period,
                'included' => $included,
                'used' => $used,
                'overage' => $overage,
                'overage_charge' => $charge,
                'calculated_at' => date('Y-m-d H:i:s'),
            ]);
        }
    }

    public function generateOverageInvoice(int $userId, string $period): array {
        $usageRecords = Capsule::table('mod_overage_usage')
            ->where('user_id', $userId)
            ->where('period', $period)
            ->where('overage', '>', 0)
            ->get();

        if ($usageRecords->isEmpty()) {
            return ['success' => false, 'error' => 'No overage to bill'];
        }

        $lineItems = [];
        $totalAmount = 0;

        foreach ($usageRecords as $record) {
            if ($record->overage_charge > 0) {
                $lineItems[] = [
                    'description' => "Overage: {$record->resource} ({$record->overage} units)",
                    'amount' => $record->overage_charge,
                ];
                $totalAmount += $record->overage_charge;
            }
        }

        if (empty($lineItems)) {
            return ['success' => false, 'error' => 'No charges to invoice'];
        }

        $params = [
            'userid' => $userId,
            'date' => date('Y-m-d'),
            'duedate' => date('Y-m-d', strtotime('+7 days')),
            'itemdescription' => array_column($lineItems, 'description'),
            'itemamount' => array_column($lineItems, 'amount'),
        ];

        $result = localAPI('CreateInvoice', $params);

        if ($result['result'] === 'success') {
            foreach ($usageRecords as $record) {
                Capsule::table('mod_overage_usage')
                    ->where('id', $record->id)
                    ->update(['calculated_at' => date('Y-m-d H:i:s')]);
            }
        }

        return $result;
    }

    public function getOverageSummary(int $userId, string $period = 'current'): array {
        $periodStart = date('Y-m-01');
        $periodEnd = date('Y-m-t');

        if ($period === 'previous') {
            $periodStart = date('Y-m-01', strtotime('-1 month'));
            $periodEnd = date('Y-m-t', strtotime('-1 month'));
        }

        $usage = Capsule::table('mod_overage_usage')
            ->where('user_id', $userId)
            ->where('calculated_at', '>=', $periodStart)
            ->where('calculated_at', '<=', $periodEnd . ' 23:59:59')
            ->get();

        $totalCharge = $usage->sum('overage_charge');
        $totalOverage = $usage->sum('overage');

        return [
            'period' => $period,
            'period_start' => $periodStart,
            'period_end' => $periodEnd,
            'total_overage' => $totalOverage,
            'total_charge' => $totalCharge,
            'breakdown' => $usage,
        ];
    }

    public function getOveragePreview(int $userId, string $resource, float $projectedUsage): array {
        $plan = $this->getUserPlan($userId);

        if (!$plan) {
            return ['success' => false, 'error' => 'No plan found'];
        }

        $included = $this->getIncludedAmount($plan->id, $resource);
        $overage = max(0, $projectedUsage - $included);
        $rate = $this->getOverageRate($plan->id, $resource);

        $charge = $overage > 0 ? $this->calculateOverageCharge($plan->id, $resource, $overage, $rate) : 0;

        return [
            'included' => $included,
            'projected_usage' => $projectedUsage,
            'overage' => $overage,
            'rate' => $rate,
            'estimated_charge' => $charge,
        ];
    }
}
```

## Usage Integration

```php
<?php
add_hook('DailyCronJob', 1, function($vars) {
    $billingManager = new OverageBillingManager();
    $period = date('Y-m');

    // Calculate overage for all active services
    $services = Capsule::table('tblhosting')
        ->where('domainstatus', 'Active')
        ->get();

    foreach ($services as $service) {
        // Example: Calculate bandwidth overage
        $bandwidthUsed = getBandwidthUsage($service->id); // Your metric function

        $result = $billingManager->calculateOverage(
            $service->userid,
            $service->id,
            'bandwidth',
            $bandwidthUsed,
            $period
        );

        if ($result['success'] && $result['overage'] > 0) {
            logActivity("Overage calculated for user {$service->userid}: {$result['overage']} units, \${$result['charge']}");
        }
    }
});
```

---

**Related Skills:**
- whmcs-usage-tracking
- whmcs-commitment-billing
- whmcs-billing-dashboard