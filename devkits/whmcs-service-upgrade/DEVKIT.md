# WHMCS Service Upgrade Module

```php
<?php
/**
 * WHMCS Service Upgrade/Downgrade Module
 * 
 * Handles service plan changes with proration,
 * immediate/delayed activation, and billing adjustments.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function serviceupgrade_MetaData() {
    return array('DisplayName' => 'Service Upgrade', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function serviceupgrade_ConfigArray() {
    return array('FriendlyName' => array('Type' => 'System', 'Value' => 'Service Upgrade'),
        'EnableProration' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable proration'),
        'ProrationMethod' => array('Type' => 'dropdown', 'Options' => array('credit' => 'Credit Remaining', 'charge' => 'Charge Difference', 'both' => 'Both'), 'Default' => 'credit', 'Description' => 'Proration calculation method'),
        'ImmediateUpgrade' => array('Type' => 'yesno', 'Default' => 'no', 'Description' => 'Apply upgrade immediately'),
        'GenerateInvoice' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Generate invoice for upgrades'));
}

function serviceupgrade_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_serviceupgrade_requests', "
            CREATE TABLE `mod_serviceupgrade_requests` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `service_id` INT NOT NULL,
                `from_product_id` INT NOT NULL,
                `to_product_id` INT NOT NULL,
                `proration_amount` DECIMAL(10,2) DEFAULT 0.00,
                `billing_cycle` VARCHAR(50) NOT NULL,
                `activation_type` ENUM('immediate', 'end_of_period') DEFAULT 'end_of_period',
                `status` ENUM('pending', 'approved', 'processing', 'completed', 'cancelled') DEFAULT 'pending',
                `scheduled_date` DATE NULL,
                `processed_at` DATETIME NULL,
                `invoice_id` INT NULL,
                `requested_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_service_id` (`service_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Service Upgrade module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function serviceupgrade_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function serviceupgrade_CalculateProration($serviceId, $newProductId, $cycle = 'monthly') {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
    $oldProduct = Capsule::table('tblproducts')->where('id', $service->packageid)->first();
    $newProduct = Capsule::table('tblproducts')->where('id', $newProductId)->first();
    $cycleField = $cycle === 'monthly' ? 'monthly' : ($cycle === 'annually' ? 'yearly' : 'monthly');
    $oldPrice = $oldProduct->$cycleField ?? 0;
    $newPrice = $newProduct->$cycleField ?? 0;
    $nextDue = new DateTime($service->nextduedate);
    $now = new DateTime();
    $totalDays = $nextDue->diff(new DateTime($service->regdate))->days ?: 30;
    $remainingDays = $now->diff($nextDue)->days;
    $oldDailyRate = $oldPrice / max($totalDays, 1);
    $newDailyRate = $newPrice / max($totalDays, 1);
    $credit = $oldDailyRate * $remainingDays;
    $charge = $newDailyRate * $remainingDays;
    return array('credit' => round($credit, 2), 'charge' => round($charge, 2), 'difference' => round($charge - $credit, 2), 'old_price' => $oldPrice, 'new_price' => $newPrice, 'days_remaining' => $remainingDays);
}

function serviceupgrade_CreateRequest($serviceId, $newProductId, $options = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
        $proration = serviceupgrade_CalculateProration($serviceId, $newProductId, $options['cycle'] ?? 'monthly');
        $scheduledDate = null;
        if (($options['activation_type'] ?? 'end_of_period') === 'end_of_period') {
            $scheduledDate = $service->nextduedate;
        }
        Capsule::table('mod_serviceupgrade_requests')->insert(array('service_id' => $serviceId, 'from_product_id' => $service->packageid, 'to_product_id' => $newProductId, 'proration_amount' => $proration['difference'], 'billing_cycle' => $options['cycle'] ?? 'monthly', 'activation_type' => $options['activation_type'] ?? 'end_of_period', 'scheduled_date' => $scheduledDate));
        return array('success' => true, 'proration' => $proration);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function serviceupgrade_ProcessUpgrades() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $today = date('Y-m-d');
    $requests = Capsule::table('mod_serviceupgrade_requests')->where('status', 'approved')->where('scheduled_date', '<=', $today)->get();
    $processed = 0;
    foreach ($requests as $request) {
        serviceupgrade_ExecuteUpgrade($request->id);
        $processed++;
    }
    return $processed;
}

function serviceupgrade_ExecuteUpgrade($requestId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $request = Capsule::table('mod_serviceupgrade_requests')->where('id', $requestId)->first();
        Capsule::table('mod_serviceupgrade_requests')->where('id', $requestId)->update(array('status' => 'processing'));
        Capsule::table('tblhosting')->where('id', $request->service_id)->update(array('packageid' => $request->to_product_id));
        Capsule::table('mod_serviceupgrade_requests')->where('id', $requestId)->update(array('status' => 'completed', 'processed_at' => date('Y-m-d H:i:s')));
        return array('success' => true);
    } catch (\Exception $e) {
        Capsule::table('mod_serviceupgrade_requests')->where('id', $requestId)->update(array('status' => 'pending'));
        return array('success' => false, 'error' => $e->getMessage());
    }
}

function serviceupgrade_GetRequest($serviceId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_serviceupgrade_requests')->where('service_id', $serviceId)->orderBy('id', 'desc')->first();
}

function serviceupgrade_CancelRequest($requestId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_serviceupgrade_requests')->where('id', $requestId)->update(array('status' => 'cancelled'));
}
