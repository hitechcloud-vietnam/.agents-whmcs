# WHMCS Subscription Management Module

```php
<?php
/**
 * WHMCS Subscription Management Module
 * 
 * This module provides advanced subscription management with configurable billing cycles,
 * proration support, and automated renewal handling.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

// Prevent direct access
if (!defined("WHMCS")) {
    die("Direct access prohibited");
}

/**
 * Module meta data
 */
function subscriptionmanagement_MetaData()
{
    return array(
        'DisplayName' => 'Subscription Management',
        'APIVersion' => '1.1',
        'RequiresServer' => false,
    );
}

/**
 * Module configuration
 */
function subscriptionmanagement_ConfigArray()
{
    return array(
        'FriendlyName' => array(
            'Type' => 'System',
            'Value' => 'Subscription Management',
        ),
        'DefaultBillingCycle' => array(
            'Type' => 'dropdown',
            'Options' => array(
                'monthly' => 'Monthly',
                'quarterly' => 'Quarterly',
                'semiannually' => 'Semi-Annually',
                'annually' => 'Annually',
                'biennially' => 'Biennially',
                'triennially' => 'Triennially',
            ),
            'Default' => 'monthly',
            'Description' => 'Default billing cycle for new subscriptions',
        ),
        'EnableProration' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable proration when changing subscription plans',
        ),
        'ProrationMethod' => array(
            'Type' => 'dropdown',
            'Options' => array(
                'credit' => 'Credit Remaining Time',
                'charge' => 'Charge Proportional Amount',
                'both' => 'Credit and Charge',
            ),
            'Default' => 'credit',
            'Description' => 'How to handle proration differences',
        ),
        'AutoRenew' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Automatically attempt renewal before expiration',
        ),
        'RenewalNoticeDays' => array(
            'Type' => 'text',
            'Size' => '10',
            'Default' => '7,3,1',
            'Description' => 'Days before expiration to send renewal notices (comma-separated)',
        ),
        'GracePeriodDays' => array(
            'Type' => 'text',
            'Size' => '10',
            'Default' => '3',
            'Description' => 'Grace period after expiration before suspension',
        ),
    );
}

/**
 * Activate module - create database tables
 */
function subscriptionmanagement_activate()
{
    try {
        // Create subscription management table
        if (!function_exists('createTable')) {
            require_once dirname(__FILE__) . '/../../includes/modulefunctions.php';
        }
        
        $tableName = 'mod_subscription_management';
        
        $schema = "
            CREATE TABLE `{$tableName}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `rel_id` INT NOT NULL,
                `subscription_id` VARCHAR(100) UNIQUE,
                `plan_id` VARCHAR(100) NOT NULL,
                `billing_cycle` VARCHAR(50) NOT NULL DEFAULT 'monthly',
                `amount` DECIMAL(10,2) NOT NULL,
                `currency` VARCHAR(10) DEFAULT 'USD',
                `status` ENUM('active', 'paused', 'cancelled', 'expired', 'trial') DEFAULT 'active',
                `start_date` DATETIME NOT NULL,
                `next_billing_date` DATETIME NOT NULL,
                `trial_end_date` DATETIME NULL,
                `cancelled_at` DATETIME NULL,
                `proration_credit` DECIMAL(10,2) DEFAULT 0.00,
                `custom_fields` TEXT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                INDEX `idx_user_id` (`user_id`),
                INDEX `idx_rel_id` (`rel_id`),
                INDEX `idx_next_billing` (`next_billing_date`),
                INDEX `idx_status` (`status`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        
        createTable($tableName, $schema);
        
        // Create subscription plans table
        $plansTable = 'mod_subscription_plans';
        $plansSchema = "
            CREATE TABLE `{$plansTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `plan_name` VARCHAR(255) NOT NULL,
                `plan_key` VARCHAR(100) UNIQUE NOT NULL,
                `description` TEXT NULL,
                `billing_cycles` JSON NOT NULL,
                `pricing` JSON NOT NULL,
                `features` JSON NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `sort_order` INT DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        
        createTable($plansTable, $plansSchema);
        
        // Create billing events table
        $eventsTable = 'mod_subscription_events';
        $eventsSchema = "
            CREATE TABLE `{$eventsTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `subscription_id` INT NOT NULL,
                `event_type` VARCHAR(50) NOT NULL,
                `event_date` DATETIME NOT NULL,
                `amount` DECIMAL(10,2) NULL,
                `status` VARCHAR(50) DEFAULT 'pending',
                `invoice_id` INT NULL,
                `details` TEXT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_subscription` (`subscription_id`),
                INDEX `idx_event_date` (`event_date`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        
        createTable($eventsTable, $eventsSchema);
        
        return array(
            'status' => 'success',
            'description' => 'Subscription Management module activated successfully. Database tables created.',
        );
    } catch (\Exception $e) {
        return array(
            'status' => 'error',
            'description' => 'Failed to activate module: ' . $e->getMessage(),
        );
    }
}

/**
 * Deactivate module
 */
function subscriptionmanagement_deactivate()
{
    return array(
        'status' => 'success',
        'description' => 'Module deactivated. Data preserved.',
    );
}

/**
 * Upgrade module
 */
function subscriptionmanagement_upgrade($vars)
{
    $version = $vars['version'];
    
    if ($version < '1.1.0') {
        // Add new fields for v1.1.0
        try {
            $query = "ALTER TABLE `mod_subscription_management` 
                      ADD COLUMN `external_subscription_id` VARCHAR(255) NULL AFTER `subscription_id`";
            full_query($query);
        } catch (\Exception $e) {
            logActivity("Subscription Management upgrade error: " . $e->getMessage());
        }
    }
}

/**
 * Create a new subscription
 * 
 * @param int $userId Client user ID
 * @param int $relId Service/Product ID
 * @param string $planId Plan identifier
 * @param array $options Additional options
 * @return array Result with subscription ID or error
 */
function subscriptionmanagement_CreateSubscription($userId, $relId, $planId, $options = array())
{
    if (!function_exists(' Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        // Get plan details
        $plan = Capsule::table('mod_subscription_plans')
            ->where('plan_key', $planId)
            ->where('is_active', 1)
            ->first();
        
        if (!$plan) {
            return array('success' => false, 'error' => 'Invalid plan selected');
        }
        
        $billingCycle = isset($options['billing_cycle']) ? $options['billing_cycle'] : 'monthly';
        $pricing = json_decode($plan->pricing, true);
        
        if (!isset($pricing[$billingCycle])) {
            return array('success' => false, 'error' => 'Invalid billing cycle for this plan');
        }
        
        $amount = $pricing[$billingCycle];
        $startDate = new DateTime();
        $nextBillingDate = subscriptionmanagement_CalculateNextBillingDate($startDate, $billingCycle);
        
        // Generate unique subscription ID
        $subscriptionId = 'SUB-' . strtoupper(uniqid());
        
        $subscription = array(
            'user_id' => $userId,
            'rel_id' => $relId,
            'subscription_id' => $subscriptionId,
            'plan_id' => $planId,
            'billing_cycle' => $billingCycle,
            'amount' => $amount,
            'currency' => isset($options['currency']) ? $options['currency'] : 'USD',
            'status' => 'active',
            'start_date' => $startDate->format('Y-m-d H:i:s'),
            'next_billing_date' => $nextBillingDate->format('Y-m-d H:i:s'),
            'custom_fields' => isset($options['custom_fields']) ? json_encode($options['custom_fields']) : null,
        );
        
        Capsule::table('mod_subscription_management')->insert($subscription);
        $subscriptionDbId = Capsule::connection()->getPdo()->lastInsertId();
        
        // Log creation event
        subscriptionmanagement_LogEvent($subscriptionDbId, 'created', $startDate, $amount);
        
        return array(
            'success' => true,
            'subscription_id' => $subscriptionId,
            'db_id' => $subscriptionDbId,
            'next_billing_date' => $nextBillingDate->format('Y-m-d H:i:s'),
        );
        
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Calculate next billing date based on cycle
 * 
 * @param DateTime $startDate Start date
 * @param string $billingCycle Billing cycle
 * @return DateTime Next billing date
 */
function subscriptionmanagement_CalculateNextBillingDate($startDate, $billingCycle)
{
    $nextDate = clone $startDate;
    
    switch ($billingCycle) {
        case 'monthly':
            $nextDate->modify('+1 month');
            break;
        case 'quarterly':
            $nextDate->modify('+3 months');
            break;
        case 'semiannually':
            $nextDate->modify('+6 months');
            break;
        case 'annually':
            $nextDate->modify('+1 year');
            break;
        case 'biennially':
            $nextDate->modify('+2 years');
            break;
        case 'triennially':
            $nextDate->modify('+3 years');
            break;
        default:
            $nextDate->modify('+1 month');
    }
    
    return $nextDate;
}

/**
 * Change subscription plan
 * 
 * @param string $subscriptionId Subscription ID
 * @param string $newPlanId New plan ID
 * @param string $newCycle New billing cycle
 * @return array Result
 */
function subscriptionmanagement_ChangePlan($subscriptionId, $newPlanId, $newCycle = null)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $subscription = Capsule::table('mod_subscription_management')
            ->where('subscription_id', $subscriptionId)
            ->first();
        
        if (!$subscription) {
            return array('success' => false, 'error' => 'Subscription not found');
        }
        
        $newCycle = $newCycle ?: $subscription->billing_cycle;
        
        // Get new plan details
        $plan = Capsule::table('mod_subscription_plans')
            ->where('plan_key', $newPlanId)
            ->where('is_active', 1)
            ->first();
        
        if (!$plan) {
            return array('success' => false, 'error' => 'Invalid plan');
        }
        
        $newPricing = json_decode($plan->pricing, true);
        if (!isset($newPricing[$newCycle])) {
            return array('success' => false, 'error' => 'Invalid billing cycle for plan');
        }
        
        $newAmount = $newPricing[$newCycle];
        
        // Calculate proration if enabled
        $prorationAmount = 0;
        $now = new DateTime();
        $nextBilling = new DateTime($subscription->next_billing_date);
        $daysRemaining = $now->diff($nextBilling)->days;
        $totalDays = (new DateTime($subscription->start_date))->diff($nextBilling)->days ?: 30;
        
        if ($daysRemaining > 0 && $subscription->amount != $newAmount) {
            // Calculate credit for remaining time on old plan
            $oldDailyRate = $subscription->amount / $totalDays;
            $credit = $oldDailyRate * $daysRemaining;
            
            // Calculate charge for remaining time on new plan
            $newDailyRate = $newAmount / $totalDays;
            $charge = $newDailyRate * $daysRemaining;
            
            $prorationAmount = $charge - $credit;
        }
        
        // Update subscription
        Capsule::table('mod_subscription_management')
            ->where('id', $subscription->id)
            ->update(array(
                'plan_id' => $newPlanId,
                'billing_cycle' => $newCycle,
                'amount' => $newAmount,
                'proration_credit' => $subscription->proration_credit + $prorationAmount,
                'updated_at' => date('Y-m-d H:i:s'),
            ));
        
        // Log plan change
        subscriptionmanagement_LogEvent($subscription->id, 'plan_changed', $now, $prorationAmount, json_encode(array(
            'old_plan' => $subscription->plan_id,
            'new_plan' => $newPlanId,
            'old_amount' => $subscription->amount,
            'new_amount' => $newAmount,
        )));
        
        return array(
            'success' => true,
            'proration_amount' => $prorationAmount,
            'new_amount' => $newAmount,
            'effective_date' => $now->format('Y-m-d H:i:s'),
        );
        
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Pause subscription
 * 
 * @param string $subscriptionId Subscription ID
 * @param int $pauseDays Number of days to pause (null for indefinite)
 * @return array Result
 */
function subscriptionmanagement_PauseSubscription($subscriptionId, $pauseDays = null)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $subscription = Capsule::table('mod_subscription_management')
            ->where('subscription_id', $subscriptionId)
            ->first();
        
        if (!$subscription) {
            return array('success' => false, 'error' => 'Subscription not found');
        }
        
        if ($subscription->status !== 'active') {
            return array('success' => false, 'error' => 'Only active subscriptions can be paused');
        }
        
        $now = new DateTime();
        $resumeDate = null;
        
        if ($pauseDays) {
            $resumeDate = clone $now;
            $resumeDate->modify("+{$pauseDays} days");
            $resumeDate = $resumeDate->format('Y-m-d H:i:s');
        }
        
        Capsule::table('mod_subscription_management')
            ->where('id', $subscription->id)
            ->update(array(
                'status' => 'paused',
                'cancelled_at' => $resumeDate,
                'updated_at' => $now->format('Y-m-d H:i:s'),
            ));
        
        subscriptionmanagement_LogEvent($subscription->id, 'paused', $now, null, json_encode(array(
            'resume_date' => $resumeDate,
        )));
        
        return array('success' => true, 'resume_date' => $resumeDate);
        
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Resume paused subscription
 * 
 * @param string $subscriptionId Subscription ID
 * @return array Result
 */
function subscriptionmanagement_ResumeSubscription($subscriptionId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $subscription = Capsule::table('mod_subscription_management')
            ->where('subscription_id', $subscriptionId)
            ->first();
        
        if (!$subscription) {
            return array('success' => false, 'error' => 'Subscription not found');
        }
        
        if ($subscription->status !== 'paused') {
            return array('success' => false, 'error' => 'Only paused subscriptions can be resumed');
        }
        
        $now = new DateTime();
        
        // Recalculate next billing date
        $nextBilling = subscriptionmanagement_CalculateNextBillingDate($now, $subscription->billing_cycle);
        
        Capsule::table('mod_subscription_management')
            ->where('id', $subscription->id)
            ->update(array(
                'status' => 'active',
                'next_billing_date' => $nextBilling->format('Y-m-d H:i:s'),
                'cancelled_at' => null,
                'updated_at' => $now->format('Y-m-d H:i:s'),
            ));
        
        subscriptionmanagement_LogEvent($subscription->id, 'resumed', $now, null, json_encode(array(
            'new_billing_date' => $nextBilling->format('Y-m-d H:i:s'),
        )));
        
        return array('success' => true, 'next_billing_date' => $nextBilling->format('Y-m-d H:i:s'));
        
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Cancel subscription
 * 
 * @param string $subscriptionId Subscription ID
 * @param bool $immediate Cancel immediately or at period end
 * @param string $reason Cancellation reason
 * @return array Result
 */
function subscriptionmanagement_CancelSubscription($subscriptionId, $immediate = false, $reason = '')
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $subscription = Capsule::table('mod_subscription_management')
            ->where('subscription_id', $subscriptionId)
            ->first();
        
        if (!$subscription) {
            return array('success' => false, 'error' => 'Subscription not found');
        }
        
        $now = new DateTime();
        
        if ($immediate) {
            Capsule::table('mod_subscription_management')
                ->where('id', $subscription->id)
                ->update(array(
                    'status' => 'cancelled',
                    'cancelled_at' => $now->format('Y-m-d H:i:s'),
                    'updated_at' => $now->format('Y-m-d H:i:s'),
                ));
            
            $cancelledAt = $now->format('Y-m-d H:i:s');
        } else {
            $cancelledAt = $subscription->next_billing_date;
        }
        
        subscriptionmanagement_LogEvent($subscription->id, 'cancelled', $now, null, json_encode(array(
            'immediate' => $immediate,
            'cancelled_at' => $cancelledAt,
            'reason' => $reason,
        )));
        
        return array('success' => true, 'cancelled_at' => $cancelledAt);
        
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Process subscription renewal
 * 
 * @param int $subscriptionDbId Database ID of subscription
 * @return array Result with invoice ID if created
 */
function subscriptionmanagement_ProcessRenewal($subscriptionDbId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    if (!function_exists('createInvoices')) {
        require_once dirname(__FILE__) . '/../../includes/invoicefunctions.php';
    }
    
    try {
        $subscription = Capsule::table('mod_subscription_management')
            ->where('id', $subscriptionDbId)
            ->first();
        
        if (!$subscription) {
            return array('success' => false, 'error' => 'Subscription not found');
        }
        
        if ($subscription->status !== 'active') {
            return array('success' => false, 'error' => 'Subscription is not active');
        }
        
        $now = new DateTime();
        
        // Create invoice
        $invoiceId = createInvoices($subscription->user_id, true);
        
        if ($invoiceId) {
            // Update billing date
            $nextBilling = subscriptionmanagement_CalculateNextBillingDate($now, $subscription->billing_cycle);
            
            Capsule::table('mod_subscription_management')
                ->where('id', $subscription->id)
                ->update(array(
                    'next_billing_date' => $nextBilling->format('Y-m-d H:i:s'),
                    'proration_credit' => 0,
                    'updated_at' => $now->format('Y-m-d H:i:s'),
                ));
            
            subscriptionmanagement_LogEvent($subscription->id, 'renewed', $now, $subscription->amount, json_encode(array(
                'invoice_id' => $invoiceId,
            )));
            
            return array('success' => true, 'invoice_id' => $invoiceId);
        }
        
        return array('success' => false, 'error' => 'Failed to create invoice');
        
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Log subscription event
 * 
 * @param int $subscriptionId Database subscription ID
 * @param string $eventType Event type
 * @param DateTime $eventDate Event date
 * @param float $amount Amount
 * @param string $details Additional details JSON
 */
function subscriptionmanagement_LogEvent($subscriptionId, $eventType, $eventDate, $amount = null, $details = null)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    Capsule::table('mod_subscription_events')->insert(array(
        'subscription_id' => $subscriptionId,
        'event_type' => $eventType,
        'event_date' => $eventDate->format('Y-m-d H:i:s'),
        'amount' => $amount,
        'details' => $details,
    ));
}

/**
 * Get subscription details
 * 
 * @param string $subscriptionId Subscription ID
 * @return array|null Subscription data
 */
function subscriptionmanagement_GetSubscription($subscriptionId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    return Capsule::table('mod_subscription_management')
        ->where('subscription_id', $subscriptionId)
        ->first();
}

/**
 * Get subscription by user ID
 * 
 * @param int $userId Client user ID
 * @param array $filters Optional filters
 * @return array Subscriptions
 */
function subscriptionmanagement_GetSubscriptionsByUser($userId, $filters = array())
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $query = Capsule::table('mod_subscription_management')
        ->where('user_id', $userId);
    
    if (isset($filters['status'])) {
        $query->where('status', $filters['status']);
    }
    
    if (isset($filters['plan_id'])) {
        $query->where('plan_id', $filters['plan_id']);
    }
    
    return $query->get();
}

/**
 * Get subscriptions due for renewal
 * 
 * @param int $daysAhead Days ahead to look for renewals
 * @return array Subscriptions
 */
function subscriptionmanagement_GetDueRenewals($daysAhead = 3)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $targetDate = date('Y-m-d H:i:s', strtotime("+{$daysAhead} days"));
    $now = date('Y-m-d H:i:s');
    
    return Capsule::table('mod_subscription_management')
        ->where('status', 'active')
        ->where('next_billing_date', '<=', $targetDate)
        ->where('next_billing_date', '>=', $now)
        ->get();
}

/**
 * Create subscription plan
 * 
 * @param array $planData Plan data
 * @return array Result
 */
function subscriptionmanagement_CreatePlan($planData)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $planKey = preg_replace('/[^a-zA-Z0-9_-]/', '', strtolower($planData['plan_name']));
        $planKey .= '-' . substr(md5(uniqid()), 0, 6);
        
        $data = array(
            'plan_name' => $planData['plan_name'],
            'plan_key' => $planKey,
            'description' => isset($planData['description']) ? $planData['description'] : '',
            'billing_cycles' => json_encode($planData['billing_cycles']),
            'pricing' => json_encode($planData['pricing']),
            'features' => isset($planData['features']) ? json_encode($planData['features']) : null,
            'sort_order' => isset($planData['sort_order']) ? $planData['sort_order'] : 0,
        );
        
        Capsule::table('mod_subscription_plans')->insert($data);
        
        return array('success' => true, 'plan_key' => $planKey);
        
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Get all subscription plans
 * 
 * @param bool $activeOnly Only active plans
 * @return array Plans
 */
function subscriptionmanagement_GetPlans($activeOnly = true)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $query = Capsule::table('mod_subscription_plans');
    
    if ($activeOnly) {
        $query->where('is_active', 1);
    }
    
    $plans = $query->orderBy('sort_order', 'asc')->get();
    
    foreach ($plans as &$plan) {
        $plan->billing_cycles = json_decode($plan->billing_cycles, true);
        $plan->pricing = json_decode($plan->pricing, true);
        $plan->features = json_decode($plan->features, true);
    }
    
    return $plans;
}

/**
 * Update subscription plan
 * 
 * @param string $planKey Plan key
 * @param array $planData Updated data
 * @return array Result
 */
function subscriptionmanagement_UpdatePlan($planKey, $planData)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $updateData = array_filter(array(
            'plan_name' => isset($planData['plan_name']) ? $planData['plan_name'] : null,
            'description' => isset($planData['description']) ? $planData['description'] : null,
            'billing_cycles' => isset($planData['billing_cycles']) ? json_encode($planData['billing_cycles']) : null,
            'pricing' => isset($planData['pricing']) ? json_encode($planData['pricing']) : null,
            'features' => isset($planData['features']) ? json_encode($planData['features']) : null,
            'is_active' => isset($planData['is_active']) ? $planData['is_active'] : null,
            'sort_order' => isset($planData['sort_order']) ? $planData['sort_order'] : null,
        ), function($v) { return $v !== null; });
        
        if (empty($updateData)) {
            return array('success' => false, 'error' => 'No data to update');
        }
        
        $affected = Capsule::table('mod_subscription_plans')
            ->where('plan_key', $planKey)
            ->update($updateData);
        
        return array('success' => $affected > 0, 'affected' => $affected);
        
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Delete subscription plan
 * 
 * @param string $planKey Plan key
 * @return array Result
 */
function subscriptionmanagement_DeletePlan($planKey)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        // Check if any active subscriptions use this plan
        $activeCount = Capsule::table('mod_subscription_management')
            ->where('plan_id', $planKey)
            ->whereIn('status', array('active', 'trial', 'paused'))
            ->count();
        
        if ($activeCount > 0) {
            // Soft delete - just deactivate
            Capsule::table('mod_subscription_plans')
                ->where('plan_key', $planKey)
                ->update(array('is_active' => 0));
            
            return array(
                'success' => true,
                'deactivated' => true,
                'message' => "Plan has {$activeCount} active subscriptions. Deactivated instead of deleted.",
            );
        }
        
        Capsule::table('mod_subscription_plans')
            ->where('plan_key', $planKey)
            ->delete();
        
        return array('success' => true);
        
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Get subscription statistics
 * 
 * @param int $userId Optional user ID filter
 * @return array Statistics
 */
function subscriptionmanagement_GetStatistics($userId = null)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $query = Capsule::table('mod_subscription_management');
    
    if ($userId) {
        $query->where('user_id', $userId);
    }
    
    $total = $query->count();
    $active = (clone $query)->where('status', 'active')->count();
    $paused = (clone $query)->where('status', 'paused')->count();
    $cancelled = (clone $query)->where('status', 'cancelled')->count();
    $expired = (clone $query)->where('status', 'expired')->count();
    
    $revenue = Capsule::table('mod_subscription_management')
        ->where('status', 'active');
    
    if ($userId) {
        $revenue->where('user_id', $userId);
    }
    
    $revenue = (clone $revenue)
        ->selectRaw('SUM(amount) as total')
        ->first()
        ->total ?? 0;
    
    return array(
        'total' => $total,
        'active' => $active,
        'paused' => $paused,
        'cancelled' => $cancelled,
        'expired' => $expired,
        'monthly_recurring_revenue' => (float) $revenue,
    );
}

/**
 * Get subscription events
 * 
 * @param string $subscriptionId Subscription ID
 * @param int $limit Result limit
 * @return array Events
 */
function subscriptionmanagement_GetEvents($subscriptionId, $limit = 50)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $subscription = Capsule::table('mod_subscription_management')
        ->where('subscription_id', $subscriptionId)
        ->first();
    
    if (!$subscription) {
        return array();
    }
    
    return Capsule::table('mod_subscription_events')
        ->where('subscription_id', $subscription->id)
        ->orderBy('created_at', 'desc')
        ->limit($limit)
        ->get();
}
