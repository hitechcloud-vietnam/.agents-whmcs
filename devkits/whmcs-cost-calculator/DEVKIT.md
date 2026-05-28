# WHMCS Cost Calculator Module

Cloud cost calculator with pricing models, projections, and cost analytics.

## Features

- Multiple pricing models (per hour, per GB, per API call)
- Resource cost calculation
- Cost projections (daily, monthly, yearly)
- Rate card management
- Tiered pricing support
- Discount rules
- Cost breakdown by service/resource
- Budget tracking and alerts
- Cost comparison tools
- Invoice cost estimation
- Custom pricing formulas
- Cost reporting and analytics

## Installation

1. Copy module to `/path/to/whmcs/modules/addons/costcalculator/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure pricing rates

## Usage

```php
// Calculate resource cost
$result = costcalculator_CalculateCost(array(
    'resource_type' => 'server',
    'resources' => array(
        'cpu_cores' => 4,
        'ram_gb' => 16,
        'disk_gb' => 500,
        'bandwidth_gb' => 1000
    ),
    'period' => 'monthly'
));

// Add pricing rate
costcalculator_AddRate(array(
    'resource_name' => 'cpu_cores',
    'unit' => 'core',
    'cost_per_unit' => 0.05,
    'model' => 'per_hour'
));

// Create pricing tier
costcalculator_CreateTier(array(
    'resource_name' => 'disk_gb',
    'tier_name' => 'SSD Storage',
    'min_value' => 0,
    'max_value' => 100,
    'cost_per_unit' => 0.10
));

// Calculate service cost
$serviceCost = costcalculator_CalculateServiceCost($serviceId);

// Get cost projection
$projection = costcalculator_GetProjection($serviceId, 365);
// Returns: daily, weekly, monthly projections

// Create discount rule
costcalculator_CreateDiscount(array(
    'rule_name' => 'Volume Discount',
    'resource_name' => 'bandwidth_gb',
    'min_quantity' => 1000,
    'discount_percent' => 15
));

// Apply discount
$result = costcalculator_ApplyDiscount($userId, 'bandwidth_gb', 5000);

// Get cost breakdown
$breakdown = costcalculator_GetCostBreakdown($userId, '2026-05');

// Set budget
costcalculator_SetBudget($userId, 500.00, 'monthly');

// Check budget
$budgetStatus = costcalculator_CheckBudget($userId);
// Returns: budget_amount, spent, remaining, percent_used

// Get cost analytics
$analytics = costcalculator_GetAnalytics($userId, 90);
// Returns: total_spent, by_resource, trends, forecasts

// Get rate card
$rates = costcalculator_GetRateCard();

// Update rate
costcalculator_UpdateRate($rateId, array(
    'cost_per_unit' => 0.055
));

// Calculate custom formula
$result = costcalculator_CalculateFormula('cpu_cores * 0.05 + ram_gb * 0.02 + disk_gb * 0.01', array(
    'cpu_cores' => 4,
    'ram_gb' => 16,
    'disk_gb' => 500
));
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| DefaultCurrency | text | USD | Default currency |
| TaxRate | text | 0 | Tax rate (%) |
| EnableTieredPricing | yesno | yes | Enable tiered pricing |
| EnableBudgetTracking | yesno | yes | Enable budget tracking |
| BudgetAlertThreshold | text | 80 | Alert threshold (%) |
| MinimumCharge | text | 0.00 | Minimum invoice amount |
| ShowHiddenResources | yesno | no | Show internal resources |

## Pricing Models

| Model | Description |
|-------|-------------|
| per_hour | Cost per hour of usage |
| per_gb | Cost per GB stored/transfer |
| per_api_call | Cost per API call |
| per_unit | Cost per unit consumed |
| flat_rate | Fixed flat rate |
| tiered | Tiered pricing based on quantity |
| usage_based | Actual consumption billing |

## Resource Types

| Category | Resources |
|----------|-----------|
| compute | cpu_cores, ram_gb, gpu_units |
| storage | disk_gb, backup_gb, snapshot_gb |
| network | bandwidth_gb, ip_addresses, ssl_certs |
| api | api_calls, function_invocations |
| support | premium_support, dedicated_support |

## Database Tables

- `mod_costcalculator_rates` - Pricing rates
- `mod_costcalculator_tiers` - Tiered pricing
- `mod_costcalculator_discounts` - Discount rules
- `mod_costcalculator_budgets` - Budget limits
- `mod_costcalculator_history` - Cost history
- `mod_costcalculator_formulas` - Custom formulas

## API Functions

| Function | Description |
|----------|-------------|
| `costcalculator_CalculateCost()` | Calculate resource cost |
| `costcalculator_CalculateServiceCost()` | Calculate service cost |
| `costcalculator_CalculateFormula()` | Calculate custom formula |
| `costcalculator_AddRate()` | Add pricing rate |
| `costcalculator_GetRateCard()` | Get all rates |
| `costcalculator_UpdateRate()` | Update rate |
| `costcalculator_CreateTier()` | Create tiered pricing |
| `costcalculator_CreateDiscount()` | Create discount rule |
| `costcalculator_ApplyDiscount()` | Apply discount |
| `costcalculator_GetProjection()` | Get cost projection |
| `costcalculator_GetCostBreakdown()` | Get cost breakdown |
| `costcalculator_SetBudget()` | Set budget limit |
| `costcalculator_CheckBudget()` | Check budget status |
| `costcalculator_GetAnalytics()` | Get cost analytics |
