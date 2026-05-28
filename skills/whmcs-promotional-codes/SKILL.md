# WHMCS Promotional Codes Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Advanced promotional code features including BOGO, limited use, and conditional rules.

## Database Schema

```php
<?php
// modules/addons/promo_codes/promo_codes.php

use WHMCS\Database\Capsule;

function promo_codes_config(): array {
    return [
        'name' => 'Promotional Codes',
        'description' => 'Advanced promo code system',
        'version' => '1.0',
    ];
}

function promo_codes_activate(): array {
    Capsule::schema()->create('mod_promo_codes', function($t) {
        $t->increments('id');
        $t->string('code', 50)->unique();
        $t->string('type', 30);
        $t->decimal('value', 10, 2);
        $t->string('discount_type', 20)->default('percentage');
        $t->decimal('max_discount', 10, 2)->nullable();
        $t->integer('max_uses')->unsigned()->nullable();
        $t->integer('max_uses_per_user')->unsigned()->default(1);
        $t->integer('uses_count')->unsigned()->default(0);
        $t->decimal('min_order_value', 10, 2)->nullable();
        $t->decimal('min_item_quantity', 10)->nullable();
        $t->string('applies_to', 50)->default('all');
        $t->text('product_ids')->nullable();
        $t->text('category_ids')->nullable();
        $t->integer('user_id')->unsigned()->nullable();
        $t->integer('client_group_id')->unsigned()->nullable();
        $t->date('valid_from')->nullable();
        $t->date('valid_until')->nullable();
        $t->string('bogo_second_item', 30)->nullable();
        $t->text('conditions')->nullable();
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_promo_redemptions', function($t) {
        $t->increments('id');
        $t->integer('promo_id')->unsigned();
        $t->integer('user_id')->unsigned();
        $t->integer('order_id')->unsigned()->nullable();
        $t->decimal('discount_amount', 10, 2);
        $t->timestamp('redeemed_at')->useCurrent();
    });

    Capsule::schema()->create('mod_promo_bogo_pairs', function($t) {
        $t->increments('id');
        $t->integer('promo_id')->unsigned();
        $t->integer('buy_product_id')->unsigned();
        $t->integer('get_product_id')->unsigned();
        $t->decimal('get_discount', 10, 2)->default(100);
        $t->boolean('same_product')->default(false);
    });

    return ['status' => 'success'];
}

function promo_codes_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_promo_bogo_pairs');
    Capsule::schema()->dropIfExists('mod_promo_redemptions');
    Capsule::schema()->dropIfExists('mod_promo_codes');
    return ['status' => 'success'];
}
```

## Promotional Code Manager

```php
<?php
class PromoCodeManager {
    public function validate(string $code, int $userId, array $cart = []): array {
        $promo = Capsule::table('mod_promo_codes')
            ->where('code', strtoupper($code))
            ->where('is_active', 1)
            ->first();

        if (!$promo) {
            return ['valid' => false, 'error' => 'Invalid promo code'];
        }

        // Check user-specific code
        if ($promo->user_id && $promo->user_id !== $userId) {
            return ['valid' => false, 'error' => 'This code is not for your account'];
        }

        // Check client group
        if ($promo->client_group_id) {
            $user = Capsule::table('tblclients')->find($userId);
            if ($user->client_group_id !== $promo->client_group_id) {
                return ['valid' => false, 'error' => 'This code is not available for your account type'];
            }
        }

        // Check validity dates
        if ($promo->valid_from && $promo->valid_from > date('Y-m-d')) {
            return ['valid' => false, 'error' => 'This code is not yet active'];
        }

        if ($promo->valid_until && $promo->valid_until < date('Y-m-d')) {
            return ['valid' => false, 'error' => 'This code has expired'];
        }

        // Check total uses
        if ($promo->max_uses && $promo->uses_count >= $promo->max_uses) {
            return ['valid' => false, 'error' => 'This code has reached its usage limit'];
        }

        // Check per-user uses
        $userRedemptions = Capsule::table('mod_promo_redemptions')
            ->where('promo_id', $promo->id)
            ->where('user_id', $userId)
            ->count();

        if ($userRedemptions >= $promo->max_uses_per_user) {
            return ['valid' => false, 'error' => 'You have already used this code'];
        }

        // Check minimum order value
        if ($promo->min_order_value && $cart['total'] < $promo->min_order_value) {
            return ['valid' => false, 'error' => "Minimum order value: $" . number_format($promo->min_order_value, 2)];
        }

        // Check conditions
        $conditions = json_decode($promo->conditions, true);
        if ($conditions && !$this->evaluateConditions($conditions, $cart)) {
            return ['valid' => false, 'error' => 'Conditions not met for this code'];
        }

        return ['valid' => true, 'promo' => $promo];
    }

    public function calculateDiscount(object $promo, array $cart): float {
        $items = $cart['items'] ?? [];

        switch ($promo->type) {
            case 'percentage':
                $eligibleTotal = $this->getEligibleTotal($promo, $items);
                $discount = $eligibleTotal * ($promo->value / 100);
                break;

            case 'fixed':
                $discount = $promo->value;
                break;

            case 'bogo':
                $discount = $this->calculateBOGODiscount($promo, $items);
                break;

            case 'free_shipping':
                $discount = $cart['shipping'] ?? 0;
                break;

            default:
                $discount = 0;
        }

        // Apply max discount cap
        if ($promo->max_discount && $discount > $promo->max_discount) {
            $discount = $promo->max_discount;
        }

        return min($discount, $cart['total']);
    }

    private function getEligibleTotal(object $promo, array $items): float {
        if ($promo->applies_to === 'all') {
            return array_sum(array_column($items, 'total'));
        }

        $eligibleProductIds = json_decode($promo->product_ids, true) ?: [];
        $eligibleCategoryIds = json_decode($promo->category_ids, true) ?: [];

        $eligibleTotal = 0;
        foreach ($items as $item) {
            if (in_array($item['product_id'], $eligibleProductIds)) {
                $eligibleTotal += $item['total'];
            }
        }

        return $eligibleTotal;
    }

    private function calculateBOGODiscount(object $promo, array $items): float {
        $bogoPairs = Capsule::table('mod_promo_bogo_pairs')
            ->where('promo_id', $promo->id)
            ->get();

        $discount = 0;
        $buyCounts = [];

        foreach ($items as $item) {
            if (!isset($buyCounts[$item['product_id']])) {
                $buyCounts[$item['product_id']] = 0;
            }
            $buyCounts[$item['product_id']] += $item['quantity'];
        }

        foreach ($bogoPairs as $pair) {
            if ($pair->same_product) {
                $buyQty = $buyCounts[$pair->buy_product_id] ?? 0;
                $freeItems = floor($buyQty / 2);

                $itemPrice = $this->getProductPrice($pair->get_product_id);
                $discount += $itemPrice * $freeItems * ($pair->get_discount / 100);
            } else {
                $buyQty = $buyCounts[$pair->buy_product_id] ?? 0;
                $getQty = min($buyQty, $buyCounts[$pair->get_product_id] ?? 0);

                $itemPrice = $this->getProductPrice($pair->get_product_id);
                $discount += $itemPrice * $getQty * ($pair->get_discount / 100);
            }
        }

        return $discount;
    }

    private function getProductPrice(int $productId): float {
        $pricing = Capsule::table('tblpricing')
            ->where('type', 'product')
            ->where('relid', $productId)
            ->first();

        return $pricing->monthly ?? 0;
    }

    private function evaluateConditions(array $conditions, array $cart): bool {
        foreach ($conditions as $condition) {
            switch ($condition['type']) {
                case 'product_in_cart':
                    $found = false;
                    foreach ($cart['items'] ?? [] as $item) {
                        if ($item['product_id'] == $condition['product_id']) {
                            $found = true;
                            break;
                        }
                    }
                    if (!$found) return false;
                    break;

                case 'category_in_cart':
                    $found = false;
                    foreach ($cart['items'] ?? [] as $item) {
                        if (in_array($condition['category_id'], $item['categories'] ?? [])) {
                            $found = true;
                            break;
                        }
                    }
                    if (!$found) return false;
                    break;

                case 'min_cart_value':
                    if (($cart['total'] ?? 0) < $condition['value']) return false;
                    break;

                case 'new_customer_only':
                    $userOrderCount = Capsule::table('tblorders')
                        ->where('userid', $_SESSION['uid'] ?? 0)
                        ->count();
                    if ($userOrderCount > 0) return false;
                    break;
            }
        }

        return true;
    }

    public function redeem(int $promoId, int $userId, ?int $orderId, float $discountAmount): void {
        Capsule::table('mod_promo_redemptions')->insert([
            'promo_id' => $promoId,
            'user_id' => $userId,
            'order_id' => $orderId,
            'discount_amount' => $discountAmount,
        ]);

        Capsule::table('mod_promo_codes')
            ->where('id', $promoId)
            ->increment('uses_count');
    }
}
```

## Admin Interface

```php
<?php
function promo_codes_output(array $vars): void {
    $action = $_GET['action'] ?? 'list';

    if ($action === 'create') {
        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            check_token('WHMCS.admin.default');

            $code = strtoupper($_POST['code']);

            Capsule::table('mod_promo_codes')->insert([
                'code' => $code,
                'type' => $_POST['type'],
                'value' => $_POST['value'],
                'discount_type' => $_POST['discount_type'],
                'max_discount' => $_POST['max_discount'] ?: null,
                'max_uses' => $_POST['max_uses'] ?: null,
                'max_uses_per_user' => $_POST['max_uses_per_user'] ?: 1,
                'min_order_value' => $_POST['min_order_value'] ?: null,
                'applies_to' => $_POST['applies_to'],
                'product_ids' => json_encode($_POST['product_ids'] ?? []),
                'valid_from' => $_POST['valid_from'] ?: null,
                'valid_until' => $_POST['valid_until'] ?: null,
                'conditions' => json_encode($_POST['conditions'] ?? []),
            ]);

            redir('created=1');
        }

        $products = Capsule::table('tblproducts')->get(['id', 'name']);
        $categories = Capsule::table('tblproductgroups')->get(['id', 'name']);

        echo '<h2>Create Promo Code</h2>';
        echo '<form method="post" class="promo-form">';
        echo '<input type="hidden" name="_token" value="' . generate_token() . '">';

        echo '<div class="form-group"><label>Code</label><input type="text" name="code" required></div>';
        echo '<div class="form-group"><label>Type</label><select name="type" onchange="toggleBOGO(this.value)">';
        echo '<option value="percentage">Percentage</option>';
        echo '<option value="fixed">Fixed Amount</option>';
        echo '<option value="bogo">Buy One Get One</option>';
        echo '<option value="free_shipping">Free Shipping</option>';
        echo '</select></div>';
        echo '<div class="form-group"><label>Value</label><input type="number" step="0.01" name="value" required></div>';
        echo '<div class="form-group"><label>Max Discount</label><input type="number" step="0.01" name="max_discount"></div>';
        echo '<div class="form-group"><label>Max Uses (total)</label><input type="number" name="max_uses"></div>';
        echo '<div class="form-group"><label>Max Uses Per User</label><input type="number" name="max_uses_per_user" value="1"></div>';
        echo '<div class="form-group"><label>Valid From</label><input type="date" name="valid_from"></div>';
        echo '<div class="form-group"><label>Valid Until</label><input type="date" name="valid_until"></div>';

        echo '<button type="submit" class="btn btn-primary">Create Code</button>';
        echo '</form>';
    } else {
        $codes = Capsule::table('mod_promo_codes')
            ->orderBy('created_at', 'desc')
            ->get();

        echo '<h2>Promotional Codes</h2>';
        echo '<a href="?action=create" class="btn btn-primary">Create Code</a>';
        echo '<table class="datatable"><thead><tr>';
        echo '<th>Code</th><th>Type</th><th>Value</th><th>Uses</th><th>Status</th>';
        echo '</tr></thead><tbody>';

        foreach ($codes as $c) {
            echo '<tr>';
            echo "<td>{$c->code}</td>";
            echo "<td>{$c->type}</td>";
            echo "<td>{$c->value}" . ($c->discount_type === 'percentage' ? '%' : '$') . "</td>";
            echo "<td>{$c->uses_count}" . ($c->max_uses ? " / {$c->max_uses}" : '') . "</td>";
            echo "<td>" . ($c->is_active ? 'Active' : 'Inactive') . "</td>";
            echo '</tr>';
        }

        echo '</tbody></table>';
    }
}
```

## Cart Integration

```php
<?php
add_hook('ShoppingCartValidateCheckout', 1, function($vars) {
    $promoCode = $_SESSION['cart']['promo_code'] ?? null;

    if (!$promoCode) return null;

    $manager = new PromoCodeManager();
    $result = $manager->validate($promoCode, $_SESSION['uid'] ?? 0, $vars);

    if (!$result['valid']) {
        return ['allowed' => false, 'errormessage' => $result['error']];
    }

    return null;
});

add_hook('AfterCartCalculateTotals', 1, function($vars) {
    $promoCode = $_SESSION['cart']['promo_code'] ?? null;

    if (!$promoCode) return;

    $manager = new PromoCodeManager();
    $result = $manager->validate($promoCode, $_SESSION['uid'] ?? 0, $vars);

    if ($result['valid']) {
        $discount = $manager->calculateDiscount($result['promo'], $vars);
        $_SESSION['cart']['promo_discount'] = $discount;
    }
});
```

---

**Related Skills:**
- whmcs-coupon-system
- whmcs-pricing-strategy
- whmcs-volume-discounts