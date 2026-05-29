# WHMCS Affiliate Functions

Complete reference for affiliate management functions in WHMCS.

## Overview

WHMCS provides comprehensive affiliate management including affiliate registration, commission tracking, and payout processing.

## Affiliate CRUD Operations

### createAffiliate()

Creates a new affiliate account.

```php
/**
 * Create a new affiliate
 * 
 * @param int $clientId Client ID
 * @param array $data Affiliate data
 * @return int Affiliate ID
 */
function createAffiliate(int $clientId, array $data = []): int
{
    return Capsule::table('tblaffiliates')->insertGetId([
        'clientid' => $clientId,
        'code' => generateAffiliateCode(),
        'date' => date('Y-m-d H:i:s'),
        'status' => 'Active',
        'commission' => $data['commission'] ?? 0,
        'paymonthly' => $data['paymonthly'] ?? 0,
        'noemptyaccounts' => $data['noemptyaccounts'] ?? 0,
    ]);
}
```

**Example:**
```php
$affiliateId = createAffiliate(123, [
    'commission' => 20, // 20% commission
    'paymonthly' => 1
]);
```

### getAffiliate()

Retrieves an affiliate by ID.

```php
/**
 * Get affiliate by ID
 * 
 * @param int $affiliateId Affiliate ID
 * @return array|null Affiliate data
 */
function getAffiliate(int $affiliateId): ?array
{
    $result = Capsule::table('tblaffiliates')
        ->where('id', $affiliateId)
        ->first();
    
    return $result ? (array) $result : null;
}
```

### getAffiliateByCode()

Retrieves an affiliate by referral code.

```php
/**
 * Get affiliate by code
 * 
 * @param string $code Affiliate code
 * @return array|null Affiliate data
 */
function getAffiliateByCode(string $code): ?array
{
    $result = Capsule::table('tblaffiliates')
        ->where('code', $code)
        ->first();
    
    return $result ? (array) $result : null;
}
```

**Example:**
```php
$affiliate = getAffiliateByCode('REF123ABC');

if ($affiliate) {
    echo "Affiliate: {$affiliate['clientid']}";
}
```

### updateAffiliate()

Updates an existing affiliate.

```php
/**
 * Update an affiliate
 * 
 * @param int $affiliateId Affiliate ID
 * @param array $data Updated data
 * @return bool Success status
 */
function updateAffiliate(int $affiliateId, array $data): bool
{
    return Capsule::table('tblaffiliates')
        ->where('id', $affiliateId)
        ->update($data) > 0;
}
```

**Example:**
```php
updateAffiliate(1, [
    'commission' => 25,
    'paymonthly' => 1
]);
```

## Affiliate Referrals

### addAffiliateReferral()

Records an affiliate referral.

```php
/**
 * Add affiliate referral
 * 
 * @param int $affiliateId Affiliate ID
 * @param int $orderId Order ID
 * @param int $clientId Referred client ID
 * @param float $amount Order amount
 * @return int Referral ID
 */
function addAffiliateReferral(
    int $affiliateId,
    int $orderId,
    int $clientId,
    float $amount
): int {
    $affiliate = getAffiliate($affiliateId);
    
    $commission = calculateAffiliateCommission($affiliateId, $amount);
    
    return Capsule::table('tblaffiliatesessions')->insertGetId([
        'affiliateid' => $affiliateId,
        'date' => date('Y-m-d H:i:s'),
        'ip' => getClientIp(),
        'referredby' => $clientId,
        'orderid' => $orderId,
        'amount' => $amount,
        'commission' => $commission,
    ]);
}
```

**Example:**
```php
addAffiliateReferral(1, 1001, 456, 99.99);
```

### getAffiliateReferrals()

Gets referrals for an affiliate.

```php
/**
 * Get affiliate referrals
 * 
 * @param int $affiliateId Affiliate ID
 * @param string $status Filter by status
 * @return array Referrals
 */
function getAffiliateReferrals(int $affiliateId, string $status = ''): array
{
    $query = Capsule::table('tblaffiliatesessions')
        ->where('affiliateid', $affiliateId)
        ->orderBy('date', 'desc');
    
    if ($status === 'pending') {
        $query->where('pending', 1);
    } elseif ($status === 'paid') {
        $query->where('pending', 0);
    }
    
    return $query->get()->toArray();
}
```

## Commission Calculation

### calculateAffiliateCommission()

Calculates commission for an affiliate.

```php
/**
 * Calculate affiliate commission
 * 
 * @param int $affiliateId Affiliate ID
 * @param float $amount Order amount
 * @return float Commission
 */
function calculateAffiliateCommission(int $affiliateId, float $amount): float
{
    $affiliate = getAffiliate($affiliateId);
    
    if (!$affiliate) {
        return 0;
    }
    
    $commissionRate = $affiliate['commission'] ?? Config\Setting::getValue('AffiliateCommission');
    
    // Check for product-specific commissions
    $customCommissions = Capsule::table('tblaffiliatecommissions')
        ->where('affiliateid', $affiliateId)
        ->get()
        ->keyBy('productid');
    
    return $amount * ($commissionRate / 100);
}
```

### processAffiliateCommissions()

Processes pending commissions for an affiliate.

```php
/**
 * Process affiliate commissions
 * 
 * @param int $affiliateId Affiliate ID
 * @param string $period Period (monthly, etc.)
 * @return array Results
 */
function processAffiliateCommissions(int $affiliateId, string $period = 'monthly'): array
{
    $pending = Capsule::table('tblaffiliatesessions')
        ->where('affiliateid', $affiliateId)
        ->where('pending', 1)
        ->get();
    
    $totalCommission = 0;
    
    foreach ($pending as $referral) {
        $totalCommission += $referral->commission;
        
        Capsule::table('tblaffiliatesessions')
            ->where('id', $referral->id)
            ->update(['pending' => 0]);
    }
    
    // Update affiliate balance
    if ($totalCommission > 0) {
        Capsule::table('tblaffiliates')
            ->where('id', $affiliateId)
            ->increment('balance', $totalCommission);
    }
    
    return [
        'affiliate_id' => $affiliateId,
        'referrals' => count($pending),
        'total_commission' => $totalCommission
    ];
}
```

## Affiliate Payments

### createAffiliatePayout()

Creates an affiliate payout record.

```php
/**
 * Create affiliate payout
 * 
 * @param int $affiliateId Affiliate ID
 * @param float $amount Payout amount
 * @param string $method Payment method
 * @return int Payout ID
 */
function createAffiliatePayout(int $affiliateId, float $amount, string $method = ''): int
{
    $affiliate = getAffiliate($affiliateId);
    
    if ($amount > $affiliate['balance']) {
        return 0; // Insufficient balance
    }
    
    $payoutId = Capsule::table('tblaffiliatepayouts')->insertGetId([
        'affiliateid' => $affiliateId,
        'date' => date('Y-m-d H:i:s'),
        'amount' => $amount,
        'method' => $method,
        'status' => 'Pending',
    ]);
    
    // Deduct from balance
    Capsule::table('tblaffiliates')
        ->where('id', $affiliateId)
        ->decrement('balance', $amount);
    
    return $payoutId;
}
```

### getAffiliatePayouts()

Gets payout history for an affiliate.

```php
/**
 * Get affiliate payouts
 * 
 * @param int $affiliateId Affiliate ID
 * @return array Payouts
 */
function getAffiliatePayouts(int $affiliateId): array
{
    return Capsule::table('tblaffiliatepayouts')
        ->where('affiliateid', $affiliateId)
        ->orderBy('date', 'desc')
        ->get()
        ->toArray();
}
```

## Affiliate Statistics

### getAffiliateStatistics()

Gets affiliate statistics.

```php
/**
 * Get affiliate statistics
 * 
 * @param int $affiliateId Affiliate ID
 * @return array Statistics
 */
function getAffiliateStatistics(int $affiliateId): array
{
    $affiliate = getAffiliate($affiliateId);
    
    // Total referrals
    $totalReferrals = Capsule::table('tblaffiliatesessions')
        ->where('affiliateid', $affiliateId)
        ->count();
    
    // Pending referrals
    $pendingReferrals = Capsule::table('tblaffiliatesessions')
        ->where('affiliateid', $affiliateId)
        ->where('pending', 1)
        ->count();
    
    // Total earned
    $totalEarned = Capsule::table('tblaffiliatesessions')
        ->where('affiliateid', $affiliateId)
        ->where('pending', 0)
        ->sum('commission');
    
    // Total paid out
    $totalPaid = Capsule::table('tblaffiliatepayouts')
        ->where('affiliateid', $affiliateId)
        ->sum('amount');
    
    // This month's referrals
    $monthStart = date('Y-m-01');
    $monthReferrals = Capsule::table('tblaffiliatesessions')
        ->where('affiliateid', $affiliateId)
        ->where('date', '>=', $monthStart)
        ->count();
    
    return [
        'affiliate_id' => $affiliateId,
        'balance' => $affiliate['balance'] ?? 0,
        'total_referrals' => $totalReferrals,
        'pending_referrals' => $pendingReferrals,
        'total_earned' => $totalEarned,
        'total_paid' => $totalPaid,
        'month_referrals' => $monthReferrals,
        'pending_commission' => $totalEarned - $totalPaid - $affiliate['balance']
    ];
}
```

**Example:**
```php
$stats = getAffiliateStatistics(1);

echo "Balance: $" . number_format($stats['balance'], 2) . "\n";
echo "Total Referrals: {$stats['total_referrals']}\n";
echo "This Month: {$stats['month_referrals']} referrals";
```

## Affiliate Link Generation

### generateAffiliateLink()

Generates an affiliate referral link.

```php
/**
 * Generate affiliate link
 * 
 * @param int $affiliateId Affiliate ID
 * @param string $baseUrl Base URL of WHMCS
 * @return string Affiliate link
 */
function generateAffiliateLink(int $affiliateId, string $baseUrl = ''): string
{
    $affiliate = getAffiliate($affiliateId);
    
    if (!$affiliate) {
        return '';
    }
    
    $baseUrl = $baseUrl ?: Config\Setting::getValue('SystemURL');
    
    return rtrim($baseUrl, '/') . '/aff.php?i=' . $affiliate['code'];
}
```

### trackAffiliateReferral()

Tracks a referral click.

```php
/**
 * Track affiliate referral
 * 
 * @param string $code Affiliate code
 * @return void
 */
function trackAffiliateReferral(string $code): void
{
    $affiliate = getAffiliateByCode($code);
    
    if (!$affiliate) {
        return;
    }
    
    Capsule::table('tblaffiliates')->where('id', $affiliate['id'])->increment('clicks');
    
    // Set cookie if not exists
    if (!isset($_COOKIE['AffiliateRef'])) {
        setcookie('AffiliateRef', $code, time() + (86400 * 30), '/');
        $_SESSION['AffiliateRef'] = $code;
    }
}
```

## Affiliate Configuration

### getAffiliateSettings()

Gets affiliate system settings.

```php
/**
 * Get affiliate settings
 * 
 * @return array Settings
 */
function getAffiliateSettings(): array
{
    return [
        'commission_rate' => Config\Setting::getValue('AffiliateCommission') ?? 0,
        'min_payout' => Config\Setting::getValue('AffiliateMinimumPayout') ?? 0,
        'payout_methods' => ['paypal', 'bank_transfer', 'custom'],
        'cookie_duration' => Config\Setting::getValue('AffiliateCookieDuration') ?? 30,
        'referral_days' => Config\Setting::getValue('AffiliateReferralDays') ?? 90,
    ];
}
```

## Affiliate Validation

```php
/**
 * Check if client can be affiliate
 * 
 * @param int $clientId Client ID
 * @return array Validation result
 */
function canBecomeAffiliate(int $clientId): array
{
    // Check if already an affiliate
    $existing = Capsule::table('tblaffiliates')
        ->where('clientid', $clientId)
        ->first();
    
    if ($existing) {
        return ['valid' => false, 'error' => 'Already an affiliate'];
    }
    
    // Check minimum order requirement
    $minOrders = Config\Setting::getValue('AffiliateMinimumOrders') ?? 0;
    
    $orderCount = Capsule::table('tblorders')
        ->where('userid', $clientId)
        ->where('status', 'Active')
        ->count();
    
    if ($orderCount < $minOrders) {
        return [
            'valid' => false,
            'error' => "Minimum {$minOrders} orders required"
        ];
    }
    
    return ['valid' => true];
}
```

## Best Practices

1. **Track all referrals** - Use cookies for 30-day tracking
2. **Calculate correctly** - Apply commission to actual paid orders only
3. **Process regularly** - Run monthly commission processing
4. **Maintain balances** - Keep accurate affiliate balance records
5. **Handle reversals** - Reverse commission on refunds
6. **Pay promptly** - Process payouts on schedule

## Related Functions

- [whmcs-functions-orders.md](whmcs-functions-orders.md) - Order tracking
- [whmcs-functions-transactions.md](whmcs-functions-transactions.md) - Payment processing