# WHMCS Tax Calculation Workflow

## Purpose

Configure and manage tax calculations in WHMCS, handle multiple tax rates, apply taxes based on location, and ensure compliance with tax regulations.

## Prerequisites

- WHMCS with tax configuration access
- Understanding of applicable tax regulations
- Tax rates documented for all jurisdictions
- SSL configured for secure transactions

## Workflow Steps

### Step 1: Configure Tax Settings

Set up base tax configuration:

```php
// WHMCS Admin > Configuration > System Settings > Taxes
// Enable tax functionality with desired configuration

// Database configuration for advanced tax settings
INSERT INTO tblconfiguration (setting, value) VALUES 
('TaxEnabled', 'on'),
('TaxType', 'Inclusive'), // or 'Exclusive'
('TaxLevel', '2'), // 1 = Federal only, 2 = Federal + State/Province
('CompoundTax', 'on'),
('TaxExemptNumberEnabled', 'on');

// Create tax zones table
Capsule::schema()->create('mod_tax_zones', function($t) {
    $t->increments('id');
    $t->string('name');
    $t->string('country_code', 2);
    $t->string('state_code', 10)->nullable();
    $t->string('tax_rate_name');
    $t->decimal('tax_rate', 5, 2);
    $t->boolean('is_compound')->default(false);
    $t->boolean('is_active')->default(true);
    $t->integer('priority')->default(0);
});

Capsule::schema()->create('mod_tax_exemptions', function($t) {
    $t->increments('id');
    $t->integer('client_id');
    $t->string('tax_number');
    $t->string('exemption_certificate');
    $t->date('valid_from');
    $t->date('valid_until')->nullable();
    $t->boolean('is_active')->default(true);
    $t->timestamp('created_at')->default(Capsule::raw('CURRENT_TIMESTAMP'));
});
```

### Step 2: Create Tax Calculation Service

Build tax calculation functionality:

```php
// File: /includes/classes/TaxCalculator.php

namespace WHMCS\Billing;

class TaxCalculator {
    
    private $taxRules = [];
    private $clientTaxExempt = false;
    
    public function calculateTax($amount, $clientId, $country = null, $state = null) {
        // Check for tax exemption
        if ($this->isClientTaxExempt($clientId)) {
            return [
                'subtotal' => $amount,
                'tax' => 0,
                'total' => $amount,
                'tax_breakdown' => []
            ];
        }
        
        // Get applicable tax rules
        $taxRules = $this->getTaxRules($country, $state);
        
        $taxTotal = 0;
        $taxBreakdown = [];
        
        foreach ($taxRules as $index => $rule) {
            $taxableAmount = $amount;
            
            // Handle compound tax (tax on tax)
            if ($rule->is_compound && $index > 0) {
                $taxableAmount = $amount + $taxTotal;
            }
            
            $taxAmount = $taxableAmount * ($rule->tax_rate / 100);
            $taxTotal += $taxAmount;
            
            $taxBreakdown[] = [
                'name' => $rule->tax_rate_name,
                'rate' => $rule->tax_rate,
                'amount' => round($taxAmount, 2),
                'taxable_amount' => round($taxableAmount, 2)
            ];
        }
        
        return [
            'subtotal' => $amount,
            'tax' => round($taxTotal, 2),
            'total' => round($amount + $taxTotal, 2),
            'tax_breakdown' => $taxBreakdown
        ];
    }
    
    private function getTaxRules($country, $state) {
        $query = Capsule::table('mod_tax_zones')
            ->where('is_active', true)
            ->orderBy('priority', 'asc');
        
        if ($country) {
            $query->where(function($q) use ($country, $state) {
                $q->where('country_code', $country);
                
                if ($state) {
                    $q->where(function($q2) use ($state) {
                        $q2->where('state_code', $state)
                            ->orWhereNull('state_code');
                    });
                } else {
                    $q->whereNull('state_code');
                }
            });
        }
        
        return $query->get();
    }
    
    private function isClientTaxExempt($clientId) {
        $exemption = Capsule::table('mod_tax_exemptions')
            ->where('client_id', $clientId)
            ->where('is_active', true)
            ->where(function($q) {
                $q->whereNull('valid_until')
                    ->orWhere('valid_until', '>=', date('Y-m-d'));
            })
            ->first();
        
        return $exemption !== null;
    }
    
    public function applyTaxExemption($clientId, $taxNumber, $certificateUrl = null, $validUntil = null) {
        Capsule::table('mod_tax_exemptions')->insert([
            'client_id' => $clientId,
            'tax_number' => $taxNumber,
            'exemption_certificate' => $certificateUrl,
            'valid_from' => date('Y-m-d'),
            'valid_until' => $validUntil,
            'is_active' => true
        ]);
        
        // Update client record
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update(['taxexempt' => 1]);
        
        logActivity("Tax exemption applied for client #{$clientId}: {$taxNumber}");
    }
    
    public function removeTaxExemption($clientId) {
        Capsule::table('mod_tax_exemptions')
            ->where('client_id', $clientId)
            ->update(['is_active' => false]);
        
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update(['taxexempt' => 0]);
        
        logActivity("Tax exemption removed for client #{$clientId}");
    }
}
```

### Step 3: Create Tax Zone Management

Handle tax zones and rates:

```php
// File: /includes/hooks/tax_zone_management.php

add_hook('AdminAreaPageStart', 1, function($vars) {
    // Add tax settings to admin configuration
    if (isset($_GET['action']) && $_GET['action'] === 'tax_settings') {
        return getTaxSettingsConfig();
    }
});

function getTaxSettingsConfig() {
    return [
        'tax_zones' => Capsule::table('mod_tax_zones')
            ->where('is_active', true)
            ->orderBy('priority')
            ->get(),
        'tax_exemptions' => Capsule::table('mod_tax_exemptions')
            ->where('is_active', true)
            ->where('valid_until', '>=', date('Y-m-d'))
            ->get(),
        'tax_reports' => [
            'total_collected' => getTotalTaxCollected(),
            'exempt_customers' => Capsule::table('mod_tax_exemptions')
                ->where('is_active', true)
                ->count()
        ]
    ];
}

function addTaxZone($data) {
    $priority = Capsule::table('mod_tax_zones')
        ->where('country_code', $data['country_code'])
        ->max('priority') + 1;
    
    Capsule::table('mod_tax_zones')->insert([
        'name' => $data['name'],
        'country_code' => strtoupper($data['country_code']),
        'state_code' => $data['state_code'] ?? null,
        'tax_rate_name' => $data['tax_rate_name'],
        'tax_rate' => $data['tax_rate'],
        'is_compound' => $data['is_compound'] ?? false,
        'is_active' => true,
        'priority' => $priority
    ]);
    
    logActivity("Tax zone added: {$data['name']} at {$data['tax_rate']}%");
}

function updateTaxRate($zoneId, $newRate) {
    $zone = Capsule::table('mod_tax_zones')
        ->where('id', $zoneId)
        ->first();
    
    Capsule::table('mod_tax_zones')
        ->where('id', $zoneId)
        ->update(['tax_rate' => $newRate]);
    
    logActivity("Tax rate updated for {$zone->name}: {$zone->tax_rate}% -> {$newRate}%");
}
```

### Step 4: Create Tax Calculation Hooks

Integrate tax calculation into invoices:

```php
// File: /includes/hooks/tax_calculation.php

add_hook('InvoiceCreationPreTax', 1, function($vars) {
    $clientId = $vars['userid'];
    $client = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();
    
    // Skip if client is tax exempt
    if ($client->taxexempt) {
        return ['skip_tax' => true];
    }
    
    // Get location for tax determination
    $country = $client->country;
    $state = $client->state;
    
    $calculator = new \WHMCS\Billing\TaxCalculator();
    
    // This would modify the invoice items with calculated tax
    return [
        'country' => $country,
        'state' => $state,
        'apply_location_tax' => true
    ];
});

add_hook('InvoicePaid', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];
    
    // Record tax collected for reporting
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->first();
    
    $taxAmount = $invoice->tax + $invoice->tax2;
    
    if ($taxAmount > 0) {
        Capsule::table('mod_tax_collection')->insert([
            'invoice_id' => $invoiceId,
            'client_id' => $invoice->userid,
            'tax_amount' => $taxAmount,
            'country' => $invoice->country,
            'state' => $invoice->state,
            'collected_at' => $invoice->datepaid
        ]);
    }
});
```

### Step 5: Create Tax Reporting

Generate tax reports:

```php
// File: /includes/hooks/tax_reporting.php

add_hook('DailyCronJob', 1, function($vars) {
    // Generate tax collection report
    $month = date('Y-m');
    $startDate = date('Y-m-01');
    $endDate = date('Y-m-t');
    
    $report = [
        'period' => $month,
        'total_collected' => 0,
        'by_state' => [],
        'exempt_sales' => 0
    ];
    
    // Get tax collected this month
    $taxCollections = Capsule::table('mod_tax_collection')
        ->whereBetween('collected_at', [$startDate, $endDate . ' 23:59:59'])
        ->get();
    
    foreach ($taxCollections as $collection) {
        $report['total_collected'] += $collection->tax_amount;
        
        $location = $collection->state ?: $collection->country;
        if (!isset($report['by_state'][$location])) {
            $report['by_state'][$location] = ['amount' => 0, 'count' => 0];
        }
        $report['by_state'][$location]['amount'] += $collection->tax_amount;
        $report['by_state'][$location]['count']++;
    }
    
    // Get exempt sales
    $exemptInvoices = Capsule::table('tblinvoices')
        ->join('tblclients', 'tblinvoices.userid', '=', 'tblclients.id')
        ->where('tblclients.taxexempt', 1)
        ->whereBetween('tblinvoices.datepaid', [$startDate, $endDate . ' 23:59:59'])
        ->selectRaw('SUM(tblinvoices.subtotal) as total')
        ->first();
    
    $report['exempt_sales'] = $exemptInvoices->total ?? 0;
    
    // Save report
    file_put_contents(
        __DIR__ . "/../storage/logs/tax_reports/{$month}.json",
        json_encode($report, JSON_PRETTY_PRINT)
    );
    
    return $report;
});

function generateTaxReport($startDate, $endDate, $jurisdiction = null) {
    $query = Capsule::table('tblinvoices')
        ->where('status', 'Paid')
        ->whereBetween('datepaid', [$startDate, $endDate]);
    
    if ($jurisdiction) {
        $query->where(function($q) use ($jurisdiction) {
            $q->where('state', $jurisdiction)
                ->orWhere('country', $jurisdiction);
        });
    }
    
    $invoices = $query->get();
    
    $report = [
        'period' => $startDate . ' to ' . $endDate,
        'jurisdiction' => $jurisdiction,
        'invoice_count' => count($invoices),
        'total_sales' => 0,
        'total_tax' => 0,
        'tax_rate_breakdown' => []
    ];
    
    foreach ($invoices as $invoice) {
        $report['total_sales'] += $invoice->subtotal;
        $report['total_tax'] += $invoice->tax + $invoice->tax2;
    }
    
    return $report;
}
```

## Verification Checklist

- [ ] Tax configuration saved correctly
- [ ] Tax rates applied correctly per zone
- [ ] Compound tax calculated accurately
- [ ] Tax exemption handling working
- [ ] Tax reports generating correctly
- [ ] Client location-based tax working
- [ ] Tax-exempt customers not charged
- [ ] Invoice totals correct with tax
- [ ] Tax collected matches reporting
- [ ] Multiple tax tiers working

## Related Skills and Documentation

- [WHMCS Invoice Automation](whmcs-invoice-automation-workflow.md)
- [WHMCS Billing Audit](whmcs-billing-audit-workflow.md)
- [WHMCS Billing Reporting](whmcs-billing-reporting-workflow.md)
- WHMCS Documentation: Tax Configuration
- WHMCS Documentation: VAT/Sales Tax

## Notes

- Regularly update tax rates as regulations change
- Maintain accurate tax zone configurations
- Document tax exemption procedures
- Keep tax reports for compliance periods
- Consider automation for rate updates
- Review tax calculations with tax professional
- Ensure proper tax nexus considerations
- Handle digital goods vs physical goods tax differences