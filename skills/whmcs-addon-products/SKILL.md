# WHMCS Addon Products Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Optional addon products for core services with flexible pricing.

## Database Schema

```php
<?php
// modules/addons/addon_products/addon_products.php

use WHMCS\Database\Capsule;

function addon_products_config(): array {
    return [
        'name' => 'Addon Products',
        'description' => 'Optional addons for services',
        'version' => '1.0',
    ];
}

function addon_products_activate(): array {
    Capsule::schema()->create('mod_recommended_addons', function($t) {
        $t->increments('id');
        $t->integer('addon_id')->unsigned();
        $t->integer('product_id')->unsigned()->nullable();
        $t->string('trigger', 50)->default('purchase');
        $t->integer('position')->default(0);
        $t->boolean('is_active')->default(true);
    });

    Capsule::schema()->create('mod_addon_recommendations', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->integer('addon_id')->unsigned();
        $t->string('recommendation_type', 30);
        $t->decimal('offered_price', 10, 2)->nullable();
        $t->date('offer_expires')->nullable();
        $t->boolean('is_viewed')->default(false);
        $t->boolean('is_converted')->default(false);
        $t->timestamp('created_at')->useCurrent();
    });

    Capsule::schema()->create('mod_addon_bundles', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->text('description')->nullable();
        $t->string('discount_type', 20)->default('percentage');
        $t->decimal('discount_value', 10, 2);
        $t->string('applies_to', 50)->default('all');
        $t->text('addon_ids')->nullable();
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_addon_relationships', function($t) {
        $t->increments('id');
        $t->integer('addon_id')->unsigned();
        $t->integer('related_addon_id')->unsigned();
        $t->string('relationship_type', 30);
        $t->boolean('is_required')->default(false);
    });

    return ['status' => 'success'];
}

function addon_products_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_addon_relationships');
    Capsule::schema()->dropIfExists('mod_addon_bundles');
    Capsule::schema()->dropIfExists('mod_addon_recommendations');
    Capsule::schema()->dropIfExists('mod_recommended_addons');
    return ['status' => 'success'];
}
```

## Addon Manager

```php
<?php
class AddonProductManager {
    public function getAvailableAddons(?int $productId = null, ?int $userId = null): array {
        $query = Capsule::table('tbladdons')
            ->where('packages', 'like', '%' . ($productId ?? '') . '%')
            ->orWhere('packages', '')
            ->orWhereNull('packages');

        if ($userId) {
            $existingAddons = $this->getUserExistingAddons($userId);
            $query->whereNotIn('id', $existingAddons);
        }

        return $query->get();
    }

    private function getUserExistingAddons(int $userId): array {
        return Capsule::table('tblhostingaddons')
            ->where('userid', $userId)
            ->pluck('addonid')
            ->toArray();
    }

    public function getRecommendedAddons(int $productId, ?int $userId = null): array {
        $recommendations = Capsule::table('mod_recommended_addons')
            ->join('tbladdons', 'mod_recommended_addons.addon_id', '=', 'tbladdons.id')
            ->where('mod_recommended_addons.product_id', $productId)
            ->where('mod_recommended_addons.is_active', 1)
            ->orderBy('mod_recommended_addons.position')
            ->get();

        return $recommendations->toArray();
    }

    public function createAddonBundle(string $name, array $addonIds, float $discount, string $type = 'percentage'): int {
        return Capsule::table('mod_addon_bundles')->insertGetId([
            'name' => $name,
            'addon_ids' => json_encode($addonIds),
            'discount_type' => $type,
            'discount_value' => $discount,
            'applies_to' => 'selected',
        ]);
    }

    public function calculateBundlePrice(array $addonIds, float $discount, string $type = 'percentage'): array {
        $totalPrice = 0;
        $addons = [];

        foreach ($addonIds as $addonId) {
            $addon = Capsule::table('tbladdons')->find($addonId);
            $pricing = Capsule::table('tblpricing')
                ->where('type', 'addon')
                ->where('relid', $addonId)
                ->first();

            $price = $pricing->monthly ?? 0;
            $totalPrice += $price;

            $addons[] = [
                'id' => $addonId,
                'name' => $addon->name,
                'price' => $price,
            ];
        }

        $discountAmount = $type === 'percentage'
            ? $totalPrice * ($discount / 100)
            : $discount;

        return [
            'addons' => $addons,
            'total_price' => $totalPrice,
            'discount' => $discountAmount,
            'final_price' => $totalPrice - $discountAmount,
            'savings' => ($discountAmount / $totalPrice) * 100,
        ];
    }

    public function purchaseBundle(int $userId, int $serviceId, array $addonIds): array {
        $results = [];

        foreach ($addonIds as $addonId) {
            $result = localAPI('AddAddon', [
                'clientid' => $userId,
                'serviceid' => $serviceId,
                'addonid' => $addonId,
            ]);

            $results[$addonId] = $result;
        }

        return $results;
    }

    public function getRelatedAddons(int $addonId): array {
        return Capsule::table('mod_addon_relationships')
            ->join('tbladdons', 'mod_addon_relationships.related_addon_id', '=', 'tbladdons.id')
            ->where('mod_addon_relationships.addon_id', $addonId)
            ->get();
    }

    public function createRecommendation(int $userId, int $addonId, string $type, ?float $price = null, ?DateTime $expires = null): int {
        return Capsule::table('mod_addon_recommendations')->insertGetId([
            'user_id' => $userId,
            'addon_id' => $addonId,
            'recommendation_type' => $type,
            'offered_price' => $price,
            'offer_expires' => $expires?->format('Y-m-d'),
        ]);
    }

    public function getUserRecommendations(int $userId): array {
        return Capsule::table('mod_addon_recommendations')
            ->join('tbladdons', 'mod_addon_recommendations.addon_id', '=', 'tbladdons.id')
            ->where('mod_addon_recommendations.user_id', $userId)
            ->where('mod_addon_recommendations.is_converted', 0)
            ->where(function($q) {
                $q->whereNull('offer_expires')
                  ->orWhere('offer_expires', '>=', date('Y-m-d'));
            })
            ->get();
    }

    public function convertRecommendation(int $recommendationId): array {
        $recommendation = Capsule::table('mod_addon_recommendations')->find($recommendationId);

        if (!$recommendation) {
            return ['success' => false, 'error' => 'Recommendation not found'];
        }

        $user = Capsule::table('tblclients')->find($recommendation->user_id);
        $service = Capsule::table('tblhosting')
            ->where('userid', $user->id)
            ->orderBy('id', 'desc')
            ->first();

        if (!$service) {
            return ['success' => false, 'error' => 'No active service found'];
        }

        $result = localAPI('AddAddon', [
            'clientid' => $user->id,
            'serviceid' => $service->id,
            'addonid' => $recommendation->addon_id,
        ]);

        if ($result['result'] === 'success') {
            Capsule::table('mod_addon_recommendations')
                ->where('id', $recommendationId)
                ->update(['is_converted' => true]);
        }

        return $result;
    }

    public function getAddonAnalytics(): array {
        return [
            'total_addons' => Capsule::table('tbladdons')->count(),
            'total_purchases' => Capsule::table('tblhostingaddons')->count(),
            'top_addons' => Capsule::table('tblhostingaddons')
                ->selectRaw('addonid, COUNT(*) as count')
                ->groupBy('addonid')
                ->orderBy('count', 'desc')
                ->limit(10)
                ->get(),
            'conversion_rate' => $this->calculateConversionRate(),
        ];
    }

    private function calculateConversionRate(): float {
        $recommendations = Capsule::table('mod_addon_recommendations')->count();
        $conversions = Capsule::table('mod_addon_recommendations')
            ->where('is_converted', 1)
            ->count();

        return $recommendations > 0 ? ($conversions / $recommendations) * 100 : 0;
    }
}
```

## Trigger-Based Recommendations

```php
<?php
add_hook('AfterOrderPaid', 1, function($vars) {
    $order = Capsule::table('tblorders')->find($vars['orderid']);
    $userId = $order->userid;

    $manager = new AddonProductManager();
    $recommendations = $manager->getRecommendedAddons($order->pid, $userId);

    foreach ($recommendations as $addon) {
        if ($addon->trigger === 'purchase') {
            $manager->createRecommendation($userId, $addon->id, 'post_purchase');
        }
    }
});

add_hook('ServiceCreated', 1, function($vars) {
    $service = Capsule::table('tblhosting')->find($vars['serviceid']);
    $manager = new AddonProductManager();

    $addons = $manager->getRecommendedAddons($service->packageid, $service->userid);

    foreach ($addons as $addon) {
        if ($addon->trigger === 'activation') {
            $manager->createRecommendation($service->userid, $addon->id, 'service_activation');
        }
    }
});
```

## Client Area

```php
<?php
function addon_products_clientarea(array $vars): array {
    $userId = $_SESSION['uid'];
    $manager = new AddonProductManager();

    $recommendations = $manager->getUserRecommendations($userId);
    $userAddons = Capsule::table('tblhostingaddons')
        ->join('tbladdons', 'tblhostingaddons.addonid', '=', 'tbladdons.id')
        ->where('tblhostingaddons.userid', $userId)
        ->get(['tblhostingaddons.*', 'tbladdons.name', 'tbladdons.description']);

    return [
        'pagetitle' => 'Add-ons',
        'templatefile' => 'addons',
        'vars' => [
            'recommendations' => $recommendations,
            'user_addons' => $userAddons,
        ],
    ];
}
```

---

**Related Skills:**
- whmcs-bundle-products
- whmcs-product-configurator
- whmcs-pricing-strategy