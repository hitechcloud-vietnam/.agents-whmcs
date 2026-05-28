# WHMCS Coupon Campaign Workflow

## Purpose

Create and manage promotional coupon campaigns, handle discount codes, track campaign effectiveness, and analyze coupon usage patterns.

## Prerequisites

- WHMCS with coupon/promo code functionality
- Admin access to marketing/promotions
- Email system configured for campaign communications
- Analytics/tracking system (optional)

## Workflow Steps

### Step 1: Configure Coupon Settings

Set up coupon configuration:

```php
// Database configuration for coupons
INSERT INTO tblconfiguration (setting, value) VALUES 
('CouponsEnabled', 'on'),
('CouponMinOrderAmount', '10.00'),
('CouponMaxUsesPerClient', '3'),
('CouponCodeLength', '8'),
('AutoExpireCoupons', 'on');

// Create coupon tracking table
Capsule::schema()->create('mod_coupon_campaigns', function($t) {
    $t->increments('id');
    $t->string('name');
    $t->string('code');
    $t->string('type'); // percentage, fixed, trial
    $t->decimal('value', 10, 2);
    $t->decimal('min_order', 10, 2)->default(0);
    $t->integer('max_uses')->default(0);
    $t->integer('uses')->default(0);
    $t->integer('max_uses_per_client')->default(1);
    $t->date('start_date');
    $t->date('end_date');
    $t->json('applies_to'); // product IDs or categories
    $t->boolean('is_active')->default(true);
    $t->timestamp('created_at')->default(Capsule::raw('CURRENT_TIMESTAMP'));
});

Capsule::schema()->create('mod_coupon_usage', function($t) {
    $t->increments('id');
    $t->integer('campaign_id');
    $t->integer('client_id');
    $t->integer('invoice_id');
    $t->decimal('discount_amount', 10, 2);
    $t->timestamp('used_at')->default(Capsule::raw('CURRENT_TIMESTAMP'));
});
```

### Step 2: Create Coupon Management Class

Build coupon functionality:

```php
// File: /includes/classes/CouponManager.php

namespace WHMCS\Marketing;

class CouponManager {
    
    public function createCoupon($data) {
        // Validate input
        $this->validateCouponData($data);
        
        // Generate unique code if not provided
        if (empty($data['code'])) {
            $data['code'] = $this->generateCouponCode();
        }
        
        // Create coupon
        $campaignId = Capsule::table('mod_coupon_campaigns')->insertGetId([
            'name' => $data['name'],
            'code' => strtoupper($data['code']),
            'type' => $data['type'] ?? 'percentage',
            'value' => $data['value'],
            'min_order' => $data['min_order'] ?? 0,
            'max_uses' => $data['max_uses'] ?? 0,
            'max_uses_per_client' => $data['max_uses_per_client'] ?? 1,
            'start_date' => $data['start_date'] ?? date('Y-m-d'),
            'end_date' => $data['end_date'] ?? null,
            'applies_to' => json_encode($data['applies_to'] ?? []),
            'is_active' => true
        ]);
        
        logActivity("Coupon campaign created: {$data['code']}");
        
        return $campaignId;
    }
    
    private function generateCouponCode($length = 8) {
        $chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';
        $code = '';
        for ($i = 0; $i < $length; $i++) {
            $code .= $chars[random_int(0, strlen($chars) - 1)];
        }
        
        // Ensure unique
        while (Capsule::table('mod_coupon_campaigns')
            ->where('code', $code)
            ->exists()) {
            $code = $this->generateCouponCode();
        }
        
        return $code;
    }
    
    public function validateCoupon($code, $clientId, $orderAmount, $productIds = []) {
        $coupon = Capsule::table('mod_coupon_campaigns')
            ->where('code', strtoupper($code))
            ->where('is_active', true)
            ->first();
        
        if (!$coupon) {
            return ['valid' => false, 'error' => 'Invalid coupon code'];
        }
        
        // Check dates
        if ($coupon->start_date && date('Y-m-d') < $coupon->start_date) {
            return ['valid' => false, 'error' => 'Coupon not yet active'];
        }
        
        if ($coupon->end_date && date('Y-m-d') > $coupon->end_date) {
            return ['valid' => false, 'error' => 'Coupon has expired'];
        }
        
        // Check max uses
        if ($coupon->max_uses > 0 && $coupon->uses >= $coupon->max_uses) {
            return ['valid' => false, 'error' => 'Coupon usage limit reached'];
        }
        
        // Check client usage
        $clientUsage = Capsule::table('mod_coupon_usage')
            ->where('campaign_id', $coupon->id)
            ->where('client_id', $clientId)
            ->count();
        
        if ($clientUsage >= $coupon->max_uses_per_client) {
            return ['valid' => false, 'error' => 'You have already used this coupon'];
        }
        
        // Check minimum order
        if ($coupon->min_order > 0 && $orderAmount < $coupon->min_order) {
            return ['valid' => false, 'error' => "Minimum order of {$coupon->min_order} required"];
        }
        
        // Check applies to products
        if (!empty($productIds) && !empty($coupon->applies_to)) {
            $appliesTo = json_decode($coupon->applies_to, true);
            $hasValidProduct = false;
            
            foreach ($productIds as $productId) {
                if (in_array($productId, $appliesTo)) {
                    $hasValidProduct = true;
                    break;
                }
            }
            
            if (!$hasValidProduct) {
                return ['valid' => false, 'error' => 'Coupon not valid for selected products'];
            }
        }
        
        // Calculate discount
        $discount = $this->calculateDiscount($coupon, $orderAmount);
        
        return [
            'valid' => true,
            'discount' => $discount,
            'coupon_id' => $coupon->id,
            'coupon_name' => $coupon->name
        ];
    }
    
    private function calculateDiscount($coupon, $orderAmount) {
        if ($coupon->type === 'percentage') {
            return $orderAmount * ($coupon->value / 100);
        } else {
            return min($coupon->value, $orderAmount);
        }
    }
    
    public function redeemCoupon($code, $clientId, $invoiceId, $discountAmount) {
        $coupon = Capsule::table('mod_coupon_campaigns')
            ->where('code', strtoupper($code))
            ->first();
        
        // Record usage
        Capsule::table('mod_coupon_usage')->insert([
            'campaign_id' => $coupon->id,
            'client_id' => $clientId,
            'invoice_id' => $invoiceId,
            'discount_amount' => $discountAmount,
            'used_at' => Capsule::raw('NOW()')
        ]);
        
        // Update usage count
        Capsule::table('mod_coupon_campaigns')
            ->where('id', $coupon->id)
            ->increment('uses');
        
        logActivity("Coupon {$code} redeemed for invoice #{$invoiceId}: discount of {$discountAmount}");
    }
}
```

### Step 3: Create Coupon Validation Hook

Integrate coupon validation into checkout:

```php
// File: /includes/hooks/coupon_validation.php

add_hook('CheckoutCompleted', 1, function($vars) {
    if (isset($vars['coupon_code']) && !empty($vars['coupon_code'])) {
        $couponManager = new \WHMCS\Marketing\CouponManager();
        
        $productIds = array_map(function($item) {
            return $item['product_id'];
        }, $vars['items'] ?? []);
        
        $result = $couponManager->validateCoupon(
            $vars['coupon_code'],
            $vars['client_id'],
            $vars['subtotal'],
            $productIds
        );
        
        if ($result['valid']) {
            $couponManager->redeemCoupon(
                $vars['coupon_code'],
                $vars['client_id'],
                $vars['invoice_id'],
                $result['discount']
            );
        }
    }
});

add_hook('OrderFormCheckoutCompletePage', 1, function($vars) {
    // Display applied coupon in checkout
    $sessionCoupon = $_SESSION['coupon_code'] ?? null;
    
    if ($sessionCoupon) {
        return [
            'applied_coupon' => $sessionCoupon,
            'discount_amount' => $_SESSION['coupon_discount'] ?? 0
        ];
    }
});
```

### Step 4: Create Coupon Campaign Admin Interface

Add admin functionality:

```php
// File: /modules/addons/coupon_manager/admin.php

function coupon_manager_config() {
    return [
        'name' => 'Coupon Campaign Manager',
        'description' => 'Manage promotional coupons and campaigns',
        'version' => '1.0',
        'author' => 'Your Company',
    ];
}

function coupon_manager_output($vars) {
    $action = $_GET['action'] ?? 'list';
    
    echo '<div class="coupon-manager">';
    
    switch ($action) {
        case 'create':
            echo couponCreationForm();
            break;
        case 'analytics':
            echo couponAnalytics();
            break;
        case 'list':
        default:
            echo couponList();
            break;
    }
    
    echo '</div>';
}

function couponCreationForm() {
    $output = '<h2>Create Coupon Campaign</h2>';
    $output .= '<form method="post" action="?action=save">';
    $output .= '<input type="hidden" name="token" value="' . generate_token() . '">';
    
    $output .= '<div class="form-group">';
    $output .= '<label>Campaign Name</label>';
    $output .= '<input type="text" name="name" required>';
    $output .= '</div>';
    
    $output .= '<div class="form-group">';
    $output .= '<label>Coupon Type</label>';
    $output .= '<select name="type">';
    $output .= '<option value="percentage">Percentage Discount</option>';
    $output .= '<option value="fixed">Fixed Amount</option>';
    $output .= '</select>';
    $output .= '</div>';
    
    $output .= '<div class="form-group">';
    $output .= '<label>Value</label>';
    $output .= '<input type="number" name="value" step="0.01" required>';
    $output .= '</div>';
    
    $output .= '<div class="form-group">';
    $output .= '<label>Start Date</label>';
    $output .= '<input type="date" name="start_date">';
    $output .= '</div>';
    
    $output .= '<div class="form-group">';
    $output .= '<label>End Date</label>';
    $output .= '<input type="date" name="end_date">';
    $output .= '</div>';
    
    $output .= '<button type="submit" class="btn btn-primary">Create Campaign</button>';
    $output .= '</form>';
    
    return $output;
}

function couponList() {
    $coupons = Capsule::table('mod_coupon_campaigns')
        ->orderBy('created_at', 'desc')
        ->limit(50)
        ->get();
    
    $output = '<h2>Active Campaigns</h2>';
    $output .= '<table class="data-table">';
    $output .= '<thead><tr>';
    $output .= '<th>Code</th><th>Name</th><th>Type</th><th>Value</th>';
    $output .= '<th>Uses</th><th>Start</th><th>End</th><th>Status</th>';
    $output .= '</tr></thead><tbody>';
    
    foreach ($coupons as $coupon) {
        $output .= '<tr>';
        $output .= '<td><strong>' . $coupon->code . '</strong></td>';
        $output .= '<td>' . htmlspecialchars($coupon->name) . '</td>';
        $output .= '<td>' . ucfirst($coupon->type) . '</td>';
        $output .= '<td>' . $coupon->value . '</td>';
        $output .= '<td>' . $coupon->uses . ($coupon->max_uses ? '/' . $coupon->max_uses : '') . '</td>';
        $output .= '<td>' . $coupon->start_date . '</td>';
        $output .= '<td>' . ($coupon->end_date ?: 'No limit') . '</td>';
        $output .= '<td>' . ($coupon->is_active ? 'Active' : 'Inactive') . '</td>';
        $output .= '</tr>';
    }
    
    $output .= '</tbody></table>';
    
    return $output;
}

function couponAnalytics() {
    $stats = Capsule::table('mod_coupon_usage')
        ->selectRaw('campaign_id, COUNT(*) as uses, SUM(discount_amount) as total_discount')
        ->groupBy('campaign_id')
        ->get();
    
    $output = '<h2>Coupon Analytics</h2>';
    $output .= '<table class="data-table">';
    $output .= '<thead><tr>';
    $output .= '<th>Campaign</th><th>Code</th><th>Uses</th><th>Total Discount</th>';
    $output .= '<th>Avg Discount</th>';
    $output .= '</tr></thead><tbody>';
    
    foreach ($stats as $stat) {
        $campaign = Capsule::table('mod_coupon_campaigns')
            ->where('id', $stat->campaign_id)
            ->first();
        
        $output .= '<tr>';
        $output .= '<td>' . htmlspecialchars($campaign->name) . '</td>';
        $output .= '<td>' . $campaign->code . '</td>';
        $output .= '<td>' . $stat->uses . '</td>';
        $output .= '<td>$' . number_format($stat->total_discount, 2) . '</td>';
        $output .= '<td>$' . number_format($stat->total_discount / $stat->uses, 2) . '</td>';
        $output .= '</tr>';
    }
    
    $output .= '</tbody></table>';
    
    return $output;
}
```

### Step 5: Coupon Email Campaign Integration

Send promotional emails:

```php
// File: /includes/hooks/coupon_email_campaigns.php

function sendCouponCampaignEmail($clientIds, $campaignId) {
    $campaign = Capsule::table('mod_coupon_campaigns')
        ->where('id', $campaignId)
        ->first();
    
    foreach ($clientIds as $clientId) {
        $client = Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();
        
        $templateVars = [
            'client_name' => $client->firstname,
            'coupon_code' => $campaign->code,
            'discount_value' => $campaign->type === 'percentage' 
                ? $campaign->value . '%' 
                : '$' . $campaign->value,
            'expiry_date' => $campaign->end_date,
            'min_order' => $campaign->min_order
        ];
        
        sendTemplatedEmail('CouponCampaign', $clientId, $templateVars);
    }
    
    logActivity("Coupon campaign email sent to " . count($clientIds) . " clients");
}

add_hook('DailyCronJob', 1, function($vars) {
    // Check for expiring coupons and send reminders
    $threeDaysFromNow = date('Y-m-d', strtotime('+3 days'));
    
    $expiringCoupons = Capsule::table('mod_coupon_campaigns')
        ->where('end_date', $threeDaysFromNow)
        ->where('is_active', true)
        ->get();
    
    foreach ($expiringCoupons as $coupon) {
        $recentRecipients = Capsule::table('mod_coupon_usage')
            ->where('campaign_id', $coupon->id)
            ->where('used_at', '>', date('Y-m-d', strtotime('-7 days')))
            ->pluck('client_id')
            ->toArray();
        
        if (!empty($recentRecipients)) {
            foreach ($recentRecipients as $clientId) {
                sendTemplatedEmail('CouponExpiringReminder', $clientId, [
                    'coupon_code' => $coupon->code,
                    'expiry_date' => $coupon->end_date
                ]);
            }
        }
    }
});
```

## Verification Checklist

- [ ] Coupon creation working in admin panel
- [ ] Coupon validation at checkout functioning
- [ ] Discount applied correctly to invoices
- [ ] Usage limits enforced per client
- [ ] Date restrictions working (start/end dates)
- [ ] Coupon analytics showing accurate data
- [ ] Email campaigns sent successfully
- [ ] Expiring coupon reminders sent
- [ ] Coupon codes unique and not duplicating
- [ ] Product/category restrictions working

## Related Skills and Documentation

- [WHMCS Invoice Automation](whmcs-invoice-automation-workflow.md)
- [WHMCS Payment Processing](whmcs-payment-processing-workflow.md)
- [WHMCS Billing Reporting](whmcs-billing-reporting-workflow.md)
- WHMCS Documentation: Promotional Coupons
- WHMCS Documentation: Discount Codes

## Notes

- Test all coupon scenarios before launching campaigns
- Track coupon effectiveness for future planning
- Set clear terms and conditions for coupons
- Monitor for coupon abuse and fraud
- Consider single-use vs multi-use coupon types
- Review campaign ROI after completion
- Keep coupon codes memorable but secure