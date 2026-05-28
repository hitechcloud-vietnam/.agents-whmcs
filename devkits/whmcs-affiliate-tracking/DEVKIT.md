# WHMCS Affiliate Tracking Module

```php
<?php
/**
 * WHMCS Affiliate Tracking Module
 * 
 * Advanced affiliate tracking with commission tiers,
 * referral tracking, and payout management.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function affiliatetracking_MetaData() {
    return array('DisplayName' => 'Affiliate Tracking', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function affiliatetracking_ConfigArray() {
    return array('FriendlyName' => array('Type' => 'System', 'Value' => 'Affiliate Tracking'),
        'DefaultCommission' => array('Type' => 'text', 'Size' => '10', 'Default' => '10', 'Description' => 'Default commission %'),
        'CookieDuration' => array('Type' => 'text', 'Size' => '10', 'Default' => '30', 'Description' => 'Cookie duration (days)'),
        'EnableTiers' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable multi-tier commissions'),
        'MinimumPayout' => array('Type' => 'text', 'Size' => '10', 'Default' => '50', 'Description' => 'Minimum payout amount'));
}

function affiliatetracking_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_affiliatetracking_affiliates', "
            CREATE TABLE `mod_affiliatetracking_affiliates` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL UNIQUE,
                `affiliate_code` VARCHAR(50) UNIQUE NOT NULL,
                `commission_rate` DECIMAL(5,2) DEFAULT 10.00,
                `tier` INT DEFAULT 1,
                `total_referred` INT DEFAULT 0,
                `total_earned` DECIMAL(10,2) DEFAULT 0.00,
                `total_paid` DECIMAL(10,2) DEFAULT 0.00,
                `pending_balance` DECIMAL(10,2) DEFAULT 0.00,
                `status` ENUM('active', 'suspended') DEFAULT 'active',
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_affiliatetracking_referrals', "
            CREATE TABLE `mod_affiliatetracking_referrals` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `affiliate_id` INT NOT NULL,
                `referred_user_id` INT NOT NULL,
                `commission_type` VARCHAR(50) DEFAULT 'sale',
                `sale_amount` DECIMAL(10,2) DEFAULT 0.00,
                `commission_amount` DECIMAL(10,2) DEFAULT 0.00,
                `status` ENUM('pending', 'approved', 'paid', 'rejected') DEFAULT 'pending',
                `invoice_id` INT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_affiliate_id` (`affiliate_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_affiliatetracking_clicks', "
            CREATE TABLE `mod_affiliatetracking_clicks` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `affiliate_id` INT NOT NULL,
                `ip_address` VARCHAR(45) NULL,
                `user_agent` VARCHAR(500) NULL,
                `landing_page` VARCHAR(500) NULL,
                `clicked_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_affiliate_clicks` (`affiliate_id`, `clicked_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_affiliatetracking_payouts', "
            CREATE TABLE `mod_affiliatetracking_payouts` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `affiliate_id` INT NOT NULL,
                `amount` DECIMAL(10,2) NOT NULL,
                `method` VARCHAR(50) NOT NULL,
                `status` ENUM('pending', 'processing', 'completed', 'failed') DEFAULT 'pending',
                `transaction_id` VARCHAR(255) NULL,
                `processed_at` DATETIME NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Affiliate Tracking module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function affiliatetracking_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function affiliatetracking_RegisterAffiliate($userId, $commissionRate = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $code = 'AFF' . strtoupper(substr(md5($userId . uniqid()), 0, 8));
        Capsule::table('mod_affiliatetracking_affiliates')->insert(array('user_id' => $userId, 'affiliate_code' => $code, 'commission_rate' => $commissionRate ?? 10));
        return array('success' => true, 'affiliate_code' => $code);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function affiliatetracking_GetAffiliate($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_affiliatetracking_affiliates')->where('user_id', $userId)->first();
}

function affiliatetracking_GetAffiliateByCode($code) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_affiliatetracking_affiliates')->where('affiliate_code', $code)->first();
}

function affiliatetracking_TrackClick($affiliateId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_affiliatetracking_clicks')->insert(array('affiliate_id' => $affiliateId, 'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null, 'user_agent' => substr($_SERVER['HTTP_USER_AGENT'] ?? '', 0, 500), 'landing_page' => substr($_SERVER['REQUEST_URI'] ?? '', 0, 500)));
}

function affiliatetracking_RecordReferral($affiliateId, $referredUserId, $saleAmount, $invoiceId = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $affiliate = Capsule::table('mod_affiliatetracking_affiliates')->where('id', $affiliateId)->first();
        if (!$affiliate) return array('success' => false, 'error' => 'Affiliate not found');
        $commission = $saleAmount * ($affiliate->commission_rate / 100);
        Capsule::table('mod_affiliatetracking_referrals')->insert(array('affiliate_id' => $affiliateId, 'referred_user_id' => $referredUserId, 'sale_amount' => $saleAmount, 'commission_amount' => $commission, 'invoice_id' => $invoiceId));
        Capsule::table('mod_affiliatetracking_affiliates')->where('id', $affiliateId)->update(array('total_referred' => $affiliate->total_referred + 1, 'pending_balance' => $affiliate->pending_balance + $commission));
        return array('success' => true, 'commission' => $commission);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function affiliatetracking_GetReferrals($affiliateId, $status = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_affiliatetracking_referrals')->where('affiliate_id', $affiliateId);
    if ($status) { $query->where('status', $status); }
    return $query->orderBy('created_at', 'desc')->get();
}

function affiliatetracking_GetStatistics($affiliateId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $affiliate = affiliatetracking_GetAffiliateById($affiliateId);
    $clicks = Capsule::table('mod_affiliatetracking_clicks')->where('affiliate_id', $affiliateId)->count();
    $conversions = Capsule::table('mod_affiliatetracking_referrals')->where('affiliate_id', $affiliateId)->count();
    return array('total_clicks' => $clicks, 'total_referrals' => $conversions, 'conversion_rate' => $clicks > 0 ? round(($conversions / $clicks) * 100, 2) : 0, 'total_earned' => $affiliate->total_earned, 'pending_balance' => $affiliate->pending_balance, 'total_paid' => $affiliate->total_paid);
}

function affiliatetracking_GetAffiliateById($affiliateId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_affiliatetracking_affiliates')->where('id', $affiliateId)->first();
}

function affiliatetracking_SetCommissionRate($affiliateId, $rate) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_affiliatetracking_affiliates')->where('id', $affiliateId)->update(array('commission_rate' => $rate));
}

function affiliatetracking_CreatePayout($affiliateId, $amount, $method) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $affiliate = affiliatetracking_GetAffiliateById($affiliateId);
        if ($affiliate->pending_balance < $amount) return array('success' => false, 'error' => 'Insufficient balance');
        Capsule::table('mod_affiliatetracking_payouts')->insert(array('affiliate_id' => $affiliateId, 'amount' => $amount, 'method' => $method));
        Capsule::table('mod_affiliatetracking_affiliates')->where('id', $affiliateId)->update(array('pending_balance' => $affiliate->pending_balance - $amount, 'total_paid' => $affiliate->total_paid + $amount));
        Capsule::table('mod_affiliatetracking_referrals')->where('affiliate_id', $affiliateId)->where('status', 'pending')->update(array('status' => 'paid'));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}
