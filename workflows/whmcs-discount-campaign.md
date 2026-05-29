# WHMCS Discount Campaign Workflow

## Overview
Comprehensive workflow for creating and managing discount campaigns with automated coupon generation, tracking, and analytics.

## Prerequisites
- WHMCS v8.0+
- Marketing automation module

## Step-by-Step Guide

### Step 1: Create Campaign
```php
<?php
class DiscountCampaign
{
    public int $id;
    public string $name;
    public string $type; // percentage, fixed, bogo
    public float $value;
    public DateTime $startDate;
    public DateTime $endDate;
    public int $maxUses;
    public int $currentUses = 0;

    public function create(): int
    {
        return \WHMCS\Database\Capsule::table('mod_yourmodule_campaigns')->insertGetId([
            'name' => $this->name,
            'type' => $this->type,
            'value' => $this->value,
            'start_date' => $this->startDate->format('Y-m-d H:i:s'),
            'end_date' => $this->endDate->format('Y-m-d H:i:s'),
            'max_uses' => $this->maxUses,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

### Step 2: Coupon Generation
```php
<?php
class CouponGenerator
{
    public function generate(int $campaignId, int $quantity = 1): array
    {
        $campaign = \WHMCS\Database\Capsule::table('mod_yourmodule_campaigns')
            ->where('id', $campaignId)
            ->first();

        $coupons = [];
        for ($i = 0; $i < $quantity; $i++) {
            $code = $this->generateCode($campaign->name);
            
            $couponId = \WHMCS\Database\Capsule::table('mod_yourmodule_coupons')->insertGetId([
                'campaign_id' => $campaignId,
                'code' => $code,
                'uses' => 0,
                'created_at' => date('Y-m-d H:i:s'),
            ]);

            $coupons[] = ['id' => $couponId, 'code' => $code];
        }

        return $coupons;
    }

    private function generateCode(string $name): string
    {
        $prefix = strtoupper(substr($name, 0, 4));
        $random = strtoupper(substr(md5(uniqid()), 0, 6));
        return "{$prefix}{$random}";
    }
}
```

### Step 3: Apply Discount to Cart
```php
<?php
add_hook('ShoppingCartValidateCheckout', 1, function($vars) {
    $couponCode = $_POST['couponcode'] ?? null;
    
    if (!$couponCode) {
        return;
    }

    $coupon = \WHMCS\Database\Capsule::table('mod_yourmodule_coupons')
        ->join('mod_yourmodule_campaigns', 'mod_yourmodule_coupons.campaign_id', '=', 'mod_yourmodule_campaigns.id')
        ->where('mod_yourmodule_coupons.code', $couponCode)
        ->where('mod_yourmodule_campaigns.start_date', '<=', date('Y-m-d H:i:s'))
        ->where('mod_yourmodule_campaigns.end_date', '>=', date('Y-m-d H:i:s'))
        ->where('mod_yourmodule_campaigns.max_uses', '>', \WHMCS\Database\Capsule::raw('mod_yourmodule_coupons.uses'))
        ->first();

    if (!$coupon) {
        return ['error' => 'Invalid or expired coupon code'];
    }

    // Calculate discount
    $cartTotal = \WHMCS\Session::get('cart_total');
    $discount = $this->calculateDiscount($coupon, $cartTotal);

    return [
        'success' => true,
        'discount' => $discount,
        'coupon_id' => $coupon->id,
    ];
});
```

### Step 4: Campaign Analytics
```php
<?php
class CampaignAnalytics
{
    public function getCampaignStats(int $campaignId): array
    {
        $campaign = \WHMCS\Database\Capsule::table('mod_yourmodule_campaigns')
            ->where('id', $campaignId)
            ->first();

        $redemptions = \WHMCS\Database\Capsule::table('mod_yourmodule_coupon_usage')
            ->where('campaign_id', $campaignId)
            ->count();

        $totalDiscount = \WHMCS\Database\Capsule::table('mod_yourmodule_coupon_usage')
            ->where('campaign_id', $campaignId)
            ->sum('discount_amount');

        return [
            'campaign_name' => $campaign->name,
            'redemptions' => $redemptions,
            'max_uses' => $campaign->max_uses,
            'utilization_rate' => $campaign->max_uses > 0 
                ? round(($redemptions / $campaign->max_uses) * 100, 2) 
                : 0,
            'total_discount_given' => $totalDiscount,
            'start_date' => $campaign->start_date,
            'end_date' => $campaign->end_date,
            'status' => $this->getCampaignStatus($campaign),
        ];
    }
}
```

## Campaign Workflow
1. Create campaign with discount parameters
2. Generate unique coupon codes
3. Distribute coupons via email/social
4. Track coupon usage automatically
5. Monitor analytics and adjust as needed
