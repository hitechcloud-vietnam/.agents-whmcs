# WHMCS Tax Rules Configuration

## Overview

WHMCS provides comprehensive tax rule configuration supporting multiple tax levels, compound tax calculation, VAT handling, and location-based tax rules.

## Accessing Tax Configuration

**Admin Area > Configuration > System Settings > Tax**

## Tax Setup Levels

### Single Tax Level
- Simple percentage-based tax
- Applied to all products/services
- Configuration: `Tax Enabled: Yes`, `Tax Level: Single`

### Two Tax Levels (US Style)
- Separate state/county and local taxes
- Tax levels stack (compound or inclusive)
- Configuration: `Tax Level: Two`

## Tax Rule Configuration

### Creating Tax Rules

**Path**: Configuration > Tax > Add New Tax Rule

| Field | Description |
|-------|-------------|
| Name | Tax rule name (e.g., "State Tax", "VAT") |
| Rate | Tax percentage (e.g., 20 for 20%) |
| Country | Apply to specific country (or "All Countries") |
| State/Region | Apply to specific state/province |
| City | Apply to specific city |
| Applied To | Products that tax applies to |
| Priority | Lower number = applied first |

### Tax Rules Table

```php
// Example tax rule structure
[
    'name' => 'VAT',
    'rate' => 20.00,
    'country' => 'GB',
    'state' => '',
    'city' => '',
    'level' => 1,
    'priority' => 1,
    'inclusive' => false
]
```

## Tax Configuration Settings

**Configuration > System > Tax**

| Setting | Description | Default |
|---------|-------------|------------------|
| Tax Enabled | Enable/disable tax | Disabled |
| Tax Level | Single/Two tax levels | Single |
| Compound Tax | Stack taxes compoundly | No |
| Tax Matching | Country/State/City matching | Country & State |
| Default Tax Code | Default tax classification | None |
| EU VAT Checking | Enable EU VAT number validation | No |
| Auto-detect Client Tax | Match tax rules on signup | Yes |

## Location-Based Tax Rules

### Country-Level Rules
```
Country: United States
Rate: 0%
```

### State-Level Rules
```
Country: United States
State: California
Rate: 7.5%
```

### City-Level Rules
```
Country: United States
State: California
City: San Francisco
Rate: 8.625%
```

### Priority System
When multiple rules match, they are applied by priority (lower number first).

## Tax Inclusive Pricing

Configure whether product prices include tax:

```php
// Product Configuration
Prices Include Tax: Yes/No
```

### Inclusive Tax Calculation
```php
// For 20% inclusive VAT
// Original price from product: $100
// Actual net amount: $83.33
// VAT amount: $16.67
// Gross total: $100.00
```

## Compound Tax

When enabled, tax levels are calculated on top of each other:

```php
// Example: Compound Tax Enabled
// Subtotal: $100
// Level 1 Tax (10%): $10
// Level 2 Tax (5%): $5.50  (calculated on $110)
// Total: $115.50
```

### Non-Compound (Cumulative)
```php
// Example: Compound Tax Disabled
// Subtotal: $100
// Level 1 Tax (10%): $10
// Level 2 Tax (5%): $5     (calculated on $100)
// Total: $115.00
```

## EU VAT Handling

### VAT Number Validation

```php
// Configuration > Tax > EU VAT Settings
Enable VIES VAT Validation: Yes
Validate VAT Number on Signup: Yes
Apply Reverse Charge: Yes
```

### Reverse Charge for B2B
When valid VAT number provided:
- Customer bears responsibility for VAT
- Reverse charge applied
- No VAT collected on invoice

### VIES API Integration
```php
// VAT number format validation
GB 123 456 789
DE 123 456 789
FR 12 345 678 901
```

## Product Tax Settings

### Global Tax Setting
Applied to all products unless overridden.

### Product-Specific Tax
```php
// In Product Configuration > Pricing Tab
Tax: Follow Global Setting
     Always Apply Tax
     Never Apply Tax
```

### Tax Exemption
```php
// Configuration > Tax > Exemption Settings
Allow Tax Exempt: Yes
Require Validation: Yes
Default Exemption: No
```

## Tax API Functions

```php
// Calculate tax for client
$taxCalc = Module::buildData('TaxCalculator', [
    'clientState' => $clientState,
    'clientCountry' => $clientCountry,
    'taxRates' => $taxRates
]);

// Get applicable tax rules
$rules = WHMCS\\Tax\\TaxRule::getRulesForLocation($country, $state);
```

## Tax Reports

**Reports > Billing > Tax Summary**

- Tax collected by period
- Tax liability by jurisdiction
- Tax breakdown per invoice

### Export Options
- CSV export
- PDF summary
- Date range filtering

## Hook Integration

```php
// Hook: TaxCalculation
add_hook('TaxCalculation', 1, function($vars) {
    // $vars['taxRules']
    // $vars['subTotal']
    // $vars['clientCountry']
    // $vars['clientState']
    
    // Modify tax calculation
    return [
        'override' => false,
        'taxRules' => $vars['taxRules']
    ];
});
```

## Best Practices

1. **Clear naming**: Use descriptive names for tax rules
2. **Priority planning**: Set priorities logically
3. **Regular review**: Update tax rates as legislation changes
4. **Documentation**: Track tax rule changes
5. **Test thoroughly**: Test various scenarios before live

## Common Tax Scenarios

### US Sales Tax
```php
Rule 1: Country=US, Rate=0%
Rule 2: Country=US, State=CA, Rate=7.5%
Rule 3: Country=US, State=TX, Rate=6.25%
```

### UK VAT
```php
Rule: Country=GB, Rate=20%
```

### Australia GST
```php
Rule: Country=AU, Rate=10%
```

## Related Documentation

- [Invoice Generation](./whmcs-invoice-generation.md)
- [Pricing Tiers](./whmcs-pricing-tiers.md)
- [Multi-Currency](./whmcs-multi-currency.md)
- [Client Custom Fields](./whmcs-client-custom-fields.md)