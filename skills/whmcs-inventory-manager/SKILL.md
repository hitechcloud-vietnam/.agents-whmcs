# WHMCS Inventory Manager Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building inventory management modules for physical/digital products.

## When to Use

- Creating inventory tracking modules
- Building stock management systems
- Managing product availability

## Inventory Manager Patterns

```php
<?php
// modules/addons/{inventorymodule}/{inventorymodule}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {inventorymodule}_config(): array {
    return [
        'name' => 'Inventory Manager',
        'description' => 'Stock and inventory management system',
        'version' => '1.0',
        'author' => 'Author',
        'low_stock_threshold' => ['FriendlyName' => 'Low Stock Threshold', 'Type' => 'text', 'Default' => '10'],
        'auto_reserve' => ['FriendlyName' => 'Auto-reserve on Order', 'Type' => 'yesno'],
        'expiry_tracking' => ['FriendlyName' => 'Track Expiry Dates', 'Type' => 'yesno'],
    ];
}

function {inventorymodule}_activate(): array {
    Capsule::schema()->create('mod_inventory_products', function($t) {
        $t->increments('id');
        $t->string('sku', 100)->unique();
        $t->string('name', 255);
        $t->text('description')->nullable();
        $t->string('category', 100);
        $t->decimal('cost_price', 10, 2)->nullable();
        $t->decimal('selling_price', 10, 2)->nullable();
        $t->integer('quantity')->unsigned()->default(0);
        $t->integer('reserved_quantity')->unsigned()->default(0);
        $t->integer('low_stock_threshold')->unsigned()->default(10);
        $t->string('warehouse_location', 100)->nullable();
        $t->date('expiry_date')->nullable();
        $t->boolean('track_inventory')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_inventory_transactions', function($t) {
        $t->increments('id');
        $t->integer('product_id')->unsigned();
        $t->integer('order_id')->unsigned()->nullable();
        $t->string('type', 50);
        $t->integer('quantity_change');
        $t->integer('balance_after');
        $t->string('reference', 100)->nullable();
        $t->text('notes')->nullable();
        $t->integer('admin_id')->unsigned();
        $t->timestamp('created_at');
    });

    Capsule::schema()->create('mod_inventory_reservations', function($t) {
        $t->increments('id');
        $t->integer('product_id')->unsigned();
        $t->integer('order_id')->unsigned();
        $t->integer('quantity')->unsigned();
        $t->string('status', 20)->default('reserved');
        $t->timestamp('expires_at');
        $t->timestamp('created_at');
    });

    Capsule::schema()->create('mod_inventory_alerts', function($t) {
        $t->increments('id');
        $t->integer('product_id')->unsigned();
        $t->string('alert_type', 50);
        $t->string('message', 255);
        $t->boolean('acknowledged')->default(false);
        $t->timestamp('created_at');
    });

    return ['status' => 'success', 'description' => 'Inventory module activated'];
}

function {inventorymodule}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_inventory_products');
    Capsule::schema()->dropIfExists('mod_inventory_transactions');
    Capsule::schema()->dropIfExists('mod_inventory_reservations');
    Capsule::schema()->dropIfExists('mod_inventory_alerts');
    return ['status' => 'success', 'description' => 'Inventory module deactivated'];
}

function {inventorymodule}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';
   include __DIR__ . '/templates/admin/' . $action . '.tpl';
}
```

### Inventory Management Functions

```php
function getAvailableQuantity(int $productId): int {
    $product = Capsule::table('mod_inventory_products')
        ->where('id', $productId)
        ->first();

    if (!$product) {
        return 0;
    }

    return max(0, $product->quantity - $product->reserved_quantity);
}

function reserveStock(int $productId, int $quantity, int $orderId): bool {
    $available = getAvailableQuantity($productId);

    if ($available < $quantity) {
        return false;
    }

    // Create reservation
    Capsule::table('mod_inventory_reservations')->insert([
        'product_id' => $productId,
        'order_id' => $orderId,
        'quantity' => $quantity,
        'status' => 'reserved',
        'expires_at' => date('Y-m-d H:i:s', strtotime('+30 days')),
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    // Update reserved quantity
    Capsule::table('mod_inventory_products')
        ->where('id', $productId)
        ->increment('reserved_quantity', $quantity);

    return true;
}

function confirmReservation(int $orderId): void {
    $reservations = Capsule::table('mod_inventory_reservations')
        ->where('order_id', $orderId)
        ->where('status', 'reserved')
        ->get();

    foreach ($reservations as $reservation) {
        // Deduct from actual stock
        Capsule::table('mod_inventory_products')
            ->where('id', $reservation->product_id)
            ->decrement('quantity', $reservation->quantity);

        // Log transaction
        logInventoryTransaction(
            $reservation->product_id,
            $orderId,
            'order',
            -$reservation->quantity,
            $reservation->product_id,
            $orderId,
            'Stock sold'
        );

        // Mark reservation as fulfilled
        Capsule::table('mod_inventory_reservations')
            ->where('id', $reservation->id)
            ->update(['status' => 'fulfilled']);
    }
}

function releaseReservation(int $orderId): void {
    $reservations = Capsule::table('mod_inventory_reservations')
        ->where('order_id', $orderId)
        ->where('status', 'reserved')
        ->get();

    foreach ($reservations as $reservation) {
        // Decrease reserved quantity
        Capsule::table('mod_inventory_products')
            ->where('id', $reservation->product_id)
            ->decrement('reserved_quantity', $reservation->quantity);

        Capsule::table('mod_inventory_reservations')
            ->where('id', $reservation->id)
            ->update(['status' => 'released']);
    }
}

function addStock(int $productId, int $quantity, int $adminId, string $reference = '', string $notes = ''): void {
    Capsule::table('mod_inventory_products')
        ->where('id', $productId)
        ->increment('quantity', $quantity);

    logInventoryTransaction(
        $productId,
        0,
        'purchase',
        $quantity,
        Capsule::table('mod_inventory_products')->where('id', $productId)->value('quantity') + $quantity,
        $adminId,
        $notes,
        $reference
    );

    // Check low stock
    checkLowStock($productId);
}

function removeStock(int $productId, int $quantity, int $adminId, string $notes, string $reference = ''): bool {
    $product = Capsule::table('mod_inventory_products')
        ->where('id', $productId)
        ->first();

    if ($product->quantity < $quantity) {
        return false;
    }

    Capsule::table('mod_inventory_products')
        ->where('id', $productId)
        ->decrement('quantity', $quantity);

    logInventoryTransaction(
        $productId,
        0,
        'adjustment',
        -$quantity,
        $product->quantity - $quantity,
        $adminId,
        $notes,
        $reference
    );

    return true;
}

function logInventoryTransaction(
    int $productId,
    int $orderId,
    string $type,
    int $quantityChange,
    int $balanceAfter,
    int $adminId,
    string $notes = '',
    string $reference = ''
): void {
    Capsule::table('mod_inventory_transactions')->insert([
        'product_id' => $productId,
        'order_id' => $orderId ?: null,
        'type' => $type,
        'quantity_change' => $quantityChange,
        'balance_after' => $balanceAfter,
        'reference' => $reference ?: null,
        'notes' => $notes,
        'admin_id' => $adminId,
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

function checkLowStock(int $productId): void {
    $product = Capsule::table('mod_inventory_products')->where('id', $productId)->first();

    if ($product->quantity <= $product->low_stock_threshold) {
        Capsule::table('mod_inventory_alerts')->insert([
            'product_id' => $productId,
            'alert_type' => 'low_stock',
            'message' => "Low stock alert: {$product->name} (Quantity: {$product->quantity})",
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

### Stock Report

```php
function getInventoryReport(array $filters = []): array {
    $query = Capsule::table('mod_inventory_products')
        ->selectRaw('
            mod_inventory_products.*,
            (SELECT SUM(quantity_change) FROM mod_inventory_transactions WHERE product_id = mod_inventory_products.id AND type = "purchase" AND created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)) as total_purchased,
            (SELECT SUM(ABS(quantity_change)) FROM mod_inventory_transactions WHERE product_id = mod_inventory_products.id AND type = "order" AND created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)) as total_sold,
            (SELECT COUNT(*) FROM mod_inventory_alerts WHERE product_id = mod_inventory_products.id AND acknowledged = 0) as active_alerts
        ')
        ->where('track_inventory', 1);

    if (!empty($filters['category'])) {
        $query->where('category', $filters['category']);
    }

    if (!empty($filters['low_stock'])) {
        $query->whereRaw('quantity <= low_stock_threshold');
    }

    if (!empty($filters['out_of_stock'])) {
        $query->where('quantity', 0);
    }

    return $query->get();
}
```

### Hook Integration

```php
add_hook('OrderPaid', 1, function($vars) {
    $order = Capsule::table('tblorders')
        ->where('id', $vars['order_id'])
        ->first();

    $items = Capsule::table('tblorderitems')
        ->where('order_id', $vars['order_id'])
        ->get();

    foreach ($items as $item) {
        $inventoryProduct = Capsule::table('mod_inventory_products')
            ->where('sku', $item->item_sku ?? '')
            ->first();

        if ($inventoryProduct && $inventoryProduct->track_inventory) {
            if (!reserveStock($inventoryProduct->id, $item->qty, $vars['order_id'])) {
                logActivity("Failed to reserve stock for order {$vars['order_id']}, product: {$inventoryProduct->sku}");
            }
        }
    }
});

add_hook('OrderCancelled', 1, function($vars) {
    releaseReservation($vars['order_id']);
});

add_hook('OrderCompleted', 1, function($vars) {
    confirmReservation($vars['order_id']);
});
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-reporting
- whmcs-cron-automation
