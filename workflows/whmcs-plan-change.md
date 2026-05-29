# WHMCS Plan Change Workflow

## Overview
This workflow automates service plan upgrades and downgrades.

## Prerequisites
- WHMCS installation
- Product pricing configured

## Step-by-Step Guide

### Step 1: Create Upgrade Hook
```php
add_hook('ServiceUpgradeComplete', 1, function($vars) {
    $serviceId = $vars['serviceId'];
    $oldPlanId = $vars['oldPlanId'];
    $newPlanId = $vars['newPlanId'];
    
    // Get plan details
    $oldPlan = get_product($oldPlanId);
    $newPlan = get_product($newPlanId);
    
    // Update external service
    $api = new YourModuleAPI();
    $api->change_plan($serviceId, $newPlan->name);
    
    // Calculate prorated amount
    $proration = calculate_proration($serviceId, $oldPlan, $newPlan);
    
    // Create invoice for difference
    if ($proration != 0) {
        create_invoice_item($serviceId, $proration, 'Plan upgrade proration');
    }
    
    // Log the change
    log_plan_change($serviceId, $oldPlanId, $newPlanId);
});

function calculate_proration(int $serviceId, $oldPlan, $newPlan): float
{
    $service = \WHMCS\Database\Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();
    
    $billingCycle = $service->billingcycle;
    $daysRemaining = get_days_remaining($service->nextduedate);
    $daysInCycle = get_days_in_cycle($service->nextduedate);
    
    $oldPrice = get_price($oldPlan, $billingCycle);
    $newPrice = get_price($newPlan, $billingCycle);
    
    $dailyOldRate = $oldPrice / $daysInCycle;
    $dailyNewRate = $newPrice / $daysInCycle;
    
    $credit = $dailyOldRate * $daysRemaining;
    $charge = $dailyNewRate * $daysRemaining;
    
    return round($charge - $credit, 2);
}
```

## Plan Change Checklist

### Processing
- [ ] New plan applied
- [ ] External system updated
- [ ] Proration calculated

### Billing
- [ ] Invoice created
- [ ] Credits applied
- [ ] Payment processed
