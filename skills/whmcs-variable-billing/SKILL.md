# WHMCS Variable Billing

## Concept Explanation

Variable billing (usage-based billing) calculates charges based on actual resource consumption rather than fixed amounts. This model is common for cloud services, bandwidth overages, API usage, storage consumption, and metered billing. WHMCS can track usage metrics and generate dynamic invoices based on consumption data.

### Billing Models

- **Per-Unit Pricing**: Fixed rate per unit consumed (e.g., $0.05 per GB)
- **Tiered Pricing**: Volume discounts based on usage levels
- **Hybrid**: Base fee + variable usage charges
- **Overage Billing**: Base allocation with per-unit overage charges

## Code Patterns & Templates

### Usage Tracking Module

```php
<?php
// modules/usage_tracker/usage_tracker.php

namespace WHMCS\Module\UsageTracker;

class UsageTracker {
    
    protected $serviceId;
    protected $metrics = [];
    
    /**
     * Track a usage metric
     */
    public function track($serviceId, $metricType, $value, $timestamp = null) {
        $timestamp = $timestamp ?? date('Y-m-d H:i:s');
        
        insert_query('mod_usage_metrics', [
            'service_id' => $serviceId,
            'metric_type' => $metricType,
            'value' => $value,
            'recorded_at' => $timestamp,
            'created_at' => date('Y-m-d H:i:s')
        ]);
        
        return true;
    }
    
    /**
     * Get aggregated usage for billing period
     */
    public function getUsage($serviceId, $metricType, $startDate, $endDate) {
        $query = "SELECT 
            SUM(value) as total_usage,
            MIN(value) as min_usage,
            MAX(value) as max_usage,
            AVG(value) as avg_usage
        FROM mod_usage_metrics
        WHERE service_id = ?
        AND metric_type = ?
        AND recorded_at BETWEEN ? AND ?";
        
        $result = full_query($query, [$serviceId, $metricType, $startDate, $endDate]);
        return mysql_fetch_array($result);
    }
    
    /**
     * Calculate billable amount based on pricing tiers
     */
    public function calculateTieredCost($totalUsage, $pricingTiers) {
        $cost = 0;
        $remainingUsage = $totalUsage;
        
        usort($pricingTiers, function($a, $b) {
            return $a['min_units'] - $b['min_units'];
        });
        
        foreach ($pricingTiers as $tier) {
            if ($remainingUsage <= 0) break;
            
            $tierUnits = $tier['max_units'] - $tier['min_units'];
            $unitsInTier = min($remainingUsage, $tierUnits);
            
            $cost += $unitsInTier * $tier['price_per_unit'];
            $remainingUsage -= $unitsInTier;
        }
        
        return $cost;
    }
}
```

### Overage Billing Handler

```php
<?php
// hooks/overage_billing.php

use WHMCS\Service\Service;

/**
 * Process overage charges for a billing period
 */
function process_overage_billing($serviceId, $billingPeriod) {
    $service = Service::find($serviceId);
    $product = getProduct($service->packageid);
    
    $startDate = get_billing_period_start($service->nextduedate, $billingPeriod);
    $endDate = $service->nextduedate;
    
    $usageData = [];
    
    // Track bandwidth usage
    if ($product['configoption1'] === 'bandwidth_tracked') {
        $bandwidthUsage = get_bandwidth_usage($serviceId, $startDate, $endDate);
        $includedBandwidth = $product['configoption2']; // GB included
        
        if ($bandwidthUsage > $includedBandwidth) {
            $overageBandwidth = $bandwidthUsage - $includedBandwidth;
            $overageRate = $product['configoption3']; // Rate per GB
            
            $usageData['bandwidth_overage'] = [
                'used' => $bandwidthUsage,
                'included' => $includedBandwidth,
                'overage' => $overageBandwidth,
                'rate' => $overageRate,
                'charge' => $overageBandwidth * $overageRate
            ];
        }
    }
    
    // Track storage usage
    if ($product['configoption4'] === 'storage_tracked') {
        $storageUsage = get_storage_usage($serviceId, $endDate);
        $includedStorage = $product['configoption5'];
        
        if ($storageUsage > $includedStorage) {
            $overageStorage = $storageUsage - $includedStorage;
            $overageRate = $product['configoption6'];
            
            $usageData['storage_overage'] = [
                'used' => $storageUsage,
                'included' => $includedStorage,
                'overage' => $overageStorage,
                'rate' => $overageRate,
                'charge' => $overageStorage * $overageRate
            ];
        }
    }
    
    // Generate overage invoice if charges exist
    if (!empty($usageData)) {
        createOverageInvoice($service->userid, $serviceId, $usageData);
    }
    
    return $usageData;
}

/**
 * Cron job for processing overage billing daily
 */
add_hook('DailyCronJob', 1, function() {
    $services = get_all_active_services_with_overage();
    
    foreach ($services as $service) {
        $daysUntilDue = days_until($service['nextduedate']);
        
        // Process billing 7 days before due date
        if ($daysUntilDue <= 7 && $daysUntilDue > 0) {
            process_overage_billing($service['id'], $service['billingcycle']);
        }
    }
});
```

### Real-Time Usage API

```php
<?php
// api/usage_api.php

/**
 * API endpoint for external systems to report usage
 */
add_hook('ApiController', 1, function($vars) {
    if ($vars['action'] === 'ReportUsage') {
        $serviceId = (int)$vars['service_id'];
        $metricType = $vars['metric_type'];
        $value = (float)$vars['value'];
        
        // Validate service ownership
        $service = Service::find($serviceId);
        if (!$service || $service->userid !== (int)$vars['client_id']) {
            return ['success' => false, 'error' => 'Invalid service'];
        }
        
        // Record usage
        $tracker = new \WHMCS\Module\UsageTracker\UsageTracker();
        $tracker->track($serviceId, $metricType, $value);
        
        // Check for threshold alerts
        check_usage_thresholds($serviceId, $metricType, $value);
        
        return ['success' => true, 'recorded' => true];
    }
    
    if ($vars['action'] === 'GetUsageSummary') {
        $serviceId = (int)$vars['service_id'];
        $period = $vars['period'] ?? 'current';
        
        $startDate = get_period_start_date($period);
        $endDate = date('Y-m-d');
        
        $tracker = new \WHMCS\Module\UsageTracker\UsageTracker();
        $metrics = get_all_metrics_for_service($serviceId);
        
        $summary = [];
        foreach ($metrics as $metric) {
            $summary[$metric] = $tracker->getUsage($serviceId, $metric, $startDate, $endDate);
        }
        
        return ['success' => true, 'period' => $period, 'metrics' => $summary];
    }
});
```

## Step-by-Step Implementation

### 1. Design Usage Tracking Schema

```sql
CREATE TABLE mod_usage_metrics (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    metric_type VARCHAR(50) NOT NULL,
    value DECIMAL(15,4) NOT NULL,
    recorded_at DATETIME NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_service_metric (service_id, metric_type),
    INDEX idx_recorded_at (recorded_at)
);

CREATE TABLE mod_usage_pricing_tiers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL,
    metric_type VARCHAR(50) NOT NULL,
    min_units DECIMAL(15,4) DEFAULT 0,
    max_units DECIMAL(15,4) DEFAULT 0,
    price_per_unit DECIMAL(10,4) NOT NULL,
    FOREIGN KEY (product_id) REFERENCES tblproducts(id)
);
```

### 2. Configure Product for Usage Tracking

1. Create custom configuration options:
   - Enable usage tracking
   - Define included units
   - Set overage rates
   - Configure billing thresholds

### 3. Implement Usage Collection

Create API endpoints or agent scripts to collect usage data:
- API integration for real-time reporting
- Cron-based collection from servers
- Manual entry option for non-automated tracking

### 4. Set Up Billing Calculation

Implement the billing calculation logic:
- Aggregate usage for billing period
- Apply pricing tiers or flat rates
- Generate overage invoices

### 5. Configure Notifications

Set up alerts for usage thresholds:
- Warning at 75% of included usage
- Alert at 90%
- Critical at 100%

## Examples

### Example: API Call Billing

```php
<?php
// Billing based on API calls per month
add_hook('ApiLogAction', 1, function($vars) {
    $apiLog = $vars['log'];
    $clientId = $apiLog['userid'];
    $serviceId = get_service_for_api_key($apiLog['api_key']);
    
    if ($serviceId) {
        $tracker = new \WHMCS\Module\UsageTracker\UsageTracker();
        $tracker->track($serviceId, 'api_calls', 1);
        
        // Check threshold
        $monthStart = date('Y-m-01');
        $monthEnd = date('Y-m-t');
        $usage = $tracker->getUsage($serviceId, 'api_calls', $monthStart, $monthEnd);
        
        $includedCalls = get_product_included_units($serviceId, 'api_calls');
        $utilization = $usage['total_usage'] / $includedCalls;
        
        if ($utilization >= 0.9) {
            send_usage_warning_email($clientId, 'api_calls', $utilization);
        }
    }
});
```

### Example: Bandwidth Overage Billing

```php
<?php
// Process bandwidth overages on billing cycle
add_hook('InvoiceCreationPreEmail', 1, function($vars) {
    $invoice = getInvoice($vars['invoiceid']);
    
    if ($invoice['status'] !== 'Unpaid') return $vars;
    
    $items = $invoice['lineitems'];
    
    foreach ($items as $item) {
        if ($item['type'] === 'Hosting') {
            $overageCharges = get_bandwidth_overage($item['relid']);
            
            if ($overageCharges > 0) {
                addInvoiceLineItem($invoice['id'], 'Bandwidth Overage', 
                    'overage_bandwidth', $overageCharges);
            }
        }
    }
    
    return $vars;
});
```

## Implementation Checklist

- [ ] Create usage tracking database schema
- [ ] Design pricing tier configuration
- [ ] Implement usage collection mechanism
- [ ] Create billing calculation module
- [ ] Set up overage invoice generation
- [ ] Configure usage threshold alerts
- [ ] Implement customer usage dashboard
- [ ] Add API endpoints for external tracking
- [ ] Test tiered pricing calculations
- [ ] Verify overage billing accuracy
- [ ] Document usage limits for customers
