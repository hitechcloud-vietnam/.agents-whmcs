# WHMCS Pricing Engine Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build dynamic pricing modules with custom pricing rules and calculations.

## Pricing Module Structure

```php
<?php
/**
 * Pricing Module (Addon)
 * Location: modules/addons/{module}/
 */

function {module}_config(): array {
    return [
        'name' => 'Dynamic Pricing',
        'description' => 'Custom pricing rules and calculations',
        'version' => '1.0',
    ];
}

function {module}_activate(): array {
    Capsule::schema()->create('mod_pricing_rules', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('rule_type', 50);
        $t->text('conditions');
        $t->text('action');
        $t->integer('priority')->default(0);
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_pricing_logs', function($t) {
        $t->increments('id');
        $t->integer('invoice_id')->unsigned();
        $t->string('rule_applied', 100);
        $t->decimal('original_price', 10, 2);
        $t->decimal('new_price', 10, 2);
        $t->timestamps();
    });

    return ['status' => 'success', 'description' => 'Pricing module activated'];
}

function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_pricing_rules');
    Capsule::schema()->dropIfExists('mod_pricing_logs');
    return ['status' => 'success'];
}
```

## Pricing Rules Engine

```php
<?php
class PricingEngine {
    public function calculatePrice(array $product, array $context): array {
        $basePrice = $product['monthly'] ?? 0;
        $modifications = [];
        $totalAdjustment = 0;

        $rules = Capsule::table('mod_pricing_rules')
            ->where('is_active', 1)
            ->orderBy('priority', 'desc')
            ->get();

        foreach ($rules as $rule) {
            if ($this->evaluateConditions($rule->conditions, $context)) {
                $adjustment = $this->applyAction($rule->action, $basePrice, $context);
                $modifications[] = [
                    'rule' => $rule->name,
                    'adjustment' => $adjustment,
                ];
                $totalAdjustment += $adjustment;
            }
        }

        return [
            'base_price' => $basePrice,
            'final_price' => max(0, $basePrice + $totalAdjustment),
            'adjustments' => $modifications,
            'total_adjustment' => $totalAdjustment,
        ];
    }

    private function evaluateConditions(string $conditionsJson, array $context): bool {
        $conditions = json_decode($conditionsJson, true);

        foreach ($conditions as $condition) {
            if (!$this->checkCondition($condition, $context)) {
                return false;
            }
        }

        return true;
    }

    private function checkCondition(array $condition, array $context): bool {
        return match ($condition['type']) {
            'user_group' => in_array($context['user_group'] ?? '', $condition['values'] ?? []),
            'product_category' => in_array($context['category'] ?? '', $condition['values'] ?? []),
            'quantity_min' => ($context['quantity'] ?? 1) >= ($condition['value'] ?? 0),
            'time_range' => $this->isInTimeRange($condition),
            'user_first_order' => ($context['order_count'] ?? 0) === 0,
            'country' => ($context['country'] ?? '') === ($condition['value'] ?? ''),
            default => false,
        };
    }

    private function applyAction(array $action, float $basePrice, array $context): float {
        return match ($action['type']) {
            'percentage_discount' => -($basePrice * $action['value'] / 100),
            'percentage_markup' => $basePrice * $action['value'] / 100,
            'fixed_discount' => -($action['value'] ?? 0),
            'fixed_markup' => $action['value'] ?? 0,
            'tiered' => $this->calculateTieredPrice($action, $context),
            default => 0,
        };
    }
}
```

## Hook Integration

```php
add_hook('BeforePricingCalculation', 1, function($vars) {
    $engine = new PricingEngine();

    $product = $vars['product'];
    $context = [
        'user_id' => $_SESSION['uid'],
        'user_group' => getClientGroup($_SESSION['uid']),
        'category' => $product['gid'] ?? null,
        'country' => getClientCountry($_SESSION['uid']),
        'order_count' => getClientOrderCount($_SESSION['uid']),
        'quantity' => $vars['quantity'] ?? 1,
    ];

    $pricing = $engine->calculatePrice($product, $context);

    return ['price' => $pricing['final_price'], 'adjustments' => $pricing['adjustments']];
});
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-service-billing