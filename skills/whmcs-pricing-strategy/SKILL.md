# WHMCS Pricing Strategy Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Implement dynamic pricing strategies based on market conditions, demand, and customer segments.

## Database Schema

```php
<?php
// modules/addons/pricing_strategy/pricing_strategy.php

use WHMCS\Database\Capsule;

function pricing_strategy_config(): array {
    return [
        'name' => 'Pricing Strategy',
        'description' => 'Dynamic pricing engine with multiple strategies',
        'version' => '1.0',
    ];
}

function pricing_strategy_activate(): array {
    Capsule::schema()->create('mod_pricing_strategies', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('type', 50);
        $t->text('configuration');
        $t->integer('priority')->default(0);
        $t->string('conditions', 500)->nullable();
        $t->boolean('is_active')->default(true);
        $t->timestamp('starts_at')->nullable();
        $t->timestamp('ends_at')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_pricing_history', function($t) {
        $t->increments('id');
        $t->integer('product_id')->unsigned();
        $t->string('strategy_type', 50);
        $t->decimal('original_price', 10, 2);
        $t->decimal('final_price', 10, 2);
        $t->decimal('adjustment_amount', 10, 2)->default(0);
        $t->string('adjustment_type', 20);
        $t->string('customer_segment', 50)->nullable();
        $t->timestamp('applied_at')->useCurrent();
    });

    Capsule::schema()->create('mod_pricing_segments', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('description')->nullable();
        $t->text('rules');
        $t->string('discount_type', 20)->default('percentage');
        $t->decimal('discount_value', 10, 2)->default(0);
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    return ['status' => 'success'];
}

function pricing_strategy_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_pricing_segments');
    Capsule::schema()->dropIfExists('mod_pricing_history');
    Capsule::schema()->dropIfExists('mod_pricing_strategies');
    return ['status' => 'success'];
}
```

## Pricing Engine Class

```php
<?php
class PricingEngine {
    private array $strategies = [];

    public function __construct() {
        $this->loadStrategies();
    }

    private function loadStrategies(): void {
        $strategies = Capsule::table('mod_pricing_strategies')
            ->where('is_active', 1)
            ->orderBy('priority', 'desc')
            ->get();

        foreach ($strategies as $strategy) {
            $this->strategies[] = [
                'id' => $strategy->id,
                'name' => $strategy->name,
                'type' => $strategy->type,
                'config' => json_decode($strategy->configuration, true),
                'conditions' => json_decode($strategy->conditions, true),
                'starts_at' => $strategy->starts_at,
                'ends_at' => $strategy->ends_at,
            ];
        }
    }

    public function calculatePrice(int $productId, float $basePrice, array $context = []): array {
        $originalPrice = $basePrice;
        $adjustments = [];
        $finalPrice = $basePrice;

        foreach ($this->strategies as $strategy) {
            if (!$this->isStrategyActive($strategy)) {
                continue;
            }

            if (!$this->evaluateConditions($strategy['conditions'], $context)) {
                continue;
            }

            $result = $this->applyStrategy($strategy, $finalPrice, $context);

            if ($result['applied']) {
                $adjustments[] = [
                    'strategy' => $strategy['name'],
                    'type' => $strategy['type'],
                    'amount' => $result['adjustment'],
                ];

                $finalPrice = $result['price'];
            }
        }

        // Record pricing history
        $this->recordHistory($productId, $originalPrice, $finalPrice, $adjustments, $context);

        return [
            'original_price' => $originalPrice,
            'final_price' => $finalPrice,
            'total_adjustment' => $originalPrice - $finalPrice,
            'adjustments' => $adjustments,
        ];
    }

    private function isStrategyActive(array $strategy): bool {
        $now = date('Y-m-d H:i:s');

        if ($strategy['starts_at'] && $strategy['starts_at'] > $now) {
            return false;
        }

        if ($strategy['ends_at'] && $strategy['ends_at'] < $now) {
            return false;
        }

        return true;
    }

    private function evaluateConditions(?array $conditions, array $context): bool {
        if (empty($conditions)) {
            return true;
        }

        foreach ($conditions as $condition) {
            if (!$this->checkCondition($condition, $context)) {
                return false;
            }
        }

        return true;
    }

    private function checkCondition(array $condition, array $context): bool {
        $field = $condition['field'];
        $operator = $condition['operator'];
        $value = $condition['value'];

        $contextValue = $context[$field] ?? null;

        switch ($operator) {
            case 'equals':
                return $contextValue == $value;
            case 'not_equals':
                return $contextValue != $value;
            case 'greater_than':
                return $contextValue > $value;
            case 'less_than':
                return $contextValue < $value;
            case 'contains':
                return strpos($contextValue, $value) !== false;
            case 'in':
                return in_array($contextValue, (array)$value);
            default:
                return true;
        }
    }

    private function applyStrategy(array $strategy, float $currentPrice, array $context): array {
        $config = $strategy['config'];

        switch ($strategy['type']) {
            case 'percentage_discount':
                $adjustment = $currentPrice * ($config['percentage'] / 100);
                $newPrice = $currentPrice - $adjustment;
                break;

            case 'fixed_discount':
                $adjustment = min($config['amount'], $currentPrice);
                $newPrice = $currentPrice - $adjustment;
                break;

            case 'quantity_break':
                $quantity = $context['quantity'] ?? 1;
                $tiers = $config['tiers'] ?? [];

                foreach ($tiers as $tier) {
                    if ($quantity >= $tier['min_qty']) {
                        $adjustment = $currentPrice * ($tier['discount'] / 100);
                        $newPrice = $currentPrice - $adjustment;
                    }
                }
                break;

            case 'time_based':
                $adjustment = $currentPrice * ($config['discount_percentage'] / 100);
                $newPrice = $currentPrice - $adjustment;
                break;

            case 'segment_pricing':
                $segment = $context['segment'] ?? 'default';
                $segmentConfig = $config['segments'][$segment] ?? null;

                if ($segmentConfig) {
                    $adjustment = $currentPrice * ($segmentConfig['discount'] / 100);
                    $newPrice = $currentPrice - $adjustment;
                } else {
                    $newPrice = $currentPrice;
                    $adjustment = 0;
                }
                break;

            default:
                $newPrice = $currentPrice;
                $adjustment = 0;
        }

        // Apply max discount cap if specified
        if (isset($config['max_discount']) && $adjustment > $config['max_discount']) {
            $newPrice = $currentPrice - $config['max_discount'];
            $adjustment = $config['max_discount'];
        }

        return [
            'applied' => $adjustment > 0,
            'price' => max($newPrice, 0),
            'adjustment' => $adjustment,
        ];
    }

    private function recordHistory(int $productId, float $originalPrice, float $finalPrice, array $adjustments, array $context): void {
        $primaryStrategy = !empty($adjustments) ? $adjustments[0]['strategy'] : 'none';

        Capsule::table('mod_pricing_history')->insert([
            'product_id' => $productId,
            'strategy_type' => $primaryStrategy,
            'original_price' => $originalPrice,
            'final_price' => $finalPrice,
            'adjustment_amount' => $originalPrice - $finalPrice,
            'adjustment_type' => !empty($adjustments) ? $adjustments[0]['type'] : 'none',
            'customer_segment' => $context['segment'] ?? null,
        ]);
    }
}
```

## Cart Integration

```php
<?php
add_hook('CartProductCalculations', 1, function($vars) {
    $pricingEngine = new PricingEngine();

    $context = [
        'user_id' => $_SESSION['uid'] ?? null,
        'segment' => getCustomerSegment($_SESSION['uid'] ?? null),
        'quantity' => $vars['quantity'] ?? 1,
        'time_of_day' => date('H'),
        'day_of_week' => date('N'),
    ];

    $productId = $vars['product_id'];
    $basePrice = $vars['base_price'];

    $result = $pricingEngine->calculatePrice($productId, $basePrice, $context);

    return [
        'price' => $result['final_price'],
        'original_price' => $result['original_price'],
        'adjustments' => $result['adjustments'],
    ];
});
```

## Strategy Management

```php
<?php
class StrategyManager {
    public function createStrategy(string $name, string $type, array $config, array $conditions = []): array {
        $id = Capsule::table('mod_pricing_strategies')->insertGetId([
            'name' => $name,
            'type' => $type,
            'configuration' => json_encode($config),
            'conditions' => json_encode($conditions),
        ]);

        return ['success' => true, 'strategy_id' => $id];
    }

    public function updateStrategy(int $strategyId, array $data): array {
        $updateData = [];

        if (isset($data['name'])) $updateData['name'] = $data['name'];
        if (isset($data['configuration'])) $updateData['configuration'] = json_encode($data['configuration']);
        if (isset($data['conditions'])) $updateData['conditions'] = json_encode($data['conditions']);
        if (isset($data['is_active'])) $updateData['is_active'] = $data['is_active'];
        if (isset($data['starts_at'])) $updateData['starts_at'] = $data['starts_at'];
        if (isset($data['ends_at'])) $updateData['ends_at'] = $data['ends_at'];

        Capsule::table('mod_pricing_strategies')
            ->where('id', $strategyId)
            ->update($updateData);

        return ['success' => true];
    }

    public function getStrategies(): array {
        return Capsule::table('mod_pricing_strategies')
            ->orderBy('priority', 'desc')
            ->get();
    }

    public function deleteStrategy(int $strategyId): array {
        Capsule::table('mod_pricing_strategies')
            ->where('id', $strategyId)
            ->delete();

        return ['success' => true];
    }

    public function getPricingAnalytics(int $productId, string $period = '30days'): array {
        $startDate = date('Y-m-d', strtotime("-{$period}"));

        $history = Capsule::table('mod_pricing_history')
            ->where('product_id', $productId)
            ->where('applied_at', '>=', $startDate)
            ->orderBy('applied_at', 'desc')
            ->get();

        $totalOriginal = 0;
        $totalFinal = 0;
        $strategyUsage = [];

        foreach ($history as $record) {
            $totalOriginal += $record->original_price;
            $totalFinal += $record->final_price;

            if (!isset($strategyUsage[$record->strategy_type])) {
                $strategyUsage[$record->strategy_type] = 0;
            }
            $strategyUsage[$record->strategy_type]++;
        }

        return [
            'total_adjustments' => count($history),
            'total_savings' => $totalOriginal - $totalFinal,
            'avg_original_price' => count($history) > 0 ? $totalOriginal / count($history) : 0,
            'avg_final_price' => count($history) > 0 ? $totalFinal / count($history) : 0,
            'strategy_usage' => $strategyUsage,
        ];
    }
}
```

## Admin Interface

```php
function pricing_strategy_output(array $vars): void {
    $action = $_GET['action'] ?? 'list';

    $manager = new StrategyManager();

    echo '<div class="pricing-strategy-admin">';

    if ($action === 'list') {
        $strategies = $manager->getStrategies();

        echo '<h2>Pricing Strategies</h2>';
        echo '<a href="?action=add" class="btn btn-primary">Add Strategy</a>';
        echo '<table class="datatable">';
        echo '<thead><tr><th>Name</th><th>Type</th><th>Status</th><th>Actions</th></tr></thead>';
        echo '<tbody>';

        foreach ($strategies as $s) {
            echo '<tr>';
            echo "<td>{$s->name}</td>";
            echo "<td>{$s->type}</td>";
            echo "<td>" . ($s->is_active ? 'Active' : 'Inactive') . "</td>";
            echo '<td><a href="?action=edit&id=' . $s->id . '">Edit</a></td>';
            echo '</tr>';
        }

        echo '</tbody></table>';
    }

    echo '</div>';
}
```

---

**Related Skills:**
- whmcs-pricing-engine
- whmcs-tiered-pricing
- whmcs-volume-discounts