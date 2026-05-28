# WHMCS Cost Calculator Module

```php
<?php
/**
 * WHMCS Cost Calculator Module
 * 
 * Cloud cost calculator with pricing models,
 * projections, and cost analytics.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function costcalculator_MetaData() {
    return array('DisplayName' => 'Cost Calculator', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function costcalculator_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Cost Calculator'),
        'DefaultCurrency' => array('Type' => 'dropdown', 'Options' => 'USD,EUR,GBP,CAD,AUD,JPY', 'Default' => 'USD', 'Description' => 'Default currency'),
        'TaxRate' => array('Type' => 'text', 'Size' => '10', 'Default' => '0', 'Description' => 'Tax rate (%)'),
        'EnableTieredPricing' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable tiered pricing'),
        'EnableBudgetTracking' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable budget tracking'),
        'BudgetAlertThreshold' => array('Type' => 'text', 'Size' => '10', 'Default' => '80', 'Description' => 'Alert threshold (%)'),
        'MinimumCharge' => array('Type' => 'text', 'Size' => '15', 'Default' => '0.00', 'Description' => 'Minimum invoice amount'),
        'ShowHiddenResources' => array('Type' => 'yesno', 'Default' => 'no', 'Description' => 'Show internal resources')
    );
}

function costcalculator_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_costcalculator_rates', "
            CREATE TABLE `mod_costcalculator_rates` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `resource_name` VARCHAR(100) NOT NULL,
                `resource_category` VARCHAR(50) DEFAULT 'compute',
                `unit` VARCHAR(50) NOT NULL,
                `cost_per_unit` DECIMAL(20,6) NOT NULL,
                `model` VARCHAR(50) DEFAULT 'per_unit',
                `currency` VARCHAR(10) DEFAULT 'USD',
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_resource_model` (`resource_name`, `model`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_costcalculator_tiers', "
            CREATE TABLE `mod_costcalculator_tiers` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `resource_name` VARCHAR(100) NOT NULL,
                `tier_name` VARCHAR(100) NOT NULL,
                `min_value` DECIMAL(20,6) NOT NULL DEFAULT 0,
                `max_value` DECIMAL(20,6) NOT NULL,
                `cost_per_unit` DECIMAL(20,6) NOT NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_costcalculator_discounts', "
            CREATE TABLE `mod_costcalculator_discounts` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `rule_name` VARCHAR(255) NOT NULL,
                `resource_name` VARCHAR(100) NULL,
                `user_id` INT NULL,
                `min_quantity` DECIMAL(20,6) DEFAULT 0,
                `discount_percent` DECIMAL(10,2) DEFAULT 0,
                `discount_amount` DECIMAL(20,6) DEFAULT 0,
                `start_date` DATE NULL,
                `end_date` DATE NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_resource` (`resource_name`),
                INDEX `idx_user` (`user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_costcalculator_budgets', "
            CREATE TABLE `mod_costcalculator_budgets` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT UNIQUE NOT NULL,
                `budget_amount` DECIMAL(20,6) NOT NULL,
                `period` VARCHAR(20) DEFAULT 'monthly',
                `alert_threshold` INT DEFAULT 80,
                `alert_sent` TINYINT(1) DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_user` (`user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_costcalculator_history', "
            CREATE TABLE `mod_costcalculator_history` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `service_id` INT NULL,
                `cost_date` DATE NOT NULL,
                `resource_name` VARCHAR(100) NOT NULL,
                `quantity` DECIMAL(20,6) NOT NULL,
                `unit_cost` DECIMAL(20,6) NOT NULL,
                `total_cost` DECIMAL(20,6) NOT NULL,
                `discount_applied` DECIMAL(20,6) DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_user_date` (`user_id`, `cost_date`),
                INDEX `idx_service_date` (`service_id`, `cost_date`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_costcalculator_formulas', "
            CREATE TABLE `mod_costcalculator_formulas` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `formula_name` VARCHAR(100) NOT NULL,
                `formula_expression` TEXT NOT NULL,
                `description` TEXT NULL,
                `variables` JSON NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        // Insert default rates
        $defaultRates = array(
            array('name' => 'cpu_cores', 'category' => 'compute', 'unit' => 'core', 'cost' => 0.05, 'model' => 'per_hour'),
            array('name' => 'ram_gb', 'category' => 'compute', 'unit' => 'GB', 'cost' => 0.02, 'model' => 'per_hour'),
            array('name' => 'disk_gb', 'category' => 'storage', 'unit' => 'GB', 'cost' => 0.10, 'model' => 'per_gb'),
            array('name' => 'bandwidth_gb', 'category' => 'network', 'unit' => 'GB', 'cost' => 0.09, 'model' => 'per_gb'),
            array('name' => 'api_calls', 'category' => 'api', 'unit' => 'call', 'cost' => 0.000001, 'model' => 'per_api_call'),
            array('name' => 'ip_addresses', 'category' => 'network', 'unit' => 'ip', 'cost' => 3.00, 'model' => 'per_hour'),
            array('name' => 'ssl_certs', 'category' => 'network', 'unit' => 'cert', 'cost' => 10.00, 'model' => 'per_month')
        );
        foreach ($defaultRates as $rate) {
            Capsule::table('mod_costcalculator_rates')->insert(array(
                'resource_name' => $rate['name'], 'resource_category' => $rate['category'],
                'unit' => $rate['unit'], 'cost_per_unit' => $rate['cost'], 'model' => $rate['model']
            ));
        }
        return array('status' => 'success', 'description' => 'Cost Calculator module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function costcalculator_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function costcalculator_AddRate($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $rateId = Capsule::table('mod_costcalculator_rates')->insertGetId(array(
            'resource_name' => $data['resource_name'], 'resource_category' => $data['resource_category'] ?? 'custom',
            'unit' => $data['unit'], 'cost_per_unit' => $data['cost_per_unit'],
            'model' => $data['model'] ?? 'per_unit', 'currency' => $data['currency'] ?? 'USD'
        ));
        return array('success' => true, 'rate_id' => $rateId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function costcalculator_GetRateCard() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_costcalculator_rates')->where('is_active', 1)->get();
}

function costcalculator_UpdateRate($rateId, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $update = array_filter(array(
            'cost_per_unit' => $data['cost_per_unit'] ?? null,
            'unit' => $data['unit'] ?? null,
            'model' => $data['model'] ?? null,
            'is_active' => isset($data['is_active']) ? (int)$data['is_active'] : null
        ), function($v) { return $v !== null; });
        Capsule::table('mod_costcalculator_rates')->where('id', $rateId)->update($update);
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function costcalculator_CreateTier($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $tierId = Capsule::table('mod_costcalculator_tiers')->insertGetId(array(
            'resource_name' => $data['resource_name'], 'tier_name' => $data['tier_name'],
            'min_value' => $data['min_value'] ?? 0, 'max_value' => $data['max_value'],
            'cost_per_unit' => $data['cost_per_unit']
        ));
        return array('success' => true, 'tier_id' => $tierId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function costcalculator_CalculateCost($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $config = costcalculator_GetConfig();
    $hoursInPeriod = costcalculator_GetHoursInPeriod($data['period'] ?? 'monthly');
    $breakdown = array();
    $subtotal = 0;
    foreach ($data['resources'] as $resource => $quantity) {
        $rate = Capsule::table('mod_costcalculator_rates')->where('resource_name', $resource)->where('is_active', 1)->first();
        if (!$rate) { continue; }
        $unitCost = $rate->cost_per_unit;
        if ($config['EnableTieredPricing'] === 'on') {
            $unitCost = costcalculator_GetTieredCost($resource, $quantity);
        }
        $resourceCost = $unitCost * $quantity;
        if ($rate->model === 'per_hour') { $resourceCost *= $hoursInPeriod; }
        $breakdown[$resource] = array('quantity' => $quantity, 'unit_cost' => $unitCost, 'model' => $rate->model, 'cost' => $resourceCost);
        $subtotal += $resourceCost;
    }
    $discounts = costcalculator_CalculateDiscounts($data['user_id'] ?? null, $breakdown);
    $totalDiscount = array_sum(array_column($discounts, 'amount'));
    $afterDiscount = $subtotal - $totalDiscount;
    $tax = ($config['TaxRate'] ?? 0) > 0 ? $afterDiscount * ($config['TaxRate'] / 100) : 0;
    $total = max($config['MinimumCharge'] ?? 0, $afterDiscount + $tax);
    return array('breakdown' => $breakdown, 'subtotal' => $subtotal, 'discounts' => $discounts, 'discount_total' => $totalDiscount, 'tax' => $tax, 'total' => $total, 'currency' => $config['DefaultCurrency'] ?? 'USD');
}

function costcalculator_CalculateServiceCost($serviceId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
    if (!$service) { return array('success' => false, 'error' => 'Service not found'); }
    $configData = json_decode($service->config_data ?? '{}', empty($service->config_data2) ? '{}' : array());
    $resourceConfig = !empty($service->config_data2) ? json_decode($service->config_data2, true) : array();
    $resources = array_merge($configData, $resourceConfig);
    return costcalculator_CalculateCost(array('resources' => $resources, 'user_id' => $service->user_id, 'service_id' => $serviceId));
}

function costcalculator_CalculateFormula($expression, $variables) {
    $safeExpression = preg_replace('/[^a-zA-Z0-9_+\-*\/\(\)\.\s]/', '', $expression);
    $safeExpression = str_replace(array_keys($variables), array_map(function($v) { return '(' . $v . ')'; }, $variables), $safeExpression);
    foreach ($variables as $name => $value) { ${$name} = $value; }
    try {
        $result = eval('return ' . $safeExpression . ';');
        return array('success' => true, 'result' => $result);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function costcalculator_GetTieredCost($resourceName, $quantity) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $tiers = Capsule::table('mod_costcalculator_tiers')->where('resource_name', $resourceName)->where('is_active', 1)->orderBy('min_value', 'asc')->get();
    if (empty($tiers)) {
        $rate = Capsule::table('mod_costcalculator_rates')->where('resource_name', $resourceName)->first();
        return $rate ? $rate->cost_per_unit : 0;
    }
    $totalCost = 0;
    $remaining = $quantity;
    foreach ($tiers as $tier) {
        if ($remaining <= 0) { break; }
        $tierRange = $tier->max_value - $tier->min_value;
        $applicableQty = min($remaining, $tierRange);
        $totalCost += $applicableQty * $tier->cost_per_unit;
        $remaining -= $applicableQty;
    }
    return $quantity > 0 ? $totalCost / $quantity : 0;
}

function costcalculator_CreateDiscount($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $discountId = Capsule::table('mod_costcalculator_discounts')->insertGetId(array(
            'rule_name' => $data['rule_name'], 'resource_name' => $data['resource_name'] ?? null,
            'user_id' => $data['user_id'] ?? null, 'min_quantity' => $data['min_quantity'] ?? 0,
            'discount_percent' => $data['discount_percent'] ?? 0, 'discount_amount' => $data['discount_amount'] ?? 0,
            'start_date' => $data['start_date'] ?? null, 'end_date' => $data['end_date'] ?? null
        ));
        return array('success' => true, 'discount_id' => $discountId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function costcalculator_ApplyDiscount($userId, $resourceName, $quantity) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $discount = Capsule::table('mod_costcalculator_discounts')->where('resource_name', $resourceName)
        ->where(function($q) { $q->whereNull('user_id')->orWhere('user_id', $userId); })
        ->where('min_quantity', '<=', $quantity)->where('is_active', 1)
        ->where(function($q) { $q->whereNull('start_date')->orWhere('start_date', '<=', date('Y-m-d')); })
        ->where(function($q) { $q->whereNull('end_date')->orWhere('end_date', '>=', date('Y-m-d')); })
        ->orderBy('discount_percent', 'desc')->first();
    if (!$discount) { return array('applied' => false, 'discount' => 0); }
    if ($discount->discount_percent > 0) { return array('applied' => true, 'type' => 'percent', 'value' => $discount->discount_percent); }
    return array('applied' => true, 'type' => 'fixed', 'value' => $discount->discount_amount);
}

function costcalculator_SetBudget($userId, $amount, $period = 'monthly', $alertThreshold = 80) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_costcalculator_budgets')->updateOrInsert(
            array('user_id' => $userId),
            array('budget_amount' => $amount, 'period' => $period, 'alert_threshold' => $alertThreshold, 'alert_sent' => 0)
        );
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function costcalculator_CheckBudget($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $budget = Capsule::table('mod_costcalculator_budgets')->where('user_id', $userId)->first();
    if (!$budget) { return array('has_budget' => false); }
    $startDate = costcalculator_GetBudgetPeriodStart($budget->period);
    $spent = Capsule::table('mod_costcalculator_history')->where('user_id', $userId)->where('cost_date', '>=', $startDate)->sum('total_cost');
    $remaining = $budget->budget_amount - $spent;
    $percentUsed = $budget->budget_amount > 0 ? ($spent / $budget->budget_amount) * 100 : 0;
    return array('has_budget' => true, 'budget_amount' => $budget->budget_amount, 'spent' => $spent, 'remaining' => max(0, $remaining), 'percent_used' => round($percentUsed, 2), 'over_budget' => $spent > $budget->budget_amount, 'period' => $budget->period);
}

function costcalculator_GetProjection($serviceIdOrUserId, $days = 365) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $isService = is_int($serviceIdOrUserId);
    $query = Capsule::table('mod_costcalculator_history');
    if ($isService) { $query->where('service_id', $serviceIdOrUserId); } else { $query->where('user_id', $serviceIdOrUserId); }
    $history = $query->where('cost_date', '>=', date('Y-m-d', strtotime("-30 days")))->get();
    if (empty($history)) { return array('daily' => 0, 'weekly' => 0, 'monthly' => 0, 'yearly' => 0, 'confidence' => 'low'); }
    $avgDaily = array_sum(array_column($history, 'total_cost')) / 30;
    return array('daily' => $avgDaily, 'weekly' => $avgDaily * 7, 'monthly' => $avgDaily * 30, 'yearly' => $avgDaily * 365, 'confidence' => count($history) >= 30 ? 'high' : 'medium');
}

function costcalculator_GetCostBreakdown($userId, $month = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    if (!$month) { $month = date('Y-m'); }
    $startDate = $month . '-01';
    $endDate = date('Y-m-d', strtotime($startDate . ' +1 month -1 day'));
    $breakdown = Capsule::table('mod_costcalculator_history')->where('user_id', $userId)
        ->whereBetween('cost_date', array($startDate, $endDate))->selectRaw("resource_name, SUM(quantity) as total_quantity, SUM(total_cost) as total_cost")->groupBy('resource_name')->get();
    return $breakdown;
}

function costcalculator_GetHoursInPeriod($period) {
    switch ($period) {
        case 'hourly': return 1;
        case 'daily': return 24;
        case 'weekly': return 168;
        case 'monthly': return 730;
        case 'yearly': return 8760;
        default: return 730;
    }
}

function costcalculator_GetBudgetPeriodStart($period) {
    $today = new DateTime();
    switch ($period) {
        case 'daily': return $today->format('Y-m-d');
        case 'weekly': return $today->modify('monday this week')->format('Y-m-d');
        case 'monthly': return $today->format('Y-m-01');
        case 'yearly': return $today->format('Y-01-01');
        default: return $today->format('Y-m-01');
    }
}

function costcalculator_CalculateDiscounts($userId, $breakdown) {
    $discounts = array();
    foreach ($breakdown as $resource => $data) {
        $discount = costcalculator_ApplyDiscount($userId, $resource, $data['quantity']);
        if ($discount['applied']) {
            $amount = $discount['type'] === 'percent' ? $data['cost'] * ($discount['value'] / 100) : $discount['value'];
            $discounts[] = array('resource' => $resource, 'type' => $discount['type'], 'value' => $discount['value'], 'amount' => $amount);
        }
    }
    return $discounts;
}

function costcalculator_GetAnalytics($userId, $days = 90) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $since = date('Y-m-d', strtotime("-{$days} days"));
    $totalSpent = Capsule::table('mod_costcalculator_history')->where('user_id', $userId)->where('cost_date', '>=', $since)->sum('total_cost');
    $byResource = Capsule::table('mod_costcalculator_history')->where('user_id', $userId)->where('cost_date', '>=', $since)->selectRaw("resource_name, SUM(total_cost) as cost")->groupBy('resource_name')->get();
    $byDate = Capsule::table('mod_costcalculator_history')->where('user_id', $userId)->where('cost_date', '>=', $since)->selectRaw("cost_date, SUM(total_cost) as cost")->groupBy('cost_date')->orderBy('cost_date', 'asc')->get();
    return array('total_spent' => $totalSpent, 'by_resource' => $byResource, 'by_date' => $byDate, 'period_days' => $days, 'currency' => costcalculator_GetConfig()['DefaultCurrency'] ?? 'USD');
}

function costcalculator_GetConfig() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $settings = Capsule::table('tbladdonmodules')->where('module', 'costcalculator')->get();
    $config = array();
    foreach ($settings as $setting) { $config[$setting['setting']] = $setting['value']; }
    return $config;
}
```
