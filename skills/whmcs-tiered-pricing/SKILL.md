# WHMCS Tiered Pricing Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Implement tiered pricing where customers get lower prices for buying more.

## Database Schema

```php
<?php
// modules/addons/tiered_pricing/tiered_pricing.php

use WHMCS\Database\Capsule;

function tiered_pricing_config(): array {
    return [
        'name' => 'Tiered Pricing',
        'description' => 'Volume-based pricing tiers',
        'version' => '1.0',
    ];
}

function tiered_pricing_activate(): array {
    Capsule::schema()->create('mod_tiered_pricing_tiers', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->text('description')->nullable();
        $t->integer('product_id')->unsigned();
        $t->integer('min_quantity')->unsigned()->default(1);
        $t->integer('max_quantity')->unsigned()->nullable();
        $t->decimal('unit_price', 10, 2);
        $t->string('discount_type', 20)->default('fixed');
        $t->decimal('discount_value', 10, 2)->default(0);
        $t->integer('sort_order')->default(0);
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_tiered_pricing_customer_tiers', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->integer('product_id')->unsigned();
        $t->integer('current_tier_id')->unsigned();
        $t->integer('total_quantity')->default(0);
        $t->decimal('lifetime_value', 12, 2)->default(0);
        $t->timestamp('tier_achieved_at')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_tiered_pricing_history', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->integer('product_id')->unsigned();
        $t->integer('tier_id')->unsigned();
        $t->integer('quantity')->unsigned();
        $t->decimal('unit_price', 10, 2);
        $t->decimal('total_price', 10, 2);
        $t->timestamp('applied_at')->useCurrent();
    });

    return ['status' => 'success'];
}

function tiered_pricing_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_tiered_pricing_history');
    Capsule::schema()->dropIfExists('mod_tiered_pricing_customer_tiers');
    Capsule::schema()->dropIfExists('mod_tiered_pricing_tiers');
    return ['status' => 'success'];
}
```

## Tiered Pricing Manager

```php
<?php
class TieredPricingManager {
    public function getProductTiers(int $productId): array {
        return Capsule::table('mod_tiered_pricing_tiers')
            ->where('product_id', $productId)
            ->where('is_active', 1)
            ->orderBy('min_quantity')
            ->get();
    }

    public function calculateTieredPrice(int $productId, int $quantity, int $userId = null): array {
        $tiers = $this->getProductTiers($productId);

        if (empty($tiers)) {
            return ['success' => false, 'error' => 'No tiers configured'];
        }

        // Find applicable tier
        $applicableTier = null;
        foreach ($tiers as $tier) {
            if ($quantity >= $tier->min_quantity) {
                if ($tier->max_quantity === null || $quantity <= $tier->max_quantity) {
                    $applicableTier = $tier;
                }
            }
        }

        if (!$applicableTier) {
            // Use highest tier if quantity exceeds all tiers
            $applicableTier = end($tiers);
        }

        $unitPrice = $this->calculateUnitPrice($applicableTier);
        $totalPrice = $unitPrice * $quantity;

        return [
            'success' => true,
            'tier_id' => $applicableTier->id,
            'tier_name' => $applicableTier->name,
            'unit_price' => $unitPrice,
            'total_price' => $totalPrice,
            'quantity' => $quantity,
            'savings' => $this->calculateSavings($productId, $quantity, $unitPrice),
        ];
    }

    private function calculateUnitPrice(object $tier): float {
        if ($tier->discount_type === 'percentage') {
            $basePrice = Capsule::table('tblpricing')
                ->where('type', 'product')
                ->where('relid', $tier->product_id)
                ->first();

            $basePrice = $basePrice->monthly ?? 0;
            return $basePrice * (1 - $tier->discount_value / 100);
        }

        return $tier->unit_price;
    }

    private function calculateSavings(int $productId, int $quantity, float $tieredPrice): float {
        $basePrice = Capsule::table('tblpricing')
            ->where('type', 'product')
            ->where('relid', $productId)
            ->first();

        $basePrice = $basePrice->monthly ?? 0;
        $regularTotal = $basePrice * $quantity;
        $tieredTotal = $tieredPrice * $quantity;

        return max(0, $regularTotal - $tieredTotal);
    }

    public function recordPurchase(int $userId, int $productId, int $quantity, float $total): void {
        // Update customer tier progress
        $customerTier = Capsule::table('mod_tiered_pricing_customer_tiers')
            ->where('user_id', $userId)
            ->where('product_id', $productId)
            ->first();

        if ($customerTier) {
            Capsule::table('mod_tiered_pricing_customer_tiers')
                ->where('id', $customerTier->id)
                ->update([
                    'total_quantity' => $customerTier->total_quantity + $quantity,
                    'lifetime_value' => $customerTier->lifetime_value + $total,
                ]);

            // Check for tier upgrade
            $this->checkTierUpgrade($userId, $productId);
        } else {
            $firstTier = Capsule::table('mod_tiered_pricing_tiers')
                ->where('product_id', $productId)
                ->orderBy('min_quantity')
                ->first();

            Capsule::table('mod_tiered_pricing_customer_tiers')->insert([
                'user_id' => $userId,
                'product_id' => $productId,
                'current_tier_id' => $firstTier->id ?? 0,
                'total_quantity' => $quantity,
                'lifetime_value' => $total,
            ]);
        }

        // Record history
        $result = $this->calculateTieredPrice($productId, $quantity, $userId);

        Capsule::table('mod_tiered_pricing_history')->insert([
            'user_id' => $userId,
            'product_id' => $productId,
            'tier_id' => $result['tier_id'],
            'quantity' => $quantity,
            'unit_price' => $result['unit_price'],
            'total_price' => $total,
        ]);
    }

    private function checkTierUpgrade(int $userId, int $productId): void {
        $customerTier = Capsule::table('mod_tiered_pricing_customer_tiers')
            ->where('user_id', $userId)
            ->where('product_id', $productId)
            ->first();

        $totalQty = $customerTier->total_quantity;

        $nextTier = Capsule::table('mod_tiered_pricing_tiers')
            ->where('product_id', $productId)
            ->where('min_quantity', '>', $customerTier->current_tier_id ? Capsule::raw("(SELECT min_quantity FROM mod_tiered_pricing_tiers WHERE id = {$customerTier->current_tier_id})") : 0)
            ->where('min_quantity', '<=', $totalQty)
            ->first();

        if ($nextTier) {
            Capsule::table('mod_tiered_pricing_customer_tiers')
                ->where('id', $customerTier->id)
                ->update([
                    'current_tier_id' => $nextTier->id,
                    'tier_achieved_at' => date('Y-m-d H:i:s'),
                ]);

            // Notify customer
            $user = Capsule::table('tblclients')->find($userId);
            sendTplEmail($user->email, 'tier_upgrade_notification', [
                'tier_name' => $nextTier->name,
                'product' => $productId,
            ]);
        }
    }
}
```

## Product Configuration

```php
<?php
function tiered_pricing_output(array $vars): void {
    $action = $_GET['action'] ?? 'list';
    $productId = (int)($_GET['product_id'] ?? 0);

    $manager = new TieredPricingManager();

    if ($action === 'configure' && $productId) {
        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            check_token('WHMCS.admin.default');

            $tiers = $_POST['tier'];

            Capsule::table('mod_tiered_pricing_tiers')
                ->where('product_id', $productId)
                ->delete();

            foreach ($tiers as $index => $tier) {
                if (empty($tier['min_qty'])) continue;

                Capsule::table('mod_tiered_pricing_tiers')->insert([
                    'name' => $tier['name'] ?? "Tier {$index}",
                    'product_id' => $productId,
                    'min_quantity' => (int)$tier['min_qty'],
                    'max_quantity' => !empty($tier['max_qty']) ? (int)$tier['max_qty'] : null,
                    'discount_type' => $tier['discount_type'] ?? 'percentage',
                    'discount_value' => (float)$tier['discount_value'],
                    'sort_order' => $index,
                ]);
            }

            echo '<div class="alert alert-success">Tiers saved successfully</div>';
        }

        $product = Capsule::table('tblproducts')->find($productId);
        $tiers = $manager->getProductTiers($productId);

        echo '<h2>Configure Tiered Pricing: ' . $product->name . '</h2>';
        echo '<form method="post">';
        echo '<input type="hidden" name="_token" value="' . generate_token() . '">';

        echo '<table id="tier-table" class="datatable">';
        echo '<thead><tr><th>Name</th><th>Min Qty</th><th>Max Qty</th><th>Discount Type</th><th>Value</th><th></th></tr></thead>';
        echo '<tbody>';

        foreach ($tiers as $index => $tier) {
            echo '<tr>';
            echo '<td><input type="text" name="tier[' . $index . '][name]" value="' . htmlspecialchars($tier->name) . '"></td>';
            echo '<td><input type="number" name="tier[' . $index . '][min_qty]" value="' . $tier->min_quantity . '"></td>';
            echo '<td><input type="number" name="tier[' . $index . '][max_qty]" value="' . $tier->max_quantity . '"></td>';
            echo '<td><select name="tier[' . $index . '][discount_type]">';
            echo '<option value="percentage"' . ($tier->discount_type === 'percentage' ? ' selected' : '') . '>Percentage</option>';
            echo '<option value="fixed"' . ($tier->discount_type === 'fixed' ? ' selected' : '') . '>Fixed Price</option>';
            echo '</select></td>';
            echo '<td><input type="number" step="0.01" name="tier[' . $index . '][discount_value]" value="' . $tier->discount_value . '"></td>';
            echo '<td><button type="button" class="remove-tier">Remove</button></td>';
            echo '</tr>';
        }

        echo '</tbody>';
        echo '</table>';

        echo '<button type="button" id="add-tier" class="btn">Add Tier</button>';
        echo '<button type="submit" class="btn btn-primary">Save Tiers</button>';
        echo '</form>';
    } else {
        // List products with tiers
        $products = Capsule::table('tblproducts')
            ->leftJoin('mod_tiered_pricing_tiers', 'tblproducts.id', '=', 'mod_tiered_pricing_tiers.product_id')
            ->select('tblproducts.*', Capsule::raw('COUNT(mod_tiered_pricing_tiers.id) as tier_count'))
            ->groupBy('tblproducts.id')
            ->get();

        echo '<h2>Products with Tiered Pricing</h2>';
        echo '<table class="datatable">';
        echo '<thead><tr><th>Product</th><th>Tiers</th><th>Actions</th></tr></thead>';
        echo '<tbody>';

        foreach ($products as $p) {
            echo '<tr>';
            echo "<td>{$p->name}</td>";
            echo "<td>{$p->tier_count}</td>";
            echo '<td><a href="?action=configure&product_id=' . $p->id . '">Configure</a></td>';
            echo '</tr>';
        }

        echo '</tbody></table>';
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

    $manager = new TieredPricingManager();
    $result = $manager->calculateTieredPrice($productId, $quantity, $userId);

    if ($result['success']) {
        return [
            'price' => $result['unit_price'],
            'tier' => $result['tier_name'],
            'savings' => $result['savings'],
        ];
    }

    return null;
});
```

---

**Related Skills:**
- whmcs-pricing-strategy
- whmcs-volume-discounts
- whmcs-pricing-engine