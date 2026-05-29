# WHMCS Order Processing Hooks Module

## Overview
Complete order processing automation module handling OrderPaid, ServiceCreated, and ModuleChangePackage hooks with fulfillment workflows.

## Module File: hooks.php

```php
<?php
/**
 * WHMCS Order Processing Hooks Module
 * 
 * @package    WHMCS
 * @subpackage Modules
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Hook: OrderPaid
 * Triggered when order payment is completed
 */
function whmcs_order_processing_order_paid(array $params): array
{
    try {
        $orderId = $params['orderid'];
        $userId = $params['userid'];
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        logModuleCall('OrderProcessing', 'OrderPaid', ['order_id' => $orderId], 'Order paid', '');

        // Get order details
        $order = Capsule::table('tblorders')
            ->where('id', $orderId)
            ->first();

        // Store order payment event
        Capsule::table('mod_order_events')->insert([
            'order_id' => $orderId,
            'event_type' => 'paid',
            'event_data' => json_encode([
                'user_id' => $userId,
                'total' => $params['totalamount'],
                'payment_method' => $params['payment_method'] ?? 'unknown',
                'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'cli',
            ]),
            'created_at' => $now,
        ]);

        // Get order items
        $orderItems = Capsule::table('tblorderitems')
            ->where('orderid', $orderId)
            ->get(['id', 'type', 'relid', 'productid', 'name', 'amount']);

        // Process each order item
        foreach ($orderItems as $item) {
            processOrderItem($orderId, $userId, $item, $config);
        }

        // Send order confirmation
        if (!empty($config['send_confirmation'])) {
            $command = 'SendEmail';
            $postData = [
                'id' => $orderId,
                'type' => 'order',
            ];
            localAPI($command, $postData);
        }

        // Trigger fulfillment webhook
        if (!empty($config['fulfillment_webhook_url'])) {
            triggerFulfillmentWebhook($orderId, $params);
        }

        // Update affiliate tracking
        if (!empty($config['affiliate_tracking'])) {
            updateAffiliateCommission($userId, $orderId, $params['totalamount']);
        }

        // High-value order alerts
        if (!empty($config['high_value_threshold']) && $params['totalamount'] >= $config['high_value_threshold']) {
            sendAdminAlert('high_value_order', [
                'order_id' => $orderId,
                'user_id' => $userId,
                'total' => $params['totalamount'],
            ]);
        }

    } catch (\Exception $e) {
        logModuleCall('OrderProcessing', 'OrderPaid Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: ServiceCreated
 * Triggered when a service/hosting account is created
 */
function whmcs_order_processing_service_created(array $params): array
{
    try {
        $serviceId = $params['serviceid'];
        $userId = $params['userid'];
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        logModuleCall('OrderProcessing', 'ServiceCreated', ['service_id' => $serviceId], 'Service created', '');

        // Get service details
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();

        // Store service creation event
        Capsule::table('mod_order_events')->insert([
            'order_id' => $service->orderid ?? 0,
            'event_type' => 'service_created',
            'event_data' => json_encode([
                'service_id' => $serviceId,
                'user_id' => $userId,
                'product_id' => $service->packageid ?? null,
                'domain' => $service->domain ?? null,
            ]),
            'created_at' => $now,
        ]);

        // Create welcome message for the service
        if (!empty($config['auto_welcome_message'])) {
            createWelcomeMessage($serviceId, $userId);
        }

        // Provision additional features
        if (!empty($config['auto_addons'])) {
            provisionAutoAddons($serviceId, $service->packageid ?? null);
        }

        // Send provisioning complete notification
        if (!empty($config['send_provisioning_notification'])) {
            $command = 'SendEmail';
            $postData = [
                'id' => $serviceId,
                'type' => 'product',
                'templatename' => 'Service Provisioned',
            ];
            localAPI($command, $postData);
        }

    } catch (\Exception $e) {
        logModuleCall('OrderProcessing', 'ServiceCreated Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: ModuleChangePackage
 * Triggered when a service package is upgraded/downgraded
 */
function whmcs_order_processing_module_change_package(array $params): array
{
    try {
        $serviceId = $params['serviceid'];
        $userId = $params['userid'];
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        $oldPackageId = $params['old_product_id'] ?? null;
        $newPackageId = $params['new_product_id'] ?? $params['productid'];
        
        logModuleCall('OrderProcessing', 'ModuleChangePackage', [
            'service_id' => $serviceId,
            'old_package' => $oldPackageId,
            'new_package' => $newPackageId,
        ], 'Package changed', '');

        // Get package details
        $oldPackage = $oldPackageId ? Capsule::table('tblproducts')
            ->where('id', $oldPackageId)
            ->first() : null;
        
        $newPackage = Capsule::table('tblproducts')
            ->where('id', $newPackageId)
            ->first();

        // Store upgrade event
        Capsule::table('mod_order_events')->insert([
            'order_id' => 0,
            'event_type' => 'package_changed',
            'event_data' => json_encode([
                'service_id' => $serviceId,
                'user_id' => $userId,
                'old_package_id' => $oldPackageId,
                'old_package_name' => $oldPackage->name ?? 'N/A',
                'new_package_id' => $newPackageId,
                'new_package_name' => $newPackage->name ?? 'N/A',
                'is_upgrade' => isUpgrade($oldPackage, $newPackage),
            ]),
            'created_at' => $now,
        ]);

        // Handle prorated billing for upgrades
        if (!empty($config['prorate_upgrades']) && isUpgrade($oldPackage, $newPackage)) {
            processUpgradeProration($serviceId, $oldPackage, $newPackage);
        }

        // Notify about change
        if (!empty($config['notify_package_change'])) {
            $command = 'SendEmail';
            $postData = [
                'id' => $serviceId,
                'type' => 'product',
                'templatename' => 'Package Upgrade Confirmation',
            ];
            localAPI($command, $postData);
        }

        // Sync with external systems
        if (!empty($config['sync_webhook_url'])) {
            syncPackageChange($serviceId, $oldPackageId, $newPackageId);
        }

    } catch (\Exception $e) {
        logModuleCall('OrderProcessing', 'ModuleChangePackage Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Process individual order item
 */
function processOrderItem(int $orderId, int $userId, object $item, array $config): void
{
    $itemType = $item->type;
    
    switch ($itemType) {
        case 'hosting':
            // Hosting account provisioning is handled by AfterModuleCreate hook
            break;
        case 'domain':
            processDomainOrder($orderId, $item, $config);
            break;
        case 'addon':
            processAddonOrder($orderId, $item, $config);
            break;
        case 'upgrade':
            processUpgradeOrder($orderId, $item, $config);
            break;
    }
}

/**
 * Process domain order
 */
function processDomainOrder(int $orderId, object $item, array $config): void
{
    Capsule::table('mod_order_events')->insert([
        'order_id' => $orderId,
        'event_type' => 'domain_ordered',
        'event_data' => json_encode([
            'domain' => $item->name,
            'relid' => $item->relid,
        ]),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Process addon order
 */
function processAddonOrder(int $orderId, object $item, array $config): void
{
    Capsule::table('mod_order_events')->insert([
        'order_id' => $orderId,
        'event_type' => 'addon_ordered',
        'event_data' => json_encode([
            'addon_id' => $item->relid,
            'addon_name' => $item->name,
            'amount' => $item->amount,
        ]),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Process upgrade order
 */
function processUpgradeOrder(int $orderId, object $item, array $config): void
{
    Capsule::table('mod_order_events')->insert([
        'order_id' => $orderId,
        'event_type' => 'upgrade_ordered',
        'event_data' => json_encode([
            'upgrade_option_id' => $item->relid,
            'amount' => $item->amount,
        ]),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Create welcome message for new service
 */
function createWelcomeMessage(int $serviceId, int $userId): void
{
    Capsule::table('mod_service_messages')->insert([
        'service_id' => $serviceId,
        'user_id' => $userId,
        'message_type' => 'welcome',
        'message' => 'Thank you for your order! Your service is now being set up.',
        'created_at' => date('Y-m-d H:i:s'),
        'is_read' => false,
    ]);
}

/**
 * Provision auto addons based on product
 */
function provisionAutoAddons(int $serviceId, ?int $productId): void
{
    if (!$productId) {
        return;
    }

    $autoAddons = Capsule::table('mod_product_auto_addons')
        ->where('product_id', $productId)
        ->where('auto_provision', true)
        ->get();

    foreach ($autoAddons as $addon) {
        $command = 'AddOrder';
        $postData = [
            'clientid' => Capsule::table('tblhosting')
                ->where('id', $serviceId)
                ->value('userid'),
            'pid' => $addon->addon_product_id,
            'billingcycle' => 'monthly',
        ];
        localAPI($command, $postData);
    }
}

/**
 * Update affiliate commission
 */
function updateAffiliateCommission(int $userId, int $orderId, float $total): void
{
    try {
        $affiliate = Capsule::table('tblaffiliates')
            ->where('clientid', $userId)
            ->first();

        if ($affiliate) {
            $commission = ($total * $affiliate->commission()) / 100;
            
            Capsule::table('tblaffiliates')
                ->where('id', $affiliate->id)
                ->increment('credit', $commission);

            Capsule::table('mod_order_events')->insert([
                'order_id' => $orderId,
                'event_type' => 'affiliate_commission',
                'event_data' => json_encode([
                    'affiliate_id' => $affiliate->id,
                    'order_total' => $total,
                    'commission_percent' => $affiliate->commission(),
                    'commission_amount' => $commission,
                ]),
                'created_at' => date('Y-m-d H:i:s'),
            ]);
        }
    } catch (\Exception $e) {
        logModuleCall('OrderProcessing', 'AffiliateCommission Error', ['user_id' => $userId], $e->getMessage(), '');
    }
}

/**
 * Trigger fulfillment webhook
 */
function triggerFulfillmentWebhook(int $orderId, array $params): void
{
    $config = require __DIR__ . '/config.php';
    
    wp_remote_post($config['fulfillment_webhook_url'], [
        'body' => json_encode([
            'event' => 'order.paid',
            'order_id' => $orderId,
            'user_id' => $params['userid'],
            'items' => $params['lineitems'] ?? [],
            'total' => $params['totalamount'],
            'timestamp' => date('Y-m-d H:i:s'),
        ]),
        'headers' => [
            'Content-Type' => 'application/json',
            'X-Webhook-Secret' => $config['webhook_secret'] ?? '',
        ],
    ]);
}

/**
 * Send admin alert
 */
function sendAdminAlert(string $type, array $data): void
{
    $command = 'SendAdminEmail';
    $postData = [
        'type' => $type,
        'customvars' => base64_encode(json_encode($data)),
    ];
    localAPI($command, $postData);
}

/**
 * Determine if upgrade
 */
function isUpgrade(?object $oldPackage, ?object $newPackage): bool
{
    if (!$oldPackage || !$newPackage) {
        return false;
    }
    return $newPackage->monthly > $oldPackage->monthly;
}

/**
 * Process upgrade proration
 */
function processUpgradeProration(int $serviceId, ?object $oldPackage, object $newPackage): void
{
    // Calculate prorated amount
    $daysRemaining = getDaysRemainingInCycle($serviceId);
    $oldMonthly = $oldPackage ? $oldPackage->monthly : 0;
    $newMonthly = $newPackage->monthly;
    
    $creditAmount = ($oldMonthly / 30) * $daysRemaining;
    $newAmount = ($newMonthly / 30) * $daysRemaining;
    $prorateAmount = max(0, $newAmount - $creditAmount);

    if ($prorateAmount > 0) {
        Capsule::table('mod_order_events')->insert([
            'order_id' => 0,
            'event_type' => 'upgrade_proration',
            'event_data' => json_encode([
                'service_id' => $serviceId,
                'credit_amount' => $creditAmount,
                'charge_amount' => $prorateAmount,
                'days_remaining' => $daysRemaining,
            ]),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

/**
 * Get days remaining in billing cycle
 */
function getDaysRemainingInCycle(int $serviceId): int
{
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first(['nextduedate']);

    if ($service && $service->nextduedate) {
        $nextDue = new DateTime($service->nextduedate);
        $now = new DateTime();
        return max(0, $now->diff($nextDue)->days);
    }
    
    return 15;
}

/**
 * Sync package change with external systems
 */
function syncPackageChange(int $serviceId, ?int $oldPackageId, int $newPackageId): void
{
    $config = require __DIR__ . '/config.php';
    
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();

    wp_remote_post($config['sync_webhook_url'], [
        'body' => json_encode([
            'event' => 'service.package_changed',
            'service_id' => $serviceId,
            'domain' => $service->domain ?? null,
            'old_package_id' => $oldPackageId,
            'new_package_id' => $newPackageId,
            'timestamp' => date('Y-m-d H:i:s'),
        ]),
        'headers' => [
            'Content-Type' => 'application/json',
            'X-Webhook-Secret' => $config['webhook_secret'] ?? '',
        ],
    ]);
}

// Register hooks
add_hook('OrderPaid', 1, 'whmcs_order_processing_order_paid');
add_hook('ServiceCreated', 1, 'whmcs_order_processing_service_created');
add_hook('ModuleChangePackage', 1, 'whmcs_order_processing_module_change_package');
```

## Configuration File: config.php

```php
<?php
return [
    // Notifications
    'send_confirmation' => true,
    'send_provisioning_notification' => true,
    'notify_package_change' => true,

    // Automation
    'auto_welcome_message' => true,
    'auto_addons' => false,
    'affiliate_tracking' => true,

    // Billing
    'prorate_upgrades' => true,

    // Alerts
    'high_value_threshold' => 500.00,

    // Webhooks
    'fulfillment_webhook_url' => '',
    'sync_webhook_url' => '',
    'webhook_secret' => '',

    // Logging
    'log_level' => 'info',
];
```

## Database Schema

```php
<?php
use WHMCS\Database\Capsule;

// Order events table
if (!Capsule::schema()->hasTable('mod_order_events')) {
    Capsule::schema()->create('mod_order_events', function ($table) {
        $table->increments('id');
        $table->integer('order_id')->unsigned();
        $table->string('event_type', 50);
        $table->longText('event_data')->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('order_id');
        $table->index('event_type');
        $table->index('created_at');
    });
}

// Service messages table
if (!Capsule::schema()->hasTable('mod_service_messages')) {
    Capsule::schema()->create('mod_service_messages', function ($table) {
        $table->increments('id');
        $table->integer('service_id')->unsigned();
        $table->integer('user_id')->unsigned();
        $table->string('message_type', 50);
        $table->text('message');
        $table->boolean('is_read')->default(false);
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('service_id');
        $table->index(['user_id', 'is_read']);
    });
}

// Product auto addons table
if (!Capsule::schema()->hasTable('mod_product_auto_addons')) {
    Capsule::schema()->create('mod_product_auto_addons', function ($table) {
        $table->increments('id');
        $table->integer('product_id')->unsigned();
        $table->integer('addon_product_id')->unsigned();
        $table->boolean('auto_provision')->default(false);
        $table->timestamp('created_at')->useCurrent();
        
        $table->unique(['product_id', 'addon_product_id']);
    });
}
```

## Activation & Deactivation

```php
<?php
function whmcs_order_processing_activate(): array
{
    try {
        require_once __DIR__ . '/schema_migration.php';
        return ['status' => 'success', 'description' => 'Order Processing Hooks activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_order_processing_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Order Processing Hooks deactivated'];
}
```

## Hooks Reference

| Hook | Description |
|------|-------------|
| OrderPaid | Order payment completed |
| ServiceCreated | New service/hosting created |
| ModuleChangePackage | Package upgraded/downgraded |

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
