# WHMCS Affiliate Payout Workflow

## Overview
Automated affiliate tracking, commission calculation, and payout processing.

## Prerequisites
- WHMCS v8.0+
- WHMCS Affiliate add-on

## Step-by-Step Guide

### Step 1: Affiliate Configuration
```php
<?php
namespace Vendor\Module;

class AffiliatePayoutService
{
    protected float $commissionRate = 0.20; // 20%
    protected int $minimumPayout = 50;

    public function calculateCommission(int $affiliateId, float $orderTotal): float
    {
        return $orderTotal * $this->commissionRate;
    }

    public function processAffiliateCommission(int $affiliateId, int $orderId): void
    {
        $order = \WHMCS\Billing\Order\Order::find($orderId);
        $commission = $this->calculateCommission($affiliateId, $order->total);

        \WHMCS\Database\Capsule::table('mod_yourmodule_affiliate_commissions')->insert([
            'affiliate_id' => $affiliateId,
            'order_id' => $orderId,
            'order_total' => $order->total,
            'commission' => $commission,
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function generatePayouts(): array
    {
        $affiliates = \WHMCS\Database\Capsule::table('mod_yourmodule_affiliate_commissions')
            ->select('affiliate_id')
            ->selectRaw('SUM(commission) as total_commission')
            ->where('status', 'pending')
            ->groupBy('affiliate_id')
            ->havingRaw('SUM(commission) >= ?', [$this->minimumPayout])
            ->get();

        $payouts = [];
        foreach ($affiliates as $affiliate) {
            $payouts[] = [
                'affiliate_id' => $affiliate->affiliate_id,
                'amount' => $affiliate->total_commission,
            ];
        }

        return $payouts;
    }
}
```

### Step 2: Payout Hook
```php
add_hook('OrderPaid', 1, function($vars) {
    $order = \WHMCS\Billing\Order\Order::with('client')->find($vars['orderId']);
    
    if ($order->client->affiliate_id) {
        $payout = new \Vendor\Module\AffiliatePayoutService();
        $payout->processAffiliateCommission($order->client->affiliate_id, $vars['orderId']);
    }
});

add_hook('AfterCronJob', 1, function() {
    $payout = new \Vendor\Module\AffiliatePayoutService();
    $payouts = $payout->generatePayouts();

    foreach ($payouts as $payoutData) {
        // Create payout records and process payments
    }
});
```

## Checklist
- Commissions calculated on orders
- Payouts generated automatically
- Payment methods processed
- Reports generated
