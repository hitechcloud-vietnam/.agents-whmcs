# WHMCS Volume Discounts Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Implement volume discount systems for bulk purchases.

## Database Schema

```php
<?php
// modules/addons/volume_discounts/volume_discounts.php

use WHMCS\Database\Capsule;

function volume_discounts_config(): array {
    return [
        'name' => 'Volume Discounts',
        'description' => 'Bulk purchase discount system',
        'version' => '1.0',
    ];
}

function volume_discounts_activate(): array {
    Capsule::schema()->create('mod_volume_discount_rules', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('discount_type', 20)->default('percentage');
        $t->string('scope', 50)->default('product');
        $t->integer('scope_id')->unsigned()->nullable();
        $t->text('conditions')->nullable();
        $t->boolean('is_active')->default(true);
        $t->timestamp('starts_at')->nullable();
        $t->timestamp('ends_at')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_volume_discount_tiers', function($t) {
        $t->increments('id');
        $t->integer('rule_id')->unsigned();
        $t->integer('min_quantity')->unsigned();
        $t->decimal('discount', 10, 2);
        $t->integer('sort_order')->default(0);
    });

    Capsule::schema()->create('mod_volume_discount_usage', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->integer('product_id')->unsigned();
        $t->integer('rule_id')->unsigned();
        $t->integer('quantity')->unsigned();
        $t->decimal('discount_amount', 10, 2);
        $t->timestamp('applied_at')->useCurrent();
    });

    return ['status' => 'success'];
}

function volume_discounts_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_volume_discount_usage');
    Capsule::schema()->dropIfExists('mod_volume_discount_tiers');
    Capsule::schema()->dropIfExists('mod_volume_discount_rules');
    return ['status' => 'success'];
}
```

## Volume Discount Manager

```php
<?php
class VolumeDiscountManager {
    public function getApplicableDiscount(int $productId, int $quantity, int $userId = null): ?array {
        $rules = $this->getActiveRules($productId, $userId);

        foreach ($rules as $rule) {
            $tier = $this->getMatchingTier($rule->id, $quantity);

            if ($tier) {
                $basePrice = $this->getProductPrice($productId);
                $discountAmount = $this->calculateDiscount($basePrice, $tier->discount, $rule->discount_type);

                return [
                    'rule_id' => $rule->id,
                    'rule_name' => $rule->name,
                    'tier_id' => $tier->id,
                    'min_quantity' => $tier->min_quantity,
                    'discount_type' => $rule->discount_type,
                    'discount_value' => $tier->discount,
                    'discount_amount' => $discountAmount,
                    'unit_price' => $basePrice - $discountAmount,
                    'total_price' => ($basePrice - $discountAmount) * $quantity,
                ];
            }
        }

        return null;
    }

    private function getActiveRules(int $productId, ?int $userId): array {
        $now = date('Y-m-d H:i:s');

        return Capsule::table('mod_volume_discount_rules')
            ->where('is_active', 1)
            ->where(function($q) use ($now) {
                $q->whereNull('starts_at')
                  ->orWhere('starts_at', '<=', $now);
            })
            ->where(function($q) use ($now) {
                $q->whereNull('ends_at')
                  ->orWhere('ends_at', '>=', $now);
            })
            ->where(function($q) use ($productId) {
                $q->where('scope', 'global')
                  ->orWhere(function($q2) use ($productId) {
                      $q2->where('scope', 'product')
                         ->where('scope_id', $productId);
                  });
            })
            ->get();
    }

    private function getMatchingTier(int $ruleId, int $quantity): ?object {
        return Capsule::table('mod_volume_discount_tiers')
            ->where('rule_id', $ruleId)
            ->where('min_quantity', '<=', $quantity)
            ->orderBy('min_quantity', 'desc')
            ->first();
    }

    private function calculateDiscount(float $basePrice, float $value, string $type): float {
        if ($type === 'percentage') {
            return $basePrice * ($value / 100);
        }

        return min($value, $basePrice);
    }

    private function getProductPrice(int $productId): float {
        $pricing = Capsule::table('tblpricing')
            ->where('type', 'product')
            ->where('relid', $productId)
            ->first();

        return $pricing->monthly ?? 0;
    }

    public function createRule(string $name, string $scope, ?int $scopeId, array $tiers): int {
        $ruleId = Capsule::table('mod_volume_discount_rules')->insertGetId([
            'name' => $name,
            'scope' => $scope,
            'scope_id' => $scopeId,
            'discount_type' => 'percentage',
        ]);

        foreach ($tiers as $index => $tier) {
            Capsule::table('mod_volume_discount_tiers')->insert([
                'rule_id' => $ruleId,
                'min_quantity' => $tier['min_quantity'],
                'discount' => $tier['discount'],
                'sort_order' => $index,
            ]);
        }

        return $ruleId;
    }

    public function recordUsage(int $userId, int $productId, int $ruleId, int $quantity, float $discount): void {
        Capsule::table('mod_volume_discount_usage')->insert([
            'user_id' => $userId,
            'product_id' => $productId,
            'rule_id' => $ruleId,
            'quantity' => $quantity,
            'discount_amount' => $discount,
        ]);
    }

    public function getDiscountAnalytics(int $ruleId = null): array {
        $query = Capsule::table('mod_volume_discount_usage')
            ->selectRaw('rule_id, COUNT(*) as uses, SUM(quantity) as total_qty, SUM(discount_amount) as total_savings')
            ->groupBy('rule_id');

        if ($ruleId) {
            $query->where('rule_id', $ruleId);
        }

        return $query->get();
    }
}
```

## Cart Integration

```php
<?php
add_hook('CartProductCalculations', 1, function($vars) {
    $userId = $_SESSION['uid'] ?? null;
    $quantity = $vars['quantity'] ?? 1;
    $productId = $vars['product_id'];

    $manager = new VolumeDiscountManager();
    $discount = $manager->getApplicableDiscount($productId, $quantity, $userId);

    if ($discount) {
        return [
            'price' => $discount['unit_price'],
            'discount_amount' => $discount['discount_amount'],
            'discount_label' => "Buy {$discount['min_quantity']}+ and save {$discount['discount_value']}%",
        ];
    }

    return null;
});

add_hook('OrderFormProductView', 1, function($vars) {
    $productId = $vars['product_id'];
    $manager = new VolumeDiscountManager();

    // Get all tiers for display
    $tiers = Capsule::table('mod_volume_discount_tiers')
        ->join('mod_volume_discount_rules', 'mod_volume_discount_tiers.rule_id', '=', 'mod_volume_discount_rules.id')
        ->where('mod_volume_discount_rules.scope', 'product')
        ->where('mod_volume_discount_rules.scope_id', $productId)
        ->where('mod_volume_discount_rules.is_active', 1)
        ->orderBy('min_quantity')
        ->get();

    if (!empty($tiers)) {
        return [
            'volume_discount_tiers' => $tiers,
        ];
    }
});
```

## Admin Configuration

```php
<?php
function volume_discounts_output(array $vars): void {
    $action = $_GET['action'] ?? 'list';

    $manager = new VolumeDiscountManager();

    if ($action === 'create') {
        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            check_token('WHMCS.admin.default');

            $tiers = [];
            foreach ($_POST['min_qty'] as $i => $minQty) {
                $tiers[] = [
                    'min_quantity' => (int)$minQty,
                    'discount' => (float)$_POST['discount'][$i],
                ];
            }

            $manager->createRule(
                $_POST['name'],
                $_POST['scope'],
                !empty($_POST['product_id']) ? (int)$_POST['product_id'] : null,
                $tiers
            );

            redir('success=1');
        }

        $products = Capsule::table('tblproducts')->get(['id', 'name']);

        echo '<h2>Create Volume Discount Rule</h2>';
        echo '<form method="post">';
        echo '<input type="hidden" name="_token" value="' . generate_token() . '">';

        echo '<div class="form-group">';
        echo '<label>Name</label>';
        echo '<input type="text" name="name" required class="form-control">';
        echo '</div>';

        echo '<div class="form-group">';
        echo '<label>Scope</label>';
        echo '<select name="scope" id="scope-select" class="form-control">';
        echo '<option value="global">Global (All Products)</option>';
        echo '<option value="product">Specific Product</option>';
        echo '</select>';
        echo '</div>';

        echo '<div class="form-group" id="product-select" style="display:none;">';
        echo '<label>Product</label>';
        echo '<select name="product_id" class="form-control">';
        foreach ($products as $p) {
            echo '<option value="' . $p->id . '">' . htmlspecialchars($p->name) . '</option>';
        }
        echo '</select>';
        echo '</div>';

        echo '<h3>Tiers</h3>';
        echo '<div id="tiers-container">';
        echo '<div class="tier-row"><input type="number" name="min_qty[]" placeholder="Min Qty"><input type="number" step="0.01" name="discount[]" placeholder="Discount %"></div>';
        echo '</div>';
        echo '<button type="button" id="add-tier">Add Tier</button>';

        echo '<button type="submit" class="btn btn-primary">Create Rule</button>';
        echo '</form>';
    } else {
        $rules = Capsule::table('mod_volume_discount_rules')
            ->orderBy('created_at', 'desc')
            ->get();

        $analytics = $manager->getDiscountAnalytics();

        echo '<h2>Volume Discount Rules</h2>';
        echo '<a href="?action=create" class="btn btn-primary">Create Rule</a>';
        echo '<table class="datatable">';
        echo '<thead><tr><th>Name</th><th>Scope</th><th>Tiers</th><th>Status</th><th>Usage</th></tr></thead><tbody>';

        foreach ($rules as $rule) {
            $tierCount = Capsule::table('mod_volume_discount_tiers')
                ->where('rule_id', $rule->id)
                ->count();

            $usage = $analytics[$rule->id] ?? (object)['uses' => 0, 'total_savings' => 0];

            echo '<tr>';
            echo "<td>{$rule->name}</td>";
            echo "<td>{$rule->scope}</td>";
            echo "<td>{$tierCount}</td>";
            echo "<td>" . ($rule->is_active ? 'Active' : 'Inactive') . "</td>";
            echo "<td>{$usage->uses} uses, $" . number_format($usage->total_savings, 2) . " saved</td>";
            echo '</tr>';
        }

        echo '</tbody></table>';
    }
}
```

## Widget for Product Page

```php
<?php
function getVolumeDiscountWidget(int $productId): string {
    $tiers = Capsule::table('mod_volume_discount_tiers')
        ->join('mod_volume_discount_rules', 'mod_volume_discount_tiers.rule_id', '=', 'mod_volume_discount_rules.id')
        ->where('mod_volume_discount_rules.scope', 'product')
        ->where('mod_volume_discount_rules.scope_id', $productId)
        ->where('mod_volume_discount_rules.is_active', 1)
        ->orderBy('min_quantity')
        ->get();

    if (empty($tiers)) {
        return '';
    }

    ob_start();
    ?>
    <div class="volume-discount-widget">
        <h4>Volume Discounts</h4>
        <table>
            <thead>
                <tr><th>Quantity</th><th>Discount</th></tr>
            </thead>
            <tbody>
                <?php foreach ($tiers as $tier): ?>
                <tr>
                    <td><?php echo $tier->min_quantity; ?>+</td>
                    <td><?php echo number_format($tier->discount, 0); ?>% off</td>
                </tr>
                <?php endforeach; ?>
            </tbody>
        </table>
    </div>
    <?php
    return ob_get_clean();
}
```

---

**Related Skills:**
- whmcs-pricing-strategy
- whmcs-tiered-pricing
- whmcs-promotional-codes