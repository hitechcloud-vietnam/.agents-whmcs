# WHMCS Usage Tracking Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Track and monitor usage metrics for metered billing.

## Database Schema

```php
<?php
// modules/addons/usage_tracking/usage_tracking.php

use WHMCS\Database\Capsule;

function usage_tracking_config(): array {
    return [
        'name' => 'Usage Tracking',
        'description' => 'Metered billing and usage tracking',
        'version' => '1.0',
    ];
}

function usage_tracking_activate(): array {
    Capsule::schema()->create('mod_usage_metrics', function($t) {
        $t->increments('id');
        $t->string('metric_key', 50)->unique();
        $t->string('name', 100);
        $t->text('description')->nullable();
        $t->string('unit', 30);
        $t->string('aggregation', 30)->default('sum');
        $t->string('pricing_type', 20)->default('per_unit');
        $t->decimal('base_price', 10, 4)->default(0);
        $t->text('tier_pricing')->nullable();
        $t->boolean('bill_in_arrears')->default(true);
        $t->boolean('is_active')->default(true);
    });

    Capsule::schema()->create('mod_usage_records', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->integer('service_id')->unsigned();
        $t->string('metric_key', 50);
        $t->decimal('quantity', 12, 4)->default(0);
        $t->string('period', 20);
        $t->date('record_date');
        $t->timestamp('recorded_at')->useCurrent();
    });

    Capsule::schema()->create('mod_usage_billing', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->integer('service_id')->unsigned();
        $t->string('period', 20);
        $t->date('billing_period_start');
        $t->date('billing_period_end');
        $t->decimal('total_amount', 10, 2)->default(0);
        $t->text('line_items')->nullable();
        $t->string('status', 20)->default('pending');
        $t->integer('invoice_id')->unsigned()->nullable();
        $t->timestamp('billed_at')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_usage_alerts', function($t) {
        $t->increments('id');
        $t->string('alert_name', 100);
        $t->integer('user_id')->unsigned()->nullable();
        $t->string('metric_key', 50);
        $t->string('alert_type', 30);
        $t->decimal('threshold_value', 12, 2);
        $t->boolean('is_active')->default(true);
        $t->timestamp('last_triggered')->nullable();
    });

    return ['status' => 'success'];
}

function usage_tracking_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_usage_alerts');
    Capsule::schema()->dropIfExists('mod_usage_billing');
    Capsule::schema()->dropIfExists('mod_usage_records');
    Capsule::schema()->dropIfExists('mod_usage_metrics');
    return ['status' => 'success'];
}
```

## Usage Tracker

```php
<?php
class UsageTracker {
    public function registerMetric(array $metricData): int {
        $existing = Capsule::table('mod_usage_metrics')
            ->where('metric_key', $metricData['key'])
            ->first();

        if ($existing) {
            Capsule::table('mod_usage_metrics')
                ->where('metric_key', $metricData['key'])
                ->update([
                    'name' => $metricData['name'],
                    'unit' => $metricData['unit'],
                    'base_price' => $metricData['price'] ?? 0,
                    'tier_pricing' => isset($metricData['tiers']) ? json_encode($metricData['tiers']) : null,
                ]);

            return $existing->id;
        }

        return Capsule::table('mod_usage_metrics')->insertGetId([
            'metric_key' => $metricData['key'],
            'name' => $metricData['name'],
            'unit' => $metricData['unit'],
            'aggregation' => $metricData['aggregation'] ?? 'sum',
            'pricing_type' => $metricData['pricing_type'] ?? 'per_unit',
            'base_price' => $metricData['price'] ?? 0,
            'tier_pricing' => isset($metricData['tiers']) ? json_encode($metricData['tiers']) : null,
            'bill_in_arrears' => $metricData['bill_in_arrears'] ?? true,
        ]);
    }

    public function recordUsage(int $userId, int $serviceId, string $metricKey, float $quantity, ?string $date = null): array {
        $metric = Capsule::table('mod_usage_metrics')
            ->where('metric_key', $metricKey)
            ->where('is_active', 1)
            ->first();

        if (!$metric) {
            return ['success' => false, 'error' => 'Metric not found'];
        }

        $recordDate = $date ?? date('Y-m-d');
        $period = date('Y-m');

        // Check for existing record for same day
        $existing = Capsule::table('mod_usage_records')
            ->where('user_id', $userId)
            ->where('service_id', $serviceId)
            ->where('metric_key', $metricKey)
            ->where('record_date', $recordDate)
            ->first();

        if ($existing) {
            Capsule::table('mod_usage_records')
                ->where('id', $existing->id)
                ->update([
                    'quantity' => $metric->aggregation === 'sum'
                        ? $existing->quantity + $quantity
                        : $quantity,
                ]);

            $recordId = $existing->id;
        } else {
            $recordId = Capsule::table('mod_usage_records')->insertGetId([
                'user_id' => $userId,
                'service_id' => $serviceId,
                'metric_key' => $metricKey,
                'quantity' => $quantity,
                'period' => $period,
                'record_date' => $recordDate,
            ]);
        }

        // Check for alerts
        $this->checkAlerts($userId, $metricKey, $quantity);

        return [
            'success' => true,
            'record_id' => $recordId,
            'metric' => $metric->name,
        ];
    }

    public function getUsageSummary(int $userId, int $serviceId, string $period = 'current'): array {
        $dateRange = $this->getPeriodRange($period);
        $periodLabel = $dateRange['period'];

        $records = Capsule::table('mod_usage_records')
            ->where('user_id', $userId)
            ->where('service_id', $serviceId)
            ->where('record_date', '>=', $dateRange['start'])
            ->where('record_date', '<=', $dateRange['end'])
            ->get();

        $summary = [];
        foreach ($records as $record) {
            if (!isset($summary[$record->metric_key])) {
                $metric = Capsule::table('mod_usage_metrics')
                    ->where('metric_key', $record->metric_key)
                    ->first();

                $summary[$record->metric_key] = [
                    'metric' => $metric,
                    'total' => 0,
                    'records' => [],
                ];
            }

            $summary[$record->metric_key]['total'] += $record->quantity;
            $summary[$record->metric_key]['records'][] = $record;
        }

        return [
            'period' => $periodLabel,
            'start_date' => $dateRange['start'],
            'end_date' => $dateRange['end'],
            'usage' => $summary,
        ];
    }

    private function getPeriodRange(string $period): array {
        return match($period) {
            'current' => [
                'period' => date('Y-m'),
                'start' => date('Y-m-01'),
                'end' => date('Y-m-t'),
            ],
            'previous' => [
                'period' => date('Y-m', strtotime('-1 month')),
                'start' => date('Y-m-01', strtotime('-1 month')),
                'end' => date('Y-m-t', strtotime('-1 month')),
            ],
            'ytd' => [
                'period' => date('Y'),
                'start' => date('Y-01-01'),
                'end' => date('Y-m-d'),
            ],
            default => [
                'period' => date('Y-m'),
                'start' => date('Y-m-01'),
                'end' => date('Y-m-t'),
            ],
        };
    }

    public function calculateBilling(int $userId, int $serviceId, string $period): array {
        $dateRange = $this->getPeriodRange($period);
        $summary = $this->getUsageSummary($userId, $serviceId, $period);

        $lineItems = [];
        $totalAmount = 0;

        foreach ($summary['usage'] as $metricKey => $data) {
            $metric = $data['metric'];
            $total = $data['total'];

            $amount = $this->calculateMetricCharge($metric, $total);

            $lineItems[] = [
                'metric' => $metric->name,
                'quantity' => $total,
                'unit' => $metric->unit,
                'rate' => $metric->base_price,
                'amount' => $amount,
            ];

            $totalAmount += $amount;
        }

        // Check if billing record exists
        $existingBilling = Capsule::table('mod_usage_billing')
            ->where('user_id', $userId)
            ->where('service_id', $serviceId)
            ->where('period', $dateRange['period'])
            ->first();

        if ($existingBilling) {
            Capsule::table('mod_usage_billing')
                ->where('id', $existingBilling->id)
                ->update([
                    'total_amount' => $totalAmount,
                    'line_items' => json_encode($lineItems),
                ]);

            $billingId = $existingBilling->id;
        } else {
            $billingId = Capsule::table('mod_usage_billing')->insertGetId([
                'user_id' => $userId,
                'service_id' => $serviceId,
                'period' => $dateRange['period'],
                'billing_period_start' => $dateRange['start'],
                'billing_period_end' => $dateRange['end'],
                'total_amount' => $totalAmount,
                'line_items' => json_encode($lineItems),
            ]);
        }

        return [
            'billing_id' => $billingId,
            'period' => $dateRange['period'],
            'line_items' => $lineItems,
            'total_amount' => $totalAmount,
        ];
    }

    private function calculateMetricCharge(object $metric, float $quantity): float {
        $tiers = json_decode($metric->tier_pricing, true);

        if (empty($tiers)) {
            return $quantity * $metric->base_price;
        }

        // Calculate tiered pricing
        $totalCost = 0;
        $remainingQty = $quantity;
        $prevQty = 0;

        usort($tiers, fn($a, $b) => $a['min_qty'] - $b['min_qty']);

        foreach ($tiers as $tier) {
            $tierMin = $tier['min_qty'];
            $tierMax = $tier['max_qty'] ?? PHP_INT_MAX;
            $tierPrice = $tier['price'];

            if ($quantity > $tierMin) {
                $unitsInTier = min($remainingQty, $tierMax - $tierMin);
                $totalCost += $unitsInTier * $tierPrice;
                $remainingQty -= $unitsInTier;
            }
        }

        // Add remaining at last tier rate
        if ($remainingQty > 0 && !empty($tiers)) {
            $lastTier = end($tiers);
            $totalCost += $remainingQty * $lastTier['price'];
        }

        return $totalCost;
    }

    public function createInvoice(int $userId, int $serviceId, string $period): array {
        $billing = $this->calculateBilling($userId, $serviceId, $period);

        if ($billing['total_amount'] <= 0) {
            return ['success' => false, 'error' => 'No usage to bill'];
        }

        $lineItems = $billing['line_items'];

        $params = [
            'userid' => $userId,
            'date' => date('Y-m-d'),
            'duedate' => date('Y-m-d', strtotime('+7 days')),
            'itemdescription' => [],
            'itemamount' => [],
        ];

        foreach ($lineItems as $item) {
            $params['itemdescription'][] = "Usage: {$item['metric']} ({$item['quantity']} {$item['unit']})";
            $params['itemamount'][] = $item['amount'];
        }

        $result = localAPI('CreateInvoice', $params);

        if ($result['result'] === 'success') {
            Capsule::table('mod_usage_billing')
                ->where('id', $billing['billing_id'])
                ->update([
                    'status' => 'invoiced',
                    'invoice_id' => $result['invoiceid'],
                    'billed_at' => date('Y-m-d H:i:s'),
                ]);
        }

        return $result;
    }

    private function checkAlerts(int $userId, string $metricKey, float $currentUsage): void {
        $alerts = Capsule::table('mod_usage_alerts')
            ->where('metric_key', $metricKey)
            ->where('is_active', 1)
            ->where(function($q) use ($userId) {
                $q->whereNull('user_id')
                  ->orWhere('user_id', $userId);
            })
            ->get();

        foreach ($alerts as $alert) {
            $shouldTrigger = false;

            switch ($alert->alert_type) {
                case 'threshold':
                    $shouldTrigger = $currentUsage >= $alert->threshold_value;
                    break;
                case 'percentage':
                    $periodTotal = $this->getPeriodTotal($userId, $metricKey);
                    $limit = $alert->threshold_value;
                    if ($periodTotal >= $limit) {
                        $percentage = ($currentUsage / $limit) * 100;
                        $shouldTrigger = $percentage >= $alert->threshold_value;
                    }
                    break;
            }

            if ($shouldTrigger) {
                $this->triggerAlert($alert, $userId, $metricKey, $currentUsage);
            }
        }
    }

    private function triggerAlert(object $alert, int $userId, string $metricKey, float $usage): void {
        $user = Capsule::table('tblclients')->find($userId);
        $metric = Capsule::table('mod_usage_metrics')->where('metric_key', $metricKey)->first();

        sendTplEmail($user->email, 'usage_alert', [
            'alert_name' => $alert->alert_name,
            'metric_name' => $metric->name,
            'current_usage' => $usage,
            'threshold' => $alert->threshold_value,
        ]);

        Capsule::table('mod_usage_alerts')
            ->where('id', $alert->id)
            ->update(['last_triggered' => date('Y-m-d H:i:s')]);
    }

    public function getUserUsageDashboard(int $userId): array {
        $currentPeriod = $this->getUsageSummary($userId, 0, 'current');
        $previousPeriod = $this->getUsageSummary($userId, 0, 'previous');

        return [
            'current' => $currentPeriod,
            'previous' => $previousPeriod,
            'trends' => $this->calculateTrends($currentPeriod, $previousPeriod),
        ];
    }
}
```

## API Endpoint for Usage Updates

```php
<?php
add_hook('ApiGateWay', 1, function($vars) {
    if ($vars['action'] === 'record_usage') {
        $userId = $vars['user_id'];
        $serviceId = $vars['service_id'];
        $metricKey = $vars['metric_key'];
        $quantity = $vars['quantity'];

        $tracker = new UsageTracker();
        $result = $tracker->recordUsage($userId, $serviceId, $metricKey, $quantity);

        return $result;
    }

    if ($vars['action'] === 'get_usage') {
        $tracker = new UsageTracker();
        $result = $tracker->getUsageSummary($vars['user_id'], $vars['service_id'], $vars['period'] ?? 'current');

        return $result;
    }
});
```

---

**Related Skills:**
- whmcs-overage-billing
- whmcs-freemium-model
- whmcs-billing-dashboard