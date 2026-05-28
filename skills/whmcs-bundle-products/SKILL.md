# WHMCS Bundle Products Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Create and manage product bundles with discounted pricing.

## Database Schema

```php
<?php
// modules/addons/bundle_products/bundle_products.php

use WHMCS\Database\Capsule;

function bundle_products_config(): array {
    return [
        'name' => 'Bundle Products',
        'description' => 'Product bundling with special pricing',
        'version' => '1.0',
    ];
}

function bundle_products_activate(): array {
    Capsule::schema()->create('mod_product_bundles', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->text('description')->nullable();
        $t->string('sku', 50)->nullable();
        $t->decimal('bundle_price', 10, 2);
        $t->decimal('regular_price', 10, 2);
        $t->string('discount_type', 20)->default('percentage');
        $t->decimal('discount_value', 10, 2)->default(0);
        $t->string('billing_cycle', 30)->default('monthly');
        $t->boolean('allow_separate_purchase')->default(true);
        $t->string('status', 20)->default('active');
        $t->text('meta_data')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_bundle_items', function($t) {
        $t->increments('id');
        $t->integer('bundle_id')->unsigned();
        $t->string('item_type', 30);
        $t->integer('item_id')->unsigned();
        $t->string('item_name', 100);
        $t->integer('quantity')->unsigned()->default(1);
        $t->boolean('is_required')->default(true);
        $t->boolean('is_selectable')->default(false);
        $t->decimal('item_price', 10, 2);
        $t->integer('sort_order')->default(0);
    });

    Capsule::schema()->create('mod_bundle_options', function($t) {
        $t->increments('id');
        $t->integer('bundle_item_id')->unsigned();
        $t->string('option_name', 100);
        $t->integer('option_value_id')->unsigned();
        $t->decimal('additional_price', 10, 2)->default(0);
    });

    Capsule::schema()->create('mod_bundle_categories', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->text('description')->nullable();
        $t->string('icon', 50)->nullable();
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_bundle_category_map', function($t) {
        $t->increments('id');
        $t->integer('bundle_id')->unsigned();
        $t->integer('category_id')->unsigned();
    });

    return ['status' => 'success'];
}

function bundle_products_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_bundle_category_map');
    Capsule::schema()->dropIfExists('mod_bundle_categories');
    Capsule::schema()->dropIfExists('mod_bundle_options');
    Capsule::schema()->dropIfExists('mod_bundle_items');
    Capsule::schema()->dropIfExists('mod_product_bundles');
    return ['status' => 'success'];
}
```

## Bundle Manager

```php
<?php
class BundleManager {
    public function createBundle(array $data): int {
        $bundleId = Capsule::table('mod_product_bundles')->insertGetId([
            'name' => $data['name'],
            'description' => $data['description'] ?? null,
            'sku' => $data['sku'] ?? null,
            'bundle_price' => $data['bundle_price'],
            'regular_price' => $data['regular_price'],
            'discount_type' => $data['discount_type'] ?? 'percentage',
            'discount_value' => $data['discount_value'] ?? 0,
            'billing_cycle' => $data['billing_cycle'] ?? 'monthly',
            'allow_separate_purchase' => $data['allow_separate_purchase'] ?? true,
        ]);

        // Add items
        if (!empty($data['items'])) {
            foreach ($data['items'] as $index => $item) {
                $this->addBundleItem($bundleId, $item, $index);
            }
        }

        // Calculate discount if not provided
        if (empty($data['bundle_price']) && !empty($data['discount_value'])) {
            $this->calculateBundlePrice($bundleId);
        }

        return $bundleId;
    }

    public function addBundleItem(int $bundleId, array $itemData, int $sortOrder = 0): int {
        $itemPrice = $this->getItemPrice($itemData['item_type'], $itemData['item_id']);

        return Capsule::table('mod_bundle_items')->insertGetId([
            'bundle_id' => $bundleId,
            'item_type' => $itemData['item_type'],
            'item_id' => $itemData['item_id'],
            'item_name' => $itemData['name'],
            'quantity' => $itemData['quantity'] ?? 1,
            'is_required' => $itemData['is_required'] ?? true,
            'is_selectable' => $itemData['is_selectable'] ?? false,
            'item_price' => $itemPrice,
            'sort_order' => $sortOrder,
        ]);
    }

    private function getItemPrice(string $type, int $id): float {
        return match($type) {
            'product' => Capsule::table('tblpricing')
                ->where('type', 'product')
                ->where('relid', $id)
                ->first()->monthly ?? 0,
            'addon' => Capsule::table('tblpricing')
                ->where('type', 'addon')
                ->where('relid', $id)
                ->first()->monthly ?? 0,
            default => 0,
        };
    }

    private function calculateBundlePrice(int $bundleId): void {
        $items = $this->getBundleItems($bundleId);

        $totalPrice = 0;
        foreach ($items as $item) {
            $totalPrice += $item->item_price * $item->quantity;
        }

        $bundle = Capsule::table('mod_product_bundles')->find($bundleId);
        $bundlePrice = $bundle->discount_type === 'percentage'
            ? $totalPrice * (1 - $bundle->discount_value / 100)
            : $totalPrice - $bundle->discount_value;

        Capsule::table('mod_product_bundles')
            ->where('id', $bundleId)
            ->update([
                'regular_price' => $totalPrice,
                'bundle_price' => $bundlePrice,
            ]);
    }

    public function getBundle(int $bundleId): ?object {
        $bundle = Capsule::table('mod_product_bundles')->find($bundleId);

        if ($bundle) {
            $bundle->items = $this->getBundleItems($bundleId);
            $bundle->categories = $this->getBundleCategories($bundleId);
        }

        return $bundle;
    }

    public function getBundleItems(int $bundleId): array {
        return Capsule::table('mod_bundle_items')
            ->where('bundle_id', $bundleId)
            ->orderBy('sort_order')
            ->get();
    }

    public function getBundleCategories(int $bundleId): array {
        return Capsule::table('mod_bundle_category_map')
            ->join('mod_bundle_categories', 'mod_bundle_category_map.category_id', '=', 'mod_bundle_categories.id')
            ->where('bundle_id', $bundleId)
            ->get();
    }

    public function calculateBundleDiscount(int $bundleId): array {
        $bundle = $this->getBundle($bundleId);

        if (!$bundle) {
            return ['success' => false, 'error' => 'Bundle not found'];
        }

        $itemsTotal = 0;
        foreach ($bundle->items as $item) {
            $itemsTotal += $item->item_price * $item->quantity;
        }

        $savings = $itemsTotal - $bundle->bundle_price;

        return [
            'success' => true,
            'items_total' => $itemsTotal,
            'bundle_price' => $bundle->bundle_price,
            'savings' => $savings,
            'savings_percentage' => ($savings / $itemsTotal) * 100,
        ];
    }

    public function purchaseBundle(int $userId, int $bundleId, array $selectedOptions = []): array {
        $bundle = $this->getBundle($bundleId);

        if (!$bundle || $bundle->status !== 'active') {
            return ['success' => false, 'error' => 'Bundle not available'];
        }

        $orderParams = [
            'clientid' => $userId,
            'pid' => 0,
            'billingcycle' => $bundle->billing_cycle,
        ];

        $createdServices = [];
        $createdAddons = [];

        foreach ($bundle->items as $item) {
            if ($item->item_type === 'product' && $item->is_required) {
                $productParams = [
                    'clientid' => $userId,
                    'pid' => $item->item_id,
                    'billingcycle' => $bundle->billing_cycle,
                    'qty' => $item->quantity,
                ];

                $result = localAPI('AddOrder', $productParams);

                if ($result['result'] === 'success') {
                    $createdServices[] = $result['orderid'];
                }
            } elseif ($item->item_type === 'addon') {
                $addonParams = [
                    'clientid' => $userId,
                    'addonid' => $item->item_id,
                ];

                $result = localAPI('AddAddon', $addonParams);

                if ($result['result'] === 'success') {
                    $createdAddons[] = $result['addonid'];
                }
            }
        }

        return [
            'success' => true,
            'bundle_id' => $bundleId,
            'services' => $createdServices,
            'addons' => $createdAddons,
            'total_value' => $bundle->bundle_price,
        ];
    }

    public function getBundlesByCategory(int $categoryId): array {
        $bundleIds = Capsule::table('mod_bundle_category_map')
            ->where('category_id', $categoryId)
            ->pluck('bundle_id');

        if ($bundleIds->isEmpty()) {
            return [];
        }

        return Capsule::table('mod_product_bundles')
            ->whereIn('id', $bundleIds->toArray())
            ->where('status', 'active')
            ->get();
    }

    public function searchBundles(string $query): array {
        return Capsule::table('mod_product_bundles')
            ->where('status', 'active')
            ->where(function($q) use ($query) {
                $q->where('name', 'like', "%{$query}%")
                  ->orWhere('description', 'like', "%{$query}%")
                  ->orWhere('sku', 'like', "%{$query}%");
            })
            ->get();
    }
}
```

## Cart Integration

```php
<?php
add_hook('CartProductCalculations', 1, function($vars) {
    $productId = $vars['product_id'];

    $bundle = Capsule::table('mod_product_bundles')
        ->where('status', 'active')
        ->first();

    $isInBundle = Capsule::table('mod_bundle_items')
        ->where('item_type', 'product')
        ->where('item_id', $productId)
        ->exists();

    if ($isInBundle && $bundle) {
        $discount = $this->calculateBundleDiscount($bundle->id);

        return [
            'price' => $vars['base_price'],
            'bundle_discount' => $discount['savings'] ?? 0,
            'is_bundle_product' => true,
        ];
    }

    return null;
});

add_hook('OrderFormViewProduct', 1, function($vars) {
    $productId = $vars['product_id'];

    $bundles = Capsule::table('mod_bundle_items')
        ->join('mod_product_bundles', 'mod_bundle_items.bundle_id', '=', 'mod_product_bundles.id')
        ->where('mod_bundle_items.item_id', $productId)
        ->where('mod_bundle_items.item_type', 'product')
        ->where('mod_product_bundles.status', 'active')
        ->get();

    if (!empty($bundles)) {
        return [
            'bundles_containing' => $bundles,
        ];
    }
});
```

---

**Related Skills:**
- whmcs-addon-products
- whmcs-pricing-strategy
- whmcs-product-configurator