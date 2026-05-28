# WHMCS Coupon System Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build discount coupon and promotion modules.

## Coupon Module Structure

```php
<?php
/**
 * Coupon Module (Addon)
 * Location: modules/addons/{module}/
 */

function {module}_config(): array {
    return [
        'name' => 'Coupon System',
        'description' => 'Discount coupons and promotions',
        'version' => '1.0',
    ];
}

function {module}_activate(): array {
    Capsule::schema()->create('mod_coupons', function($t) {
        $t->increments('id');
        $t->string('code', 50)->unique();
        $t->string('type', 20);
        $t->decimal('value', 10, 2);
        $t->decimal('max_discount', 10, 2)->nullable();
        $t->integer('max_uses')->unsigned()->nullable();
        $t->integer('uses_count')->unsigned()->default(0);
        $t->integer('max_uses_per_user')->unsigned()->default(1);
        $t->integer('min_order_value', 10, 2)->nullable();
        $t->integer('user_id')->unsigned()->nullable();
        $t->string('applies_to', 50)->default('all');
        $t->text('product_ids')->nullable();
        $t->date('valid_from')->nullable();
        $t->date('valid_until')->nullable();
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_coupon_usage', function($t) {
        $t->increments('id');
        $t->integer('coupon_id')->unsigned();
        $t->integer('user_id')->unsigned();
        $t->integer('order_id')->unsigned()->nullable();
        $t->timestamp('used_at');

        $t->index(['coupon_id', 'user_id']);
    });

    return ['status' => 'success'];
}

function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_coupons');
    Capsule::schema()->dropIfExists('mod_coupon_usage');
    return ['status' => 'success'];
}
```

## Coupon Validation

```php
<?php
class CouponValidator {
    public function validate(string $code, int $userId, float $orderTotal, array $items = []): array {
        $coupon = Capsule::table('mod_coupons')
            ->where('code', strtoupper($code))
            ->where('is_active', 1)
            ->first();

        if (!$coupon) {
            return ['valid' => false, 'error' => 'Invalid coupon code'];
        }

        if ($coupon->user_id && $coupon->user_id !== $userId) {
            return ['valid' => false, 'error' => 'Coupon not available for this account'];
        }

        if ($coupon->valid_from && $coupon->valid_from > date('Y-m-d')) {
            return ['valid' => false, 'error' => 'Coupon not yet valid'];
        }

        if ($coupon->valid_until && $coupon->valid_until < date('Y-m-d')) {
            return ['valid' => false, 'error' => 'Coupon has expired'];
        }

        if ($coupon->max_uses && $coupon->uses_count >= $coupon->max_uses) {
            return ['valid' => false, 'error' => 'Coupon usage limit reached'];
        }

        $userUsage = Capsule::table('mod_coupon_usage')
            ->where('coupon_id', $coupon->id)
            ->where('user_id', $userId)
            ->count();

        if ($userUsage >= $coupon->max_uses_per_user) {
            return ['valid' => false, 'error' => 'You have already used this coupon'];
        }

        if ($coupon->min_order_value && $orderTotal < $coupon->min_order_value) {
            return ['valid' => false, 'error' => "Minimum order value: {$coupon->min_order_value}"];
        }

        return ['valid' => true, 'coupon' => $coupon];
    }

    public function calculateDiscount(object $coupon, float $orderTotal, array $items = []): float {
        if ($coupon->type === 'percentage') {
            $discount = $orderTotal * ($coupon->value / 100);
        } else {
            $discount = $coupon->value;
        }

        if ($coupon->max_discount && $discount > $coupon->max_discount) {
            $discount = $coupon->max_discount;
        }

        return min($discount, $orderTotal);
    }

    public function recordUsage(int $couponId, int $userId, ?int $orderId = null): void {
        Capsule::table('mod_coupon_usage')->insert([
            'coupon_id' => $couponId,
            'user_id' => $userId,
            'order_id' => $orderId,
            'used_at' => date('Y-m-d H:i:s'),
        ]);

        Capsule::table('mod_coupons')
            ->where('id', $couponId)
            ->increment('uses_count');
    }
}
```

## Cart Integration

```php
add_hook('ShoppingCartValidateCheckout', 1, function($vars) {
    $couponCode = $_SESSION['cart']['coupon'] ?? null;

    if (!$couponCode) return null;

    $validator = new CouponValidator();
    $result = $validator->validate(
        $couponCode,
        $_SESSION['uid'] ?? 0,
        $vars['total'] ?? 0
    );

    if (!$result['valid']) {
        return ['allowed' => false, 'errormessage' => $result['error']];
    }

    return null;
});

add_hook('AfterCartCalculateTotals', 1, function($vars) {
    $couponCode = $_SESSION['cart']['coupon'] ?? null;

    if ($couponCode) {
        $validator = new CouponValidator();
        $result = $validator->validate($couponCode, $_SESSION['uid'] ?? 0, $vars['subtotal']);

        if ($result['valid']) {
            $discount = $validator->calculateDiscount($result['coupon'], $vars['subtotal']);
            $_SESSION['cart']['discount'] = $discount;
        }
    }
});
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-pricing-engine