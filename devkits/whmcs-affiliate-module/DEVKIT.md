# WHMCS Affiliate Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## Purpose
Create an affiliate management module for WHMCS to track referrals, manage commissions, and handle affiliate payouts.

## Module Type
Addon Module

## Use Case
- Track affiliate referrals
- Calculate and manage commissions
- Affiliate dashboard for partners
- Automatic commission rules
- Payout management

## DevKit Structure

```
devkits/whmcs-affiliate-module/
├── affiliate.php          # Main addon module
├── lib/
│   ├── AffiliateManager.php  # Affiliate operations
│   ├── CommissionCalculator.php # Commission logic
│   └── PayoutManager.php     # Payout handling
├── templates/
│   ├── admin.tpl          # Admin templates
│   └── client.tpl         # Affiliate dashboard
├── hooks.php              # Hook integrations
└── DEVKIT.md            # This file
```

## Main Module Template

```php
<?php
/**
 * WHMCS Affiliate Module: {Affiliate}
 * Affiliate Management Module Template
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function {affiliate}_config(): array {
    return [
        'name' => '{Affiliate Module}',
        'description' => 'Affiliate tracking and commission management',
        'version' => '1.0',
        'author' => '{Author Name}',

        'commission_type' => [
            'FriendlyName' => 'Commission Type',
            'Type' => 'dropdown',
            'Options' => 'percentage,fixed,both',
            'Default' => 'percentage',
            'Description' => 'How commissions are calculated',
        ],
        'default_commission' => [
            'FriendlyName' => 'Default Commission %',
            'Type' => 'text',
            'Size' => '5',
            'Default' => '10',
        ],
        'fixed_commission' => [
            'FriendlyName' => 'Fixed Commission Amount',
            'Type' => 'text',
            'Size' => '10',
            'Default' => '0',
        ],
        'cookie_days' => [
            'FriendlyName' => 'Cookie Duration (Days)',
            'Type' => 'text',
            'Size' => '5',
            'Default' => '30',
        ],
        'min_payout' => [
            'FriendlyName' => 'Minimum Payout',
            'Type' => 'text',
            'Size' => '10',
            'Default' => '50',
        ],
        'auto_approve' => [
            'FriendlyName' => 'Auto-Approve Affiliates',
            'Type' => 'yesno',
            'Description' => 'Automatically approve new affiliate applications',
        ],
        'commission_delay' => [
            'FriendlyName' => 'Commission Delay (Days)',
            'Type' => 'text',
            'Size' => '5',
            'Default' => '14',
            'Description' => 'Days before commission becomes payable',
        ],
    ];
}

function {affiliate}_activate(): array {
    try {
        // Affiliates table
        if (!Capsule::schema()->hasTable('mod_{affiliate}_affiliates')) {
            Capsule::schema()->create('mod_{affiliate}_affiliates', function($t) {
                $t->increments('id');
                $t->integer('user_id')->unsigned()->unique();
                $t->string('affiliate_code', 50)->unique();
                $t->string('referral_code', 50)->nullable()->unique();
                $t->decimal('commission_rate', 5, 2)->default(10);
                $t->string('status', 20)->default('pending');
                $t->text('notes')->nullable();
                $t->timestamp('approved_at')->nullable();
                $t->integer('approved_by')->unsigned()->nullable();
                $t->timestamps();

                $t->index('affiliate_code');
                $t->index('status');
            });
        }

        // Referrals/Tracking table
        if (!Capsule::schema()->hasTable('mod_{affiliate}_referrals')) {
            Capsule::schema()->create('mod_{affiliate}_referrals', function($t) {
                $t->increments('id');
                $t->integer('affiliate_id')->unsigned();
                $t->integer('visitor_id')->unsigned()->nullable();
                $t->string('ip_address', 45)->nullable();
                $t->string('user_agent', 255)->nullable();
                $t->string('landing_page', 255)->nullable();
                $t->string('referrer', 255)->nullable();
                $t->timestamp('clicked_at');
                $t->timestamp('converted_at')->nullable();
                $t->integer('converted_order_id')->unsigned()->nullable();

                $t->index('affiliate_id');
                $t->index('clicked_at');
                $t->index('converted_at');
            });
        }

        // Commissions table
        if (!Capsule::schema()->hasTable('mod_{affiliate}_commissions')) {
            Capsule::schema()->create('mod_{affiliate}_commissions', function($t) {
                $t->increments('id');
                $t->integer('affiliate_id')->unsigned();
                $t->integer('referral_id')->unsigned()->nullable();
                $t->integer('order_id')->unsigned()->nullable();
                $t->integer('client_id')->unsigned()->nullable();
                $t->decimal('order_amount', 15, 2);
                $t->decimal('commission_amount', 15, 2);
                $t->string('commission_type', 20);
                $t->string('status', 20)->default('pending');
                $t->timestamp('created_at');
                $t->timestamp('payable_at')->nullable();
                $t->timestamp('paid_at')->nullable();
                $t->integer('payout_id')->unsigned()->nullable();
                $t->text('notes')->nullable();

                $t->index('affiliate_id');
                $t->index('status');
                $t->index('created_at');
            });
        }

        // Payouts table
        if (!Capsule::schema()->hasTable('mod_{affiliate}_payouts')) {
            Capsule::schema()->create('mod_{affiliate}_payouts', function($t) {
                $t->increments('id');
                $t->integer('affiliate_id')->unsigned();
                $t->decimal('amount', 15, 2);
                $t->string('method', 50);
                $t->string('status', 20)->default('pending');
                $t->string('transaction_id', 100)->nullable();
                $t->text('notes')->nullable();
                $t->timestamp('requested_at');
                $t->timestamp('processed_at')->nullable();
                $t->integer('processed_by')->unsigned()->nullable();

                $t->index('affiliate_id');
                $t->index('status');
            });
        }

        // Affiliate tiers for multi-level commissions
        if (!Capsule::schema()->hasTable('mod_{affiliate}_tiers')) {
            Capsule::schema()->create('mod_{affiliate}_tiers', function($t) {
                $t->increments('id');
                $t->string('name', 100);
                $t->integer('level')->unsigned();
                $t->decimal('commission_rate', 5, 2);
                $t->integer('min_referrals')->default(0);
                $t->integer('min_payout')->default(0);
                $t->boolean('is_active')->default(true);
            });

            // Insert default tiers
            Capsule::table('mod_{affiliate}_tiers')->insert([
                ['name' => 'Bronze', 'level' => 1, 'commission_rate' => 10, 'min_referrals' => 0, 'is_active' => 1],
                ['name' => 'Silver', 'level' => 2, 'commission_rate' => 15, 'min_referrals' => 10, 'is_active' => 1],
                ['name' => 'Gold', 'level' => 3, 'commission_rate' => 20, 'min_referrals' => 25, 'is_active' => 1],
                ['name' => 'Platinum', 'level' => 4, 'commission_rate' => 25, 'min_referrals' => 50, 'is_active' => 1],
            ]);
        }

        return [
            'status' => 'success',
            'description' => '{Affiliate Module} activated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Activation failed: ' . $e->getMessage(),
        ];
    }
}

function {affiliate}_deactivate(): array {
    try {
        Capsule::schema()->dropIfExists('mod_{affiliate}_affiliates');
        Capsule::schema()->dropIfExists('mod_{affiliate}_referrals');
        Capsule::schema()->dropIfExists('mod_{affiliate}_commissions');
        Capsule::schema()->dropIfExists('mod_{affiliate}_payouts');
        Capsule::schema()->dropIfExists('mod_{affiliate}_tiers');

        return [
            'status' => 'success',
            'description' => '{Affiliate Module} deactivated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Deactivation failed: ' . $e->getMessage(),
        ];
    }
}

function {affiliate}_output(array $vars): void {
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
        handleAffiliateAction($_POST['action'] ?? '');
    }

    $tab = $_REQUEST['tab'] ?? 'overview';

    echo '<div class="affiliate-module">';
    echo '<div class="affiliate-header">';
    echo '<h1><i class="fa fa-users"></i> Affiliate Management</h1>';
    echo '</div>';

    echo '<ul class="nav nav-tabs">';
    echo '<li class="' . ($tab === 'overview' ? 'active' : '') . '"><a href="?module={affiliate}&tab=overview">Overview</a></li>';
    echo '<li class="' . ($tab === 'affiliates' ? 'active' : '') . '"><a href="?module={affiliate}&tab=affiliates">Affiliates</a></li>';
    echo '<li class="' . ($tab === 'commissions' ? 'active' : '') . '"><a href="?module={affiliate}&tab=commissions">Commissions</a></li>';
    echo '<li class="' . ($tab === 'payouts' ? 'active' : '') . '"><a href="?module={affiliate}&tab=payouts">Payouts</a></li>';
    echo '<li class="' . ($tab === 'reports' ? 'active' : '') . '"><a href="?module={affiliate}&tab=reports">Reports</a></li>';
    echo '<li class="' . ($tab === 'tiers' ? 'active' : '') . '"><a href="?module={affiliate}&tab=tiers">Tiers</a></li>';
    echo '</ul>';

    include __DIR__ . '/templates/admin/' . $tab . '.tpl';
    echo '</div>';
}

function {affiliate}_clientarea(array $vars): array {
    $userId = $_SESSION['uid'];
    $affiliate = getAffiliateByUserId($userId);

    if (!$affiliate || $affiliate->status !== 'active') {
        return [
            'pagetitle' => 'Affiliate Program',
            'templatefile' => 'templates/join',
            'vars' => [],
            'requirelogin' => true,
        ];
    }

    return [
        'pagetitle' => 'My Affiliate Dashboard',
        'templatefile' => 'templates/dashboard',
        'vars' => [
            'affiliate' => $affiliate,
            'stats' => getAffiliateStats($affiliate->id),
            'referrals' => getRecentReferrals($affiliate->id, 10),
            'commissions' => getPendingCommissions($affiliate->id),
            'referral_link' => getAffiliateLink($affiliate->affiliate_code),
        ],
        'requirelogin' => true,
    ];
}

// Helper Functions
function handleAffiliateAction(string $action): void {
    switch ($action) {
        case 'approve_affiliate':
            approveAffiliate((int)($_POST['affiliate_id'] ?? 0));
            break;
        case 'reject_affiliate':
            rejectAffiliate((int)($_POST['affiliate_id'] ?? 0), $_POST['reason'] ?? '');
            break;
        case 'update_commission':
            updateCommission((int)($_POST['commission_id'] ?? 0));
            break;
        case 'process_payout':
            processPayout((int)($_POST['payout_id'] ?? 0));
            break;
        case 'save_tier':
            saveTier();
            break;
    }

    header('Location: ?module={affiliate}&tab=' . ($_POST['redirect_tab'] ?? 'overview'));
    exit;
}

function generateAffiliateCode(): string {
    return strtoupper(substr(md5(uniqid()), 0, 8));
}

function getAffiliateLink(string $code): string {
    $systemUrl = Capsule::table('tblconfiguration')
        ->where('setting', 'SystemURL')
        ->value('value') ?? 'https://example.com';

    return rtrim($systemUrl, '/') . '/affiliate/' . $code;
}

function getAffiliateByUserId(int $userId): ?object {
    return Capsule::table('mod_{affiliate}_affiliates')
        ->where('user_id', $userId)
        ->first();
}

function getAffiliateStats(int $affiliateId): array {
    $totalClicks = Capsule::table('mod_{affiliate}_referrals')
        ->where('affiliate_id', $affiliateId)
        ->count();

    $totalConversions = Capsule::table('mod_{affiliate}_referrals')
        ->where('affiliate_id', $affiliateId)
        ->whereNotNull('converted_at')
        ->count();

    $pendingCommission = Capsule::table('mod_{affiliate}_commissions')
        ->where('affiliate_id', $affiliateId)
        ->where('status', 'pending')
        ->sum('commission_amount');

    $approvedCommission = Capsule::table('mod_{affiliate}_commissions')
        ->where('affiliate_id', $affiliateId)
        ->whereIn('status', ['approved', 'payable'])
        ->sum('commission_amount');

    $paidCommission = Capsule::table('mod_{affiliate}_commissions')
        ->where('affiliate_id', $affiliateId)
        ->where('status', 'paid')
        ->sum('commission_amount');

    return [
        'total_clicks' => $totalClicks,
        'total_conversions' => $totalConversions,
        'conversion_rate' => $totalClicks > 0 ? round(($totalConversions / $totalClicks) * 100, 2) : 0,
        'pending_commission' => $pendingCommission ?? 0,
        'approved_commission' => $approvedCommission ?? 0,
        'paid_commission' => $paidCommission ?? 0,
        'total_earned' => ($approvedCommission ?? 0) + ($paidCommission ?? 0),
    ];
}

function getRecentReferrals(int $affiliateId, int $limit = 10): array {
    return Capsule::table('mod_{affiliate}_referrals')
        ->where('affiliate_id', $affiliateId)
        ->orderBy('clicked_at', 'desc')
        ->limit($limit)
        ->get()
        ->toArray();
}

function getPendingCommissions(int $affiliateId): array {
    return Capsule::table('mod_{affiliate}_commissions')
        ->where('affiliate_id', $affiliateId)
        ->whereIn('status', ['pending', 'approved'])
        ->orderBy('created_at', 'desc')
        ->get()
        ->toArray();
}

function approveAffiliate(int $affiliateId): void {
    Capsule::table('mod_{affiliate}_affiliates')
        ->where('id', $affiliateId)
        ->update([
            'status' => 'active',
            'approved_at' => date('Y-m-d H:i:s'),
            'approved_by' => $_SESSION['adminid'],
        ]);

    $affiliate = Capsule::table('mod_{affiliate}_affiliates')
        ->where('id', $affiliateId)
        ->first();

    if ($affiliate) {
        sendEmail($affiliate->user_id, 'AffiliateApproved', ['affiliate_code' => $affiliate->affiliate_code]);
    }
}

function rejectAffiliate(int $affiliateId, string $reason): void {
    Capsule::table('mod_{affiliate}_affiliates')
        ->where('id', $affiliateId)
        ->update([
            'status' => 'rejected',
            'notes' => $reason,
        ]);
}

function updateCommission(int $commissionId): void {
    $amount = (float)($_POST['amount'] ?? 0);
    $status = $_POST['status'] ?? 'pending';

    $data = ['status' => $status];
    if ($amount > 0) {
        $data['commission_amount'] = $amount;
    }

    Capsule::table('mod_{affiliate}_commissions')
        ->where('id', $commissionId)
        ->update($data);
}

function processPayout(int $payoutId): void {
    $transactionId = $_POST['transaction_id'] ?? '';
    $status = $_POST['status'] ?? 'completed';

    Capsule::table('mod_{affiliate}_payouts')
        ->where('id', $payoutId)
        ->update([
            'status' => $status,
            'transaction_id' => $transactionId,
            'processed_at' => date('Y-m-d H:i:s'),
            'processed_by' => $_SESSION['adminid'],
        ]);

    if ($status === 'paid') {
        $payout = Capsule::table('mod_{affiliate}_payouts')
            ->where('id', $payoutId)
            ->first();

        if ($payout) {
            Capsule::table('mod_{affiliate}_commissions')
                ->where('affiliate_id', $payout->affiliate_id)
                ->whereIn('status', ['approved', 'payable'])
                ->limit(10)
                ->update([
                    'status' => 'paid',
                    'paid_at' => date('Y-m-d H:i:s'),
                    'payout_id' => $payoutId,
                ]);
        }
    }
}

function saveTier(): void {
    $id = (int)($_POST['id'] ?? 0);
    $data = [
        'name' => $_POST['name'] ?? '',
        'level' => (int)($_POST['level'] ?? 1),
        'commission_rate' => (float)($_POST['commission_rate'] ?? 10),
        'min_referrals' => (int)($_POST['min_referrals'] ?? 0),
        'min_payout' => (float)($_POST['min_payout'] ?? 0),
        'is_active' => isset($_POST['is_active']) ? 1 : 0,
    ];

    if ($id > 0) {
        Capsule::table('mod_{affiliate}_tiers')
            ->where('id', $id)
            ->update($data);
    }
}
```

## Affiliate Manager Class

```php
<?php
namespace WHMCS\Module\Addon\{Affiliate};

use WHMCS\Database\Capsule;

class AffiliateManager {

    public function registerAffiliate(int $userId, ?string $referralCode = null): ?int {
        $existing = Capsule::table('mod_{affiliate}_affiliates')
            ->where('user_id', $userId)
            ->first();

        if ($existing) {
            return $existing->id;
        }

        $settings = $this->getModuleSettings();

        return Capsule::table('mod_{affiliate}_affiliates')->insertGetId([
            'user_id' => $userId,
            'affiliate_code' => $this->generateAffiliateCode(),
            'referral_code' => $referralCode,
            'commission_rate' => $settings['default_commission'] ?? 10,
            'status' => ($settings['auto_approve'] ?? '') === 'on' ? 'active' : 'pending',
            'approved_at' => ($settings['auto_approve'] ?? '') === 'on' ? date('Y-m-d H:i:s') : null,
            'created_at' => date('Y-m-d H:i:s'),
            'updated_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function trackClick(string $affiliateCode, array $data = []): ?int {
        $affiliate = Capsule::table('mod_{affiliate}_affiliates')
            ->where('affiliate_code', $affiliateCode)
            ->where('status', 'active')
            ->first();

        if (!$affiliate) {
            return null;
        }

        $settings = $this->getModuleSettings();
        $cookieDays = (int)($settings['cookie_days'] ?? 30);

        $referralId = Capsule::table('mod_{affiliate}_referrals')->insertGetId([
            'affiliate_id' => $affiliate->id,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'user_agent' => substr($_SERVER['HTTP_USER_AGENT'] ?? '', 0, 255),
            'landing_page' => $data['landing_page'] ?? '',
            'referrer' => $_SERVER['HTTP_REFERER'] ?? '',
            'clicked_at' => date('Y-m-d H:i:s'),
        ]);

        setcookie('affiliate_ref', $affiliateCode, time() + ($cookieDays * 86400), '/');

        return $referralId;
    }

    public function trackConversion(int $referralId, int $orderId): bool {
        $referral = Capsule::table('mod_{affiliate}_referrals')
            ->where('id', $referralId)
            ->first();

        if (!$referral || $referral->converted_at) {
            return false;
        }

        Capsule::table('mod_{affiliate}_referrals')
            ->where('id', $referralId)
            ->update([
                'converted_at' => date('Y-m-d H:i:s'),
                'converted_order_id' => $orderId,
            ]);

        return true;
    }

    public function calculateCommission(int $affiliateId, float $orderAmount): array {
        $affiliate = Capsule::table('mod_{affiliate}_affiliates')
            ->where('id', $affiliateId)
            ->first();

        if (!$affiliate) {
            return ['amount' => 0, 'type' => 'none'];
        }

        $settings = $this->getModuleSettings();
        $type = $settings['commission_type'] ?? 'percentage';

        $amount = 0;
        switch ($type) {
            case 'percentage':
                $amount = ($orderAmount * $affiliate->commission_rate) / 100;
                break;
            case 'fixed':
                $amount = (float)($settings['fixed_commission'] ?? 0);
                break;
            case 'both':
                $percentAmount = ($orderAmount * $affiliate->commission_rate) / 100;
                $fixedAmount = (float)($settings['fixed_commission'] ?? 0);
                $amount = $percentAmount + $fixedAmount;
                break;
        }

        return [
            'amount' => round($amount, 2),
            'type' => $type,
            'rate' => $affiliate->commission_rate,
        ];
    }

    public function getAffiliateTier(int $affiliateId): ?object {
        $totalReferrals = Capsule::table('mod_{affiliate}_referrals')
            ->where('affiliate_id', $affiliateId)
            ->whereNotNull('converted_at')
            ->count();

        return Capsule::table('mod_{affiliate}_tiers')
            ->where('is_active', 1)
            ->where('min_referrals', '<=', $totalReferrals)
            ->orderBy('level', 'desc')
            ->first();
    }

    private function generateAffiliateCode(): string {
        return strtoupper(substr(md5(uniqid()), 0, 8));
    }

    private function getModuleSettings(): array {
        $result = Capsule::table('tbladdon_modules')
            ->where('module', '{affiliate}')
            ->first();

        return $result ? json_decode($result->value, true) : [];
    }
}
```

## Commission Calculator Class

```php
<?php
namespace WHMCS\Module\Addon\{Affiliate};

use WHMCS\Database\Capsule;

class CommissionCalculator {

    public function createCommission(int $affiliateId, int $orderId, float $orderAmount): ?int {
        $manager = new AffiliateManager();
        $commission = $manager->calculateCommission($affiliateId, $orderAmount);

        if ($commission['amount'] <= 0) {
            return null;
        }

        $settings = $this->getModuleSettings();
        $delayDays = (int)($settings['commission_delay'] ?? 14);

        return Capsule::table('mod_{affiliate}_commissions')->insertGetId([
            'affiliate_id' => $affiliateId,
            'order_id' => $orderId,
            'client_id' => $this->getOrderClientId($orderId),
            'order_amount' => $orderAmount,
            'commission_amount' => $commission['amount'],
            'commission_type' => $commission['type'],
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s'),
            'payable_at' => date('Y-m-d H:i:s', strtotime("+{$delayDays} days")),
        ]);
    }

    public function approveCommission(int $commissionId): bool {
        return Capsule::table('mod_{affiliate}_commissions')
            ->where('id', $commissionId)
            ->where('status', 'pending')
            ->update(['status' => 'approved']) > 0;
    }

    public function makePayable(int $commissionId): bool {
        return Capsule::table('mod_{affiliate}_commissions')
            ->where('id', $commissionId)
            ->where('status', 'approved')
            ->update(['status' => 'payable']) > 0;
    }

    public function markPaid(int $commissionId): bool {
        return Capsule::table('mod_{affiliate}_commissions')
            ->where('id', $commissionId)
            ->whereIn('status', ['approved', 'payable'])
            ->update([
                'status' => 'paid',
                'paid_at' => date('Y-m-d H:i:s'),
            ]) > 0;
    }

    public function getPayableAmount(int $affiliateId): float {
        return Capsule::table('mod_{affiliate}_commissions')
            ->where('affiliate_id', $affiliateId)
            ->whereIn('status', ['approved', 'payable'])
            ->sum('commission_amount') ?? 0;
    }

    public function getLifetimeEarnings(int $affiliateId): float {
        return Capsule::table('mod_{affiliate}_commissions')
            ->where('affiliate_id', $affiliateId)
            ->where('status', 'paid')
            ->sum('commission_amount') ?? 0;
    }

    private function getOrderClientId(int $orderId): ?int {
        return Capsule::table('tblorders')
            ->where('id', $orderId)
            ->value('userid');
    }

    private function getModuleSettings(): array {
        $result = Capsule::table('tbladdon_modules')
            ->where('module', '{affiliate}')
            ->first();

        return $result ? json_decode($result->value, true) : [];
    }
}
```

## Hooks Integration

```php
<?php
/**
 * WHMCS Affiliate Module Hooks
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Track affiliate clicks via referral code URL
add_hook('PreCustomerAreaPage', 1, function(array $vars) {
    $uri = $_SERVER['REQUEST_URI'] ?? '';

    if (preg_match('/\/affiliate\/([A-Z0-9]{8})/i', $uri, $matches)) {
        $affiliateCode = $matches[1];
        $manager = new \WHMCS\Module\Addon\{Affiliate}\AffiliateManager();
        $manager->trackClick($affiliateCode);
    }
});

// Create commission on order payment
add_hook('InvoicePaid', 1, function(array $vars) {
    $invoiceId = $vars['invoiceid'];
    $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

    if (!$invoice || !$invoice->userid) {
        return;
    }

    $affiliateCode = $_COOKIE['affiliate_ref'] ?? '';
    if (!$affiliateCode) {
        return;
    }

    $affiliate = Capsule::table('mod_{affiliate}_affiliates')
        ->where('affiliate_code', $affiliateCode)
        ->where('status', 'active')
        ->where('user_id', '!=', $invoice->userid)
        ->first();

    if (!$affiliate) {
        return;
    }

    $calculator = new \WHMCS\Module\Addon\{Affiliate}\CommissionCalculator();
    $commissionId = $calculator->createCommission($affiliate->id, $invoiceId, $invoice->total);

    if ($commissionId) {
        logActivity("{Affiliate}: Commission created for affiliate #{$affiliate->id}, Order #{$invoiceId}");
    }
});

// Auto-approve commission after delay
add_hook('DailyCronJob', 1, function(array $vars) {
    $pendingCommissions = Capsule::table('mod_{affiliate}_commissions')
        ->where('status', 'pending')
        ->where('payable_at', '<=', date('Y-m-d H:i:s'))
        ->get();

    foreach ($pendingCommissions as $commission) {
        Capsule::table('mod_{affiliate}_commissions')
            ->where('id', $commission->id)
            ->update(['status' => 'payable']);

        logActivity("{Affiliate}: Commission #{$commission->id} is now payable");
    }
});

// Handle affiliate joining
add_hook('ClientAdd', 1, function(array $vars) {
    $userId = $vars['userid'];
    $referralCode = $_SESSION['referral_code'] ?? '';

    if ($referralCode) {
        $manager = new \WHMCS\Module\Addon\{Affiliate}\AffiliateManager();
        $manager->registerAffiliate($userId, $referralCode);

        unset($_SESSION['referral_code']);
    }
});
```

## Database Schema

### mod_{affiliate}_affiliates
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| user_id | INT | WHMCS client ID |
| affiliate_code | VARCHAR(50) | Unique referral code |
| referral_code | VARCHAR(50) | Who referred them |
| commission_rate | DECIMAL(5,2) | Custom commission % |
| status | VARCHAR(20) | pending/active/rejected |
| notes | TEXT | Admin notes |
| approved_at | TIMESTAMP | Approval time |
| approved_by | INT | Admin who approved |

### mod_{affiliate}_commissions
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| affiliate_id | INT | Affiliate FK |
| referral_id | INT | Referral FK |
| order_id | INT | Order FK |
| client_id | INT | Client FK |
| order_amount | DECIMAL(15,2) | Order total |
| commission_amount | DECIMAL(15,2) | Commission earned |
| commission_type | VARCHAR(20) | percentage/fixed/both |
| status | VARCHAR(20) | pending/approved/payable/paid |
| payable_at | TIMESTAMP | When commission becomes payable |
| paid_at | TIMESTAMP | When paid |
| payout_id | INT | Payout FK |

### mod_{affiliate}_tiers
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| name | VARCHAR(100) | Tier name |
| level | INT | Tier level |
| commission_rate | DECIMAL(5,2) | Commission % |
| min_referrals | INT | Min referrals for tier |
| is_active | BOOLEAN | Active status |

## Checklist

```
Pre-Dev:
□ Define commission structure
□ Plan tier system
□ Design payout methods
□ Choose tracking method (cookie/URL)

Development:
□ Implement config() with all settings
□ Implement activate() → Create all tables
□ Implement deactivate() → Drop tables
□ Create AffiliateManager class
□ Create CommissionCalculator class
□ Create PayoutManager class
□ Implement affiliate registration
□ Implement click tracking
□ Implement conversion tracking
□ Create commission calculation
□ Build admin interface
□ Create affiliate dashboard
□ Add hooks for order integration

Security:
□ Validate affiliate codes
□ Sanitize all inputs
□ Use check_token() for POST
□ Prevent self-referral
□ Secure commission amounts

Testing:
□ Test affiliate registration
□ Test click tracking
□ Test conversion tracking
□ Test commission calculation
□ Test tier upgrades
□ Test payout processing
□ Verify hook integration
```
