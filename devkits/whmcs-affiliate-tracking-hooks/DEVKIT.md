# WHMCS Affiliate Tracking Hooks Module

## Overview
Comprehensive affiliate commission tracking module with AffiliateCommission hooks for tracking referrals, commissions, and payouts.

## Module File: hooks.php

```php
<?php
/**
 * WHMCS Affiliate Tracking Hooks Module
 * 
 * @package    WHMCS
 * @subpackage Modules
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Hook: AffiliateComission (custom hook for commission events)
 * Triggered when affiliate commission is awarded
 */
function whmcs_affiliate_tracking_commission(array $params): array
{
    try {
        $affiliateId = $params['affiliate_id'] ?? $params['id'];
        $referralUserId = $params['referral_user_id'] ?? $params['userid'];
        $amount = $params['amount'] ?? 0;
        $orderId = $params['order_id'] ?? 0;
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        logModuleCall('AffiliateTracking', 'Commission', [
            'affiliate_id' => $affiliateId,
            'amount' => $amount,
        ], 'Commission awarded', '');

        // Get affiliate details
        $affiliate = Capsule::table('tblaffiliates')
            ->where('id', $affiliateId)
            ->first();

        if (!$affiliate) {
            return [];
        }

        // Calculate commission amount
        $commissionPercent = $config['commission_percent'];
        $commissionAmount = ($amount * $commissionPercent) / 100;

        // Store commission record
        Capsule::table('mod_affiliate_commissions')->insert([
            'affiliate_id' => $affiliateId,
            'referral_user_id' => $referralUserId,
            'order_id' => $orderId,
            'sale_amount' => $amount,
            'commission_percent' => $commissionPercent,
            'commission_amount' => $commissionAmount,
            'status' => 'pending',
            'created_at' => $now,
        ]);

        // Update affiliate stats
        Capsule::table('mod_affiliate_stats')
            ->where('affiliate_id', $affiliateId)
            ->update([
                'total_referrals' => Capsule::raw('total_referrals + 1'),
                'pending_commissions' => Capsule::raw('pending_commissions + ' . $commissionAmount),
            ]);

        // Check for bonus commissions
        checkCommissionBonus($affiliateId, $config);

        // Notify affiliate of new commission
        if (!empty($config['notify_commission'])) {
            notifyAffiliateCommission($affiliateId, $commissionAmount);
        }

    } catch (\Exception $e) {
        logModuleCall('AffiliateTracking', 'Commission Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: AffiliateCommissionApproved
 * Triggered when commission is approved for payout
 */
function whmcs_affiliate_tracking_approved(array $params): array
{
    try {
        $affiliateId = $params['affiliate_id'] ?? $params['id'];
        $commissionId = $params['commission_id'] ?? 0;
        $amount = $params['amount'] ?? 0;
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        // Update commission status
        Capsule::table('mod_affiliate_commissions')
            ->where('id', $commissionId)
            ->update([
                'status' => 'approved',
                'approved_at' => $now,
            ]);

        // Update stats
        Capsule::table('mod_affiliate_stats')
            ->where('affiliate_id', $affiliateId)
            ->update([
                'pending_commissions' => Capsule::raw('pending_commissions - ' . $amount),
                'approved_commissions' => Capsule::raw('approved_commissions + ' . $amount),
            ]);

        // Log approval event
        Capsule::table('mod_affiliate_events')->insert([
            'affiliate_id' => $affiliateId,
            'event_type' => 'commission_approved',
            'event_data' => json_encode([
                'commission_id' => $commissionId,
                'amount' => $amount,
            ]),
            'created_at' => $now,
        ]);

    } catch (\Exception $e) {
        logModuleCall('AffiliateTracking', 'Approved Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: AffiliateCommissionPaid
 * Triggered when commission is paid out
 */
function whmcs_affiliate_tracking_paid(array $params): array
{
    try {
        $affiliateId = $params['affiliate_id'] ?? $params['id'];
        $amount = $params['amount'] ?? 0;
        $payoutId = $params['payout_id'] ?? 0;
        $now = date('Y-m-d H:i:s');
        $config = require __DIR__ . '/config.php';

        // Store payout event
        Capsule::table('mod_affiliate_events')->insert([
            'affiliate_id' => $affiliateId,
            'event_type' => 'commission_paid',
            'event_data' => json_encode([
                'payout_id' => $payoutId,
                'amount' => $amount,
                'payment_method' => $params['payment_method'] ?? 'unknown',
            ]),
            'created_at' => $now,
        ]);

        // Update stats
        Capsule::table('mod_affiliate_stats')
            ->where('affiliate_id', $affiliateId)
            ->update([
                'approved_commissions' => Capsule::raw('approved_commissions - ' . $amount),
                'total_paid' => Capsule::raw('total_paid + ' . $amount),
                'last_payout_date' => $now,
            ]);

        // Send payment notification
        if (!empty($config['notify_payout'])) {
            notifyAffiliatePayout($affiliateId, $amount);
        }

    } catch (\Exception $e) {
        logModuleCall('AffiliateTracking', 'Paid Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Hook: AffiliateRejectCommission
 * Triggered when commission is rejected
 */
function whmcs_affiliate_tracking_rejected(array $params): array
{
    try {
        $affiliateId = $params['affiliate_id'] ?? $params['id'];
        $commissionId = $params['commission_id'] ?? 0;
        $amount = $params['amount'] ?? 0;
        $reason = $params['reason'] ?? 'Unknown';
        $now = date('Y-m-d H:i:s');

        // Update commission status
        Capsule::table('mod_affiliate_commissions')
            ->where('id', $commissionId)
            ->update([
                'status' => 'rejected',
                'rejected_at' => $now,
                'rejection_reason' => $reason,
            ]);

        // Update stats
        Capsule::table('mod_affiliate_stats')
            ->where('affiliate_id', $affiliateId)
            ->update([
                'pending_commissions' => Capsule::raw('pending_commissions - ' . $amount),
                'rejected_commissions' => Capsule::raw('rejected_commissions + ' . $amount),
            ]);

        // Log rejection
        Capsule::table('mod_affiliate_events')->insert([
            'affiliate_id' => $affiliateId,
            'event_type' => 'commission_rejected',
            'event_data' => json_encode([
                'commission_id' => $commissionId,
                'amount' => $amount,
                'reason' => $reason,
            ]),
            'created_at' => $now,
        ]);

    } catch (\Exception $e) {
        logModuleCall('AffiliateTracking', 'Rejected Error', $params, $e->getMessage(), '');
    }

    return [];
}

/**
 * Check for commission bonuses
 */
function checkCommissionBonus(int $affiliateId, array $config): void
{
    if (empty($config['enable_tier_bonus'])) {
        return;
    }

    $stats = Capsule::table('mod_affiliate_stats')
        ->where('affiliate_id', $affiliateId)
        ->first();

    if (!$stats) {
        return;
    }

    $tiers = $config['bonus_tiers'] ?? [];
    $currentReferrals = $stats->total_referrals;
    
    foreach ($tiers as $tier) {
        if ($currentReferrals >= $tier['threshold'] && $stats->bonus_level < $tier['threshold']) {
            // Award bonus
            $bonusAmount = $tier['bonus_amount'];
            
            Capsule::table('mod_affiliate_commissions')->insert([
                'affiliate_id' => $affiliateId,
                'referral_user_id' => 0,
                'order_id' => 0,
                'sale_amount' => 0,
                'commission_percent' => 100,
                'commission_amount' => $bonusAmount,
                'is_bonus' => true,
                'bonus_tier' => $tier['name'],
                'status' => 'pending',
                'created_at' => date('Y-m-d H:i:s'),
            ]);

            Capsule::table('mod_affiliate_stats')
                ->where('affiliate_id', $affiliateId)
                ->update([
                    'bonus_level' => $tier['threshold'],
                    'pending_commissions' => Capsule::raw('pending_commissions + ' . $bonusAmount),
                ]);
            
            break;
        }
    }
}

/**
 * Notify affiliate of new commission
 */
function notifyAffiliateCommission(int $affiliateId, float $amount): void
{
    $affiliate = Capsule::table('tblaffiliates')
        ->join('tblclients', 'tblaffiliates.clientid', '=', 'tblclients.id')
        ->where('tblaffiliates.id', $affiliateId)
        ->first(['tblclients.email', 'tblclients.firstname', 'tblclients.lastname']);

    if ($affiliate) {
        $command = 'SendEmail';
        $postData = [
            'id' => $affiliateId,
            'type' => 'affiliate',
            'templatename' => 'Affiliate Commission Notification',
            'customvars' => base64_encode(json_encode([
                'affiliate_name' => $affiliate->firstname . ' ' . $affiliate->lastname,
                'commission_amount' => number_format($amount, 2),
            ])),
        ];
        localAPI($command, $postData);
    }
}

/**
 * Notify affiliate of payout
 */
function notifyAffiliatePayout(int $affiliateId, float $amount): void
{
    $affiliate = Capsule::table('tblaffiliates')
        ->join('tblclients', 'tblaffiliates.clientid', '=', 'tblclients.id')
        ->where('tblaffiliates.id', $affiliateId)
        ->first(['tblclients.email', 'tblclients.firstname', 'tblclients.lastname']);

    if ($affiliate) {
        $command = 'SendEmail';
        $postData = [
            'id' => $affiliateId,
            'type' => 'affiliate',
            'templatename' => 'Affiliate Payout Notification',
            'customvars' => base64_encode(json_encode([
                'affiliate_name' => $affiliate->firstname . ' ' . $affiliate->lastname,
                'payout_amount' => number_format($amount, 2),
            ])),
        ];
        localAPI($command, $postData);
    }
}

/**
 * Calculate affiliate performance metrics
 */
function calculateAffiliateMetrics(int $affiliateId): array
{
    $stats = Capsule::table('mod_affiliate_stats')
        ->where('affiliate_id', $affiliateId)
        ->first();

    $commissions = Capsule::table('mod_affiliate_commissions')
        ->where('affiliate_id', $affiliateId)
        ->where('created_at', '>=', date('Y-m-d', strtotime('-30 days')))
        ->get();

    $totalCommissions = 0;
    foreach ($commissions as $c) {
        $totalCommissions += $c->commission_amount;
    }

    return [
        'total_referrals' => $stats->total_referrals ?? 0,
        'conversion_rate' => $stats->total_referrals > 0 
            ? round(($stats->total_referrals / max(1, $stats->clicks)) * 100, 2) 
            : 0,
        'monthly_commissions' => $totalCommissions,
        'pending_commissions' => $stats->pending_commissions ?? 0,
        'total_paid' => $stats->total_paid ?? 0,
    ];
}

// Register hooks
add_hook('AffiliateCommission', 1, 'whmcs_affiliate_tracking_commission');
add_hook('AffiliateCommissionApproved', 1, 'whmcs_affiliate_tracking_approved');
add_hook('AffiliateCommissionPaid', 1, 'whmcs_affiliate_tracking_paid');
add_hook('AffiliateRejectCommission', 1, 'whmcs_affiliate_tracking_rejected');
```

## Configuration File: config.php

```php
<?php
return [
    // Commission Settings
    'commission_percent' => 10.0,
    'recurring_commission' => true,
    'recurring_months' => 12,

    // Bonus Tiers
    'enable_tier_bonus' => true,
    'bonus_tiers' => [
        ['name' => 'Bronze', 'threshold' => 10, 'bonus_amount' => 25],
        ['name' => 'Silver', 'threshold' => 25, 'bonus_amount' => 75],
        ['name' => 'Gold', 'threshold' => 50, 'bonus_amount' => 150],
        ['name' => 'Platinum', 'threshold' => 100, 'bonus_amount' => 500],
    ],

    // Notifications
    'notify_commission' => true,
    'notify_payout' => true,

    // Tracking
    'track_clicks' => true,

    // Payout Settings
    'minimum_payout' => 50.00,
    'payout_methods' => ['paypal', 'bank_transfer', 'skrill'],
];
```

## Database Schema

```php
<?php
use WHMCS\Database\Capsule;

// Commissions table
if (!Capsule::schema()->hasTable('mod_affiliate_commissions')) {
    Capsule::schema()->create('mod_affiliate_commissions', function ($table) {
        $table->increments('id');
        $table->integer('affiliate_id')->unsigned();
        $table->integer('referral_user_id')->unsigned()->default(0);
        $table->integer('order_id')->unsigned()->default(0);
        $table->decimal('sale_amount', 10, 2)->default(0);
        $table->decimal('commission_percent', 5, 2);
        $table->decimal('commission_amount', 10, 2);
        $table->boolean('is_bonus')->default(false);
        $table->string('bonus_tier', 50)->nullable();
        $table->enum('status', ['pending', 'approved', 'rejected', 'paid'])->default('pending');
        $table->timestamp('approved_at')->nullable();
        $table->timestamp('rejected_at')->nullable();
        $table->text('rejection_reason')->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('affiliate_id');
        $table->index('status');
        $table->index('created_at');
    });
}

// Stats table
if (!Capsule::schema()->hasTable('mod_affiliate_stats')) {
    Capsule::schema()->create('mod_affiliate_stats', function ($table) {
        $table->increments('id');
        $table->integer('affiliate_id')->unsigned()->unique();
        $table->integer('total_referrals')->default(0);
        $table->integer('total_clicks')->default(0);
        $table->integer('bonus_level')->default(0);
        $table->decimal('pending_commissions', 10, 2)->default(0);
        $table->decimal('approved_commissions', 10, 2)->default(0);
        $table->decimal('rejected_commissions', 10, 2)->default(0);
        $table->decimal('total_paid', 10, 2)->default(0);
        $table->timestamp('last_payout_date')->nullable();
        $table->timestamp('updated_at')->useCurrent();
    });
}

// Events table
if (!Capsule::schema()->hasTable('mod_affiliate_events')) {
    Capsule::schema()->create('mod_affiliate_events', function ($table) {
        $table->increments('id');
        $table->integer('affiliate_id')->unsigned();
        $table->string('event_type', 50);
        $table->longText('event_data')->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('affiliate_id');
        $table->index('event_type');
    });
}
```

## Activation & Deactivation

```php
<?php
function whmcs_affiliate_tracking_activate(): array
{
    try {
        require_once __DIR__ . '/schema_migration.php';
        return ['status' => 'success', 'description' => 'Affiliate Tracking Hooks activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_affiliate_tracking_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Affiliate Tracking Hooks deactivated'];
}
```

## Hooks Reference

| Hook | Description |
|------|-------------|
| AffiliateCommission | Commission awarded |
| AffiliateCommissionApproved | Commission approved |
| AffiliateCommissionPaid | Commission paid out |
| AffiliateRejectCommission | Commission rejected |

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
