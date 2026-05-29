# WHMCS New Product Launch Workflow

## Overview
Comprehensive workflow for launching new products with marketing automation, pricing strategies, and launch campaigns.

## Prerequisites
- WHMCS v8.0+
- Product configuration access

## Step-by-Step Guide

### Step 1: Product Configuration
```php
<?php
// Configure new product
function configure_new_product(array $productData): int
{
    $productId = \WHMCS\Database\Capsule::table('tblproducts')->insertGetId([
        'type' => 'hostingaccount',
        'name' => $productData['name'],
        'description' => $productData['description'],
        'gid' => $productData['group_id'],
        'stock' => $productData['stock'] ?? 0,
        'stockcontrol' => $productData['stock'] ? 1 : 0,
        'paytype' => 'recurring',
        'monthly' => $productData['monthly_price'],
        'quarterly' => $productData['quarterly_price'],
        'semiannually' => $productData['semi_annual_price'],
        'annually' => $productData['annual_price'],
        'biennially' => $productData['biennial_price'],
        'tax' => 1,
        'hidden' => $productData['hidden'] ?? false,
        'showorderboxes' => 1,
    ]);

    return $productData;
}
```

### Step 2: Launch Campaign Automation
```php
<?php
class ProductLaunchCampaign
{
    public function __construct(
        private int $productId,
        private array $config
    ) {}

    public function execute(): void
    {
        // Phase 1: Pre-launch (1 week before)
        if ($this->config['send_announcement']) {
            $this->sendAnnouncement();
        }

        // Phase 2: Launch day
        $this->activateProduct();
        
        if ($this->config['send_launch_email']) {
            $this->sendLaunchEmail();
        }

        // Phase 3: Post-launch follow-up
        if ($this->config['send_reminder']) {
            $this->scheduleReminder();
        }
    }

    private function sendAnnouncement(): void
    {
        $product = \WHMCS\Products\Product::find($this->productId);
        
        // Get all active clients
        $clients = \WHMCS\User\Client::where('status', 'Active')->get();
        
        foreach ($clients as $client) {
            send_email($client->email, 'Product Announcement', [
                'product_name' => $product->name,
                'launch_date' => $this->config['launch_date'],
                'early_bird' => $this->config['early_bird_discount'] ?? 0,
            ]);
        }
    }

    private function activateProduct(): void
    {
        \WHMCS\Database\Capsule::table('tblproducts')
            ->where('id', $this->productId)
            ->update(['hidden' => false]);
    }

    private function scheduleReminder(): void
    {
        $reminderDate = date('Y-m-d H:i:s', strtotime('+3 days'));
        
        \WHMCS\Database\Capsule::table('mod_yourmodule_campaign_scheduled')->insert([
            'product_id' => $this->productId,
            'type' => 'reminder',
            'scheduled_for' => $reminderDate,
        ]);
    }
}
```

### Step 3: Early Bird Pricing
```php
<?php
class EarlyBirdPricing
{
    private int $productId;
    private float $discount;
    private DateTime $endDate;

    public function applyEarlyBirdPricing(): void
    {
        $product = \WHMCS\Products\Product::find($this->productId);
        
        // Store original prices
        $this->storeOriginalPrices();
        
        // Apply discount
        $discountedPrices = $this->calculateDiscountedPrices();
        
        foreach ($discountedPrices as $cycle => $price) {
            \WHMCS\Database\Capsule::table('tblpricing')
                ->where('type', 'product')
                ->where('relid', $this->productId)
                ->update([$cycle => $price]);
        }
    }

    private function calculateDiscountedPrices(): array
    {
        $prices = \WHMCS\Pricing::product($this->productId);
        $discountFactor = 1 - ($this->discount / 100);

        return [
            'monthly' => $prices->monthly * $discountFactor,
            'quarterly' => $prices->quarterly * $discountFactor,
            'semiannually' => $prices->semiannually * $discountFactor,
            'annually' => $prices->annually * $discountFactor,
        ];
    }

    public function restoreOriginalPrices(): void
    {
        $original = \WHMCS\Database\Capsule::table('mod_yourmodule_original_prices')
            ->where('product_id', $this->productId)
            ->first();

        if ($original) {
            \WHMCS\Database\Capsule::table('tblpricing')
                ->where('type', 'product')
                ->where('relid', $this->productId)
                ->update([
                    'monthly' => $original->monthly,
                    'quarterly' => $original->quarterly,
                    'semiannually' => $original->semiannually,
                    'annually' => $original->annually,
                ]);
        }
    }
}
```

## Launch Checklist
- Product configured and tested
- Pricing set up with early bird option
- Announcement sent to existing clients
- Launch email prepared
- Monitoring dashboard ready
- Support team briefed
