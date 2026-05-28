# WHMCS Usage-Based Billing Module

```php
<?php
/**
 * WHMCS Usage-Based Billing Module
 * 
 * Tracks resource usage (bandwidth, storage, API calls) and
 * generates usage-based invoices with tiered pricing.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function usagebilling_MetaData() {
    return array('DisplayName' => 'Usage Billing', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function usagebilling_ConfigArray() {
    return array('FriendlyName' => array('Type' => 'System', 'Value' => 'Usage Billing'),
        'EnableTracking' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable usage tracking'),
        'BillingCycle' => array('Type' => 'dropdown', 'Options' => array('monthly' => 'Monthly', 'quarterly' => 'Quarterly', 'annually' => 'Annually'), 'Default' => 'monthly', 'Description' => 'Billing cycle'),
        'GenerateInvoice' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Auto-generate invoices'),
        'OveragePricing' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Charge overages'));
}

function usagebilling_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_usagebilling_resources', "
            CREATE TABLE `mod_usagebilling_resources` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `service_id` INT NOT NULL,
                `resource_type` VARCHAR(50) NOT NULL,
                `included_amount` DECIMAL(15,2) NOT NULL DEFAULT 0,
                `current_usage` DECIMAL(15,2) DEFAULT 0,
                `last_updated` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_service_resource` (`service_id`, `resource_type`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_usagebilling_tiers', "
            CREATE TABLE `mod_usagebilling_tiers` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `product_id` INT NOT NULL,
                `resource_type` VARCHAR(50) NOT NULL,
                `min_usage` DECIMAL(15,2) NOT NULL,
                `max_usage` DECIMAL(15,2) NULL,
                `unit_price` DECIMAL(10,4) NOT NULL,
                `tier_order` INT DEFAULT 0,
                INDEX `idx_product_resource` (`product_id`, `resource_type`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_usagebilling_records', "
            CREATE TABLE `mod_usagebilling_records` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `service_id` INT NOT NULL,
                `resource_type` VARCHAR(50) NOT NULL,
                `amount` DECIMAL(15,2) NOT NULL,
                `recorded_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_service_date` (`service_id`, `recorded_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Usage Billing module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function usagebilling_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function usagebilling_SetupResource($serviceId, $resourceType, $includedAmount, $productId = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_usagebilling_resources')->updateOrInsert(
            array('service_id' => $serviceId, 'resource_type' => $resourceType),
            array('included_amount' => $includedAmount, 'current_usage' => 0)
        );
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function usagebilling_RecordUsage($serviceId, $resourceType, $amount) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_usagebilling_records')->insert(array('service_id' => $serviceId, 'resource_type' => $resourceType, 'amount' => $amount));
        Capsule::table('mod_usagebilling_resources')->where('service_id', $serviceId)->where('resource_type', $resourceType)->increment('current_usage', $amount);
        Capsule::table('mod_usagebilling_resources')->where('service_id' => $serviceId)->where('resource_type' => $resourceType)->update(array('last_updated' => date('Y-m-d H:i:s')));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function usagebilling_GetUsage($serviceId, $resourceType = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_usagebilling_resources')->where('service_id', $serviceId);
    if ($resourceType) { $query->where('resource_type', $resourceType); }
    return $query->get();
}

function usagebilling_GetUsageHistory($serviceId, $resourceType, $days = 30) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $since = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    return Capsule::table('mod_usagebilling_records')->where('service_id', $serviceId)->where('resource_type', $resourceType)->where('recorded_at', '>=', $since)->orderBy('recorded_at', 'desc')->get();
}

function usagebilling_SetPricingTier($productId, $resourceType, $tiers) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_usagebilling_tiers')->where('product_id', $productId)->where('resource_type', $resourceType)->delete();
        foreach ($tiers as $index => $tier) {
            Capsule::table('mod_usagebilling_tiers')->insert(array('product_id' => $productId, 'resource_type' => $resourceType, 'min_usage' => $tier['min'], 'max_usage' => $tier['max'] ?? null, 'unit_price' => $tier['price'], 'tier_order' => $index));
        }
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function usagebilling_CalculateCost($serviceId, $resourceType) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $resource = Capsule::table('mod_usagebilling_resources')->where('service_id', $serviceId)->where('resource_type', $resourceType)->first();
    if (!$resource) return array('success' => false, 'error' => 'Resource not found');
    $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
    $tiers = Capsule::table('mod_usagebilling_tiers')->where('product_id', $service->packageid)->where('resource_type', $resourceType)->orderBy('tier_order', 'asc')->get();
    $usage = (float)$resource->current_usage;
    $included = (float)$resource->included_amount;
    $billableUsage = max(0, $usage - $included);
    $totalCost = 0;
    $remaining = $billableUsage;
    foreach ($tiers as $tier) {
        $tierMax = $tier->max_usage ? (float)$tier->max_usage : PHP_FLOAT_MAX;
        $tierMin = (float)$tier->min_usage;
        $tierSize = $tierMax - $tierMin;
        if ($remaining <= 0) break;
        $inThisTier = min($remaining, $tierSize);
        $totalCost += $inThisTier * (float)$tier->unit_price;
        $remaining -= $inThisTier;
    }
    return array('success' => true, 'usage' => $usage, 'included' => $included, 'billable_usage' => $billableUsage, 'total_cost' => round($totalCost, 2));
}

function usagebilling_GenerateInvoice($serviceId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $resources = usagebilling_GetUsage($serviceId);
    $items = array();
    $total = 0;
    foreach ($resources as $resource) {
        $cost = usagebilling_CalculateCost($serviceId, $resource->resource_type);
        if ($cost['billable_usage'] > 0) {
            $items[] = array('description' => ucfirst($resource->resource_type) . ' Usage', 'amount' => $cost['total_cost']);
            $total += $cost['total_cost'];
        }
    }
    return array('items' => $items, 'total' => round($total, 2));
}

function usagebilling_ResetUsage($serviceId, $resourceType = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_usagebilling_resources')->where('service_id', $serviceId);
    if ($resourceType) { $query->where('resource_type', $resourceType); }
    $query->update(array('current_usage' => 0, 'last_updated' => date('Y-m-d H:i:s')));
}
