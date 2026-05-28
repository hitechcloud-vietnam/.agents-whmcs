# WHMCS Prorating Workflow

## Purpose

Handle prorated billing calculations when clients upgrade, downgrade, or change plans mid-billing cycle, ensuring accurate charges and credits.

## Prerequisites

- WHMCS with prorating addon module installed
- Products configured with multiple billing tiers
- Understanding of current billing cycle dates
- Clear proration policy documented

## Workflow Steps

### Step 1: Calculate Proration Details

Implement prorated amount calculation:

```php
// File: /includes/proration_calculator.php

class ProrationCalculator {
    
    public function calculateProration($hostingId, $newProductId, $effectiveDate = null) {
        $effectiveDate = $effectiveDate ?? date('Y-m-d');
        
        $hosting = Capsule::table('tblhosting')
            ->where('id', $hostingId)
            ->first();
        
        $oldProduct = Capsule::table('tblproducts')
            ->where('id', $hosting->packageid)
            ->first();
        
        $newProduct = Capsule::table('tblproducts')
            ->where('id', $newProductId)
            ->first();
        
        // Get current billing info
        $invoice = Capsule::table('tblinvoices')
            ->where('userid', $hosting->userid)
            ->where('status', 'Paid')
            ->where('duedate', '>', $effectiveDate)
            ->orderBy('duedate', 'desc')
            ->first();
        
        $billingCycle = $this->getBillingCycleDays($hosting->billingcycle);
        $cycleStart = $this->getCycleStartDate($hostingId);
        $cycleEnd = date('Y-m-d', strtotime($cycleStart . ' +' . $billingCycle . ' days'));
        
        // Calculate days remaining
        $totalDays = $billingCycle;
        $daysUsed = (strtotime($effectiveDate) - strtotime($cycleStart)) / 86400;
        $daysRemaining = $totalDays - $daysUsed;
        
        // Calculate proration
        $oldDailyRate = $oldProduct->recurringamount / $totalDays;
        $newDailyRate = $newProduct->recurringamount / $totalDays;
        
        $creditFromOld = $oldDailyRate * $daysRemaining;
        $chargeForNew = $newDailyRate * $daysRemaining;
        
        $prorationAmount = $chargeForNew - $creditFromOld;
        
        return [
            'hosting_id' => $hostingId,
            'old_product_id' => $hosting->packageid,
            'new_product_id' => $newProductId,
            'effective_date' => $effectiveDate,
            'billing_cycle_days' => $totalDays,
            'days_remaining' => $daysRemaining,
            'credit_amount' => round($creditFromOld, 2),
            'charge_amount' => round($chargeForNew, 2),
            'net_amount' => round($prorationAmount, 2),
            'next_full_billing_date' => $this->getNextBillingDate($effectiveDate, $billingCycle)
        ];
    }
    
    private function getBillingCycleDays($cycle) {
        $cycles = [
            'Monthly' => 30,
            'Quarterly' => 90,
            'SemiAnnually' => 180,
            'Annually' => 365,
            'Biennially' => 730
        ];
        return $cycles[$cycle] ?? 30;
    }
    
    private function getCycleStartDate($hostingId) {
        $invoice = Capsule::table('tblinvoices')
            ->join('tblinvoiceitems', 'tblinvoices.id', '=', 'tblinvoiceitems.invoiceid')
            ->where('tblinvoiceitems.relid', $hostingId)
            ->where('tblinvoices.status', 'Paid')
            ->orderBy('tblinvoices.datepaid', 'desc')
            ->first();
        
        return $invoice ? date('Y-m-d', strtotime($invoice->datepaid)) : date('Y-m-d');
    }
    
    private function getNextBillingDate($fromDate, $cycleDays) {
        return date('Y-m-d', strtotime($fromDate . ' +' . $cycleDays . ' days'));
    }
    
    public function generateProrationInvoice($prorationData) {
        // Create proration invoice
        $invoiceId = createInvoices($prorationData['hosting_id']);
        
        $invoice = new WHMCS\Invoice();
        $invoice->setStatus('Draft');
        
        // Add proration line item
        Capsule::table('tblinvoiceitems')->insert([
            'invoiceid' => $invoiceId,
            'userid' => $prorationData['hosting_id'],
            'type' => 'Proration',
            'relid' => $prorationData['hosting_id'],
            'description' => 'Proration: Upgrade from Plan A to Plan B',
            'amount' => $prorationData['net_amount'],
            'taxed' => 0
        ]);
        
        // Add credit from old plan if negative amount
        if ($prorationData['credit_amount'] > 0) {
            Capsule::table('tblinvoiceitems')->insert([
                'invoiceid' => $invoiceId,
                'userid' => $prorationData['hosting_id'],
                'type' => 'Credit',
                'relid' => $prorationData['hosting_id'],
                'description' => 'Credit for unused ' . $prorationData['days_remaining'] . ' days on old plan',
                'amount' => -$prorationData['credit_amount'],
                'taxed' => 0
            ]);
        }
        
        return $invoiceId;
    }
}
```

### Step 2: Create Proration Hook for Upgrades

Handle plan upgrade scenarios:

```php
// File: /includes/hooks/proration_upgrade.php

add_hook('DailyCronJob', 1, function($vars) {
    // Check for pending plan changes
    $pendingChanges = Capsule::table('mod_plan_changes')
        ->where('status', 'pending')
        ->where('change_type', 'upgrade')
        ->where('effective_date', '<=', date('Y-m-d'))
        ->get();
    
    foreach ($pendingChanges as $change) {
        $calculator = new ProrationCalculator();
        
        // Calculate proration
        $proration = $calculator->calculateProration(
            $change->hosting_id,
            $change->new_product_id,
            $change->effective_date
        );
        
        if ($proration['net_amount'] > 0) {
            // Create proration invoice
            $invoiceId = $calculator->generateProrationInvoice($proration);
            
            // Mark plan change as pending payment
            Capsule::table('mod_plan_changes')
                ->where('id', $change->id)
                ->update([
                    'proration_invoice_id' => $invoiceId,
                    'proration_amount' => $proration['net_amount']
                ]);
        } else {
            // Process immediately if credit exceeds charge
            processUpgrade($change->hosting_id, $change->new_product_id);
            
            Capsule::table('mod_plan_changes')
                ->where('id', $change->id)
                ->update(['status' => 'completed']);
        }
    }
});

function processUpgrade($hostingId, $newProductId) {
    // Update hosting record
    Capsule::table('tblhosting')
        ->where('id', $hostingId)
        ->update(['packageid' => $newProductId]);
    
    // Run module change package
    $params = Capsule::table('tblhosting')
        ->where('id', $hostingId)
        ->first();
    
    $result = runModuleHook('ChangePackage', $hostingId);
    
    // Log the upgrade
    logActivity("Service {$hostingId} upgraded to product {$newProductId}");
    
    // Send confirmation email
    $hosting = Capsule::table('tblhosting')->where('id', $hostingId)->first();
    sendTemplatedEmail('ServiceUpgradeConfirmation', $hosting->userid, [
        'product_name' => Capsule::table('tblproducts')
            ->where('id', $newProductId)
            ->value('name'),
        'effective_date' => date('Y-m-d')
    ]);
}
```

### Step 3: Handle Downgrades with Credit

Process plan downgrades:

```php
// File: /includes/hooks/proration_downgrade.php

add_hook('ServiceChangePackage', 1, function($vars) {
    $hostingId = $vars['hosting_id'];
    $oldProductId = $vars['old_product_id'];
    $newProductId = $vars['new_product_id'];
    
    $calculator = new ProrationCalculator();
    $proration = $calculator->calculateProration($hostingId, $newProductId);
    
    if ($proration['credit_amount'] > $proration['charge_amount']) {
        // Credit exceeds charge - apply credit to account
        $creditAmount = $proration['credit_amount'] - $proration['charge_amount'];
        
        // Add credit to client
        Capsule::table('tblcredits')->insert([
            'clientid' => $vars['userid'],
            'amount' => $creditAmount,
            'description' => 'Plan downgrade credit - ' . date('Y-m-d'),
            'date' => date('Y-m-d H:i:s'),
            'remaining' => $creditAmount
        ]);
        
        // Process change immediately
        processDowngrade($hostingId, $newProductId);
        
        sendTemplatedEmail('PlanDowngradeProcessed', $vars['userid'], [
            'credit_amount' => $creditAmount,
            'credit_remaining' => $creditAmount
        ]);
    } else {
        // Charge exceeds credit - create invoice
        $invoiceId = $calculator->generateProrationInvoice($proration);
        
        Capsule::table('mod_plan_changes')->insert([
            'hosting_id' => $hostingId,
            'old_product_id' => $oldProductId,
            'new_product_id' => $newProductId,
            'change_type' => 'downgrade',
            'effective_date' => date('Y-m-d', strtotime('+7 days')), // Delay for payment
            'proration_amount' => abs($proration['net_amount']),
            'status' => 'pending_payment'
        ]);
        
        sendTemplatedEmail('PlanDowngradePending', $vars['userid'], [
            'proration_amount' => abs($proration['net_amount']),
            'payment_due_date' => date('Y-m-d', strtotime('+7 days'))
        ]);
    }
});

function processDowngrade($hostingId, $newProductId) {
    Capsule::table('tblhosting')
        ->where('id', $hostingId)
        ->update(['packageid' => $newProductId]);
    
    logActivity("Service {$hostingId} downgraded to product {$newProductId}");
}
```

### Step 4: Proration Configuration Hook

Customize proration behavior:

```php
// File: /includes/hooks/proration_config.php

add_hook('AdminAreaPageStart', 1, function($vars) {
    // Add proration settings to admin
    if (isset($_GET['action']) && $_GET['action'] === 'product-config') {
        return [
            'proration_enabled' => true,
            'proration_type' => 'daily', // or 'full_cycle'
            'minimum_proration_days' => 1,
            'round_proration_to' => 2 // decimal places
        ];
    }
});

// Configuration options for proration settings
function getProrationSettings() {
    return Capsule::table('tblconfiguration')
        ->whereIn('setting', [
            'ProrationEnabled',
            'ProrationType',
            'MinimumDaysForProration',
            'ProrationRoundDecimal'
        ])
        ->pluck('value', 'setting')
        ->toArray();
}

function saveProrationSettings($settings) {
    foreach ($settings as $key => $value) {
        Capsule::table('tblconfiguration')
            ->updateOrInsert(
                ['setting' => $key],
                ['value' => $value]
            );
    }
}
```

### Step 5: Test Proration Calculations

Verify proration accuracy:

```php
// File: /resources/testing/ProrationTest.php

namespace WHMCS\Testing;

class ProrationCalculatorTest extends \PHPUnit\Framework\TestCase {
    
    public function testMonthlyProrationHalfway() {
        $calculator = new \ProrationCalculator();
        
        // Test 15 days into a 30-day cycle
        $result = $calculator->calculateProration(
            $hostingId = 1,
            $newProductId = 2,
            '2024-01-15' // Midway through cycle
        );
        
        // Should have 15 days remaining
        $this->assertEquals(15, $result['days_remaining']);
        $this->assertEquals(30, $result['billing_cycle_days']);
    }
    
    public function testUpgradeChargeCalculation() {
        $calculator = new \ProrationCalculator();
        
        // $30/mo to $60/mo plan, 15 days remaining
        // Old daily rate: $30/30 = $1/day * 15 = $15 credit
        // New daily rate: $60/30 = $2/day * 15 = $30 charge
        // Net: $30 - $15 = $15 charge
        
        $result = $calculator->calculateProration(1, 2, date('Y-m-d'));
        
        $this->assertGreaterThan(0, $result['net_amount']);
    }
    
    public function testDowngradeCreditCalculation() {
        $calculator = new \ProrationCalculator();
        
        // $60/mo to $30/mo plan, 15 days remaining
        // Should result in credit (negative charge)
        $result = $calculator->calculateProration(1, 2, date('Y-m-d'));
        
        $this->assertNotEquals(0, $result['net_amount']);
    }
}

// Run tests
// php -q /var/www/whmcs/vendor/bin/phpunit resources/testing/ProrationTest.php
```

## Verification Checklist

- [ ] Proration calculation accurate for different scenarios
- [ ] Upgrade charges calculated correctly
- [ ] Downgrade credits applied properly
- [ ] Proration invoices generated and sent
- [ ] Credit balance updated for negative proration
- [ ] Plan changes processed after payment
- [ ] Email notifications sent at each step
- [ ] Proration reports accurate
- [ ] Edge cases handled (1 day remaining, etc.)
- [ ] Proration configuration working in admin panel

## Related Skills and Documentation

- [WHMCS Subscription Workflow](whmcs-subscription-workflow.md)
- [WHMCS Invoice Automation](whmcs-invoice-automation-workflow.md)
- [WHMCS Payment Processing](whmcs-payment-processing-workflow.md)
- WHMCS Documentation: Prorated Billing
- WHMCS Documentation: Product Changes

## Notes

- Document proration policy clearly for clients
- Consider minimum thresholds for small proration amounts
- Test various billing cycle scenarios thoroughly
- Handle timezone differences in proration calculations
- Consider grace periods for mid-cycle changes
- Store proration history for audit and disputes