# WHMCS Custom Tax Rules Documentation

## Overview

Custom tax rules in WHMCS allow you to define complex tax configurations for various jurisdictions, including compound taxes, exempt products, and location-based rates.

## Configuration

### Enable Custom Taxes

Navigate to: **Configuration > General Settings > Tax Settings**

```php
// Custom Tax Configuration
$customTaxConfig = [
    'enabled' => true,
    'tax_count' => 2,                     // Support up to 2 taxes

    // Basic Settings
    'compound_tax' => true,               // Tax on tax
    'inclusive_tax' => true,              // Tax-inclusive pricing
    'display_tax' => true,
    'show_tax_id' => true,

    // Calculation
    'calculation_order' => 'tax1_then_tax2',
    'round_tax_amounts' => true,
    'round_precision' => 2,

    // Products
    'tax_exempt_available' => true,
    'tax_exempt_by_product' => true,
    'tax_exempt_by_category' => true,

    // Customers
    'tax_exempt_by_customer' => true,
    'tax_exempt_requires_certificate' => true
];
```

## Tax Rule Types

### Standard Tax

```php
// Standard tax rule
$standardTax = [
    'name' => 'Sales Tax',
    'rate' => 8.25,
    'type' => 'percentage',
    'applicable_to' => 'all',
    'countries' => ['US'],
    'states' => ['TX'],                   // Optional state filter
    'products' => 'all',                  // or ['category_ids'] or ['product_ids']
    'customers' => 'all',
    'compound' => false
];
```

### Compound Tax

```php
// Compound tax (tax on tax)
$compoundTax = [
    'name' => 'City Tax',
    'rate' => 1.00,
    'type' => 'percentage',
    'applicable_to' => 'all',
    'countries' => ['US'],
    'states' => ['NY'],
    'compound_with' => 'sales_tax',       // Apply on top of sales tax
    'minimum_amount' => 0
];
```

### Tiered Tax

```php
// Tiered tax based on amount
$tieredTax = [
    'name' => 'Luxury Tax',
    'type' => 'tiered',
    'tiers' => [
        ['min' => 0, 'max' => 100, 'rate' => 0],
        ['min' => 100, 'max' => 500, 'rate' => 5],
        ['min' => 500, 'max' => 1000, 'rate' => 10],
        ['min' => 1000, 'max' => null, 'rate' => 15]
    ],
    'countries' => ['US']
];
```

### Fixed Tax

```php
// Fixed amount tax
$fixedTax = [
    'name' => 'Environmental Fee',
    'rate' => 5.00,
    'type' => 'fixed',
    'applicable_to' => 'specific_products',
    'product_ids' => [100, 101, 102],
    'countries' => ['US']
];
```

## Geographic Tax Rules

### Country-Specific Taxes

```php
// Tax rules by country
$countryTaxes = [
    'US' => [
        [
            'name' => 'Federal Tax',
            'rate' => 0,
            'type' => 'percentage'
        ]
    ],
    'US-CA' => [
        [
            'name' => 'California State Tax',
            'rate' => 7.25,
            'type' => 'percentage',
            'states' => ['CA']
        ]
    ],
    'US-NY' => [
        [
            'name' => 'New York State Tax',
            'rate' => 8.00,
            'type' => 'percentage',
            'states' => ['NY']
        ],
        [
            'name' => 'NYC Local Tax',
            'rate' => 4.875,
            'type' => 'percentage',
            'cities' => ['New York']
        ]
    ],
    'GB' => [
        [
            'name' => 'VAT',
            'rate' => 20,
            'type' => 'percentage'
        ]
    ]
];
```

### Multi-Level Tax

```php
// Multi-level geographic tax
$multiLevelTax = [
    'name' => 'Canadian GST/HST',
    'levels' => [
        'federal' => [
            'name' => 'GST',
            'rate' => 5,
            'type' => 'percentage',
            'countries' => ['CA']
        ],
        'provincial' => [
            'name' => 'PST',
            'rate' => 7,
            'type' => 'percentage',
            'countries' => ['CA'],
            'states' => ['BC']
        ]
    ]
];
```

## Product Tax Rules

### Tax by Product Category

```php
// Tax rules by product category
$categoryTaxRules = [
    'hosting' => [
        'taxable' => true,
        'tax_name' => 'Standard Rate',
        'tax_rate' => 20
    ],
    'software' => [
        'taxable' => true,
        'tax_name' => 'Software Tax',
        'tax_rate' => 25
    ],
    'digital_goods' => [
        'taxable' => true,
        'tax_name' => 'Digital Services Tax',
        'tax_rate' => 25
    ],
    'physical_goods' => [
        'taxable' => true,
        'tax_name' => 'Goods Tax',
        'tax_rate' => 20
    ]
];
```

### Tax-Exempt Products

```php
// Tax-exempt product categories
$taxExemptCategories = [
    'healthcare' => [
        'exempt_from' => ['all_taxes'],
        'requires_documentation' => true
    ],
    'education' => [
        'exempt_from' => ['vat', 'gst'],
        'requires_documentation' => false
    ],
    'exports' => [
        'exempt_from' => ['vat', 'gst', 'sales_tax'],
        'destination_required' => 'export'
    ],
    'food' => [
        'exempt_from' => ['vat'],
        'countries' => ['UK']
    ]
];
```

### Tax by Individual Product

```php
// Per-product tax override
$productTaxOverride = [
    'product_id' => 67890,
    'tax_override' => true,
    'taxable' => false,                   // or true
    'tax_rate' => 0,                       // or specific rate
    'tax_name' => 'Zero Rated',
    'reason' => 'Educational institution'
];
```

## Customer Tax Rules

### Customer Tax Profiles

```php
// Customer tax profiles
$customerTaxProfiles = [
    'default' => [
        'apply_taxes' => true,
        'tax_exempt' => false
    ],
    'business' => [
        'apply_taxes' => false,
        'tax_exempt' => true,
        'requires_vat_number' => true,
        'reverse_charge' => true
    ],
    'non_profit' => [
        'apply_taxes' => false,
        'tax_exempt' => true,
        'requires_certificate' => true
    ],
    'government' => [
        'apply_taxes' => false,
        'tax_exempt' => true,
        'requires_documentation' => true
    ]
];
```

### Tax by Customer Group

```php
// Customer group tax rules
$groupTaxRules = [
    'resellers' => [
        'default' => true,
        'taxable' => false,
        'requires_resale_certificate' => true
    ],
    'vip_customers' => [
        'default' => true,
        'tax_override' => 10,              // Reduced rate
        'discount_percentage' => 10
    ]
];
```

## API Reference

### Create Tax Rule

```http
POST /billing/tax/rules
```

**Request Body:**

```json
{
  "name": "State Sales Tax",
  "rate": 8.25,
  "type": "percentage",
  "countries": ["US"],
  "states": ["TX"],
  "priority": 1,
  "compound" => false,
  "applicable_to": {
    "type": "all"
  }
}
```

### Update Tax Rule

```http
PATCH /billing/tax/rules/{rule_id}
```

### Delete Tax Rule

```http
DELETE /billing/tax/rules/{rule_id}
```

### Get Tax Rules

```http
GET /billing/tax/rules
```

**Query Parameters:**
- `country`: Filter by country
- `state`: Filter by state
- `product_id`: Filter applicable products
- `enabled`: Filter enabled/disabled

**Response:**

```json
{
  "rules": [
    {
      "id": "RULE-001",
      "name": "State Sales Tax",
      "rate": 8.25,
      "type": "percentage",
      "countries": ["US"],
      "states": ["TX"],
      "priority": 1,
      "enabled": true
    },
    {
      "id": "RULE-002",
      "name": "VAT",
      "rate": 20,
      "type": "percentage",
      "countries": ["GB"],
      "priority": 1,
      "enabled": true
    }
  ]
}
```

### Calculate Tax

```http
POST /billing/tax/calculate
```

**Request Body:**

```json
{
  "amount": 100.00,
  "country": "US",
  "state": "TX",
  "customer_id": 12345,
  "product_ids": [100, 101]
}
```

**Response:**

```json
{
  "amount_excluding_tax": 100.00,
  "taxes": [
    {
      "rule_id": "RULE-001",
      "name": "State Sales Tax",
      "rate": 8.25,
      "amount": 8.25
    }
  ],
  "total_tax": 8.25,
  "amount_including_tax": 108.25
}
```

## Tax Zones

### Define Tax Zones

```php
// Tax zones for complex geographic rules
$taxZones = [
    'us_texas' => [
        'name' => 'Texas',
        'countries' => ['US'],
        'states' => ['TX'],
        'tax_rules' => ['RULE-001'],
        'priority' => 10
    ],
    'us_california' => [
        'name' => 'California',
        'countries' => ['US'],
        'states' => ['CA'],
        'tax_rules' => ['RULE-002'],
        'priority' => 10
    ],
    'uk' => [
        'name' => 'United Kingdom',
        'countries' => ['GB'],
        'tax_rules' => ['RULE-003'],
        'priority' => 10
    ]
];
```

### Tax Zone Assignment

```php
// Match customer to tax zone
function matchTaxZone($country, $state = null, $city = null) {
    $zones = getTaxZones();

    foreach ($zones as $zone) {
        if (in_array($country, $zone['countries'])) {
            if (isset($zone['states']) && !in_array($state, $zone['states'])) {
                continue;
            }
            if (isset($zone['cities']) && !in_array($city, $zone['cities'])) {
                continue;
            }
            return $zone;
        }
    }

    return null;  // No matching zone
}
```

## Compound Tax Calculation

### Calculation Example

```php
// Compound tax calculation
$compoundCalc = [
    'amount' => 100.00,

    // Tax 1 (applied to amount)
    'tax1' => [
        'name' => 'Sales Tax',
        'rate' => 10,
        'amount' => 10.00,
        'subtotal' => 110.00
    ],

    // Tax 2 (applied to amount + Tax 1)
    'tax2' => [
        'name' => 'Local Tax',
        'rate' => 5,
        'amount' => 5.50,                   // 10% of 110
        'subtotal' => 115.50
    ]
];

$total = 100.00 + 10.00 + 5.50;  // $115.50
```

## Tax Override

### Order-Level Override

```php
// Override taxes on specific order
$orderTaxOverride = [
    'order_id' => 12345,
    'override_taxes' => true,
    'tax_rules' => ['RULE-001'],          // Only apply this rule
    'reason' => 'Wholesale order - reduced tax',
    'approved_by' => 'admin@example.com'
];
```

### Invoice-Level Override

```php
// Override taxes on invoice
$invoiceTaxOverride = [
    'invoice_id' => 'INV-12345',
    'override_taxes' => true,
    'tax_amount' => 0,                    // Tax-free
    'reason' => 'Export outside jurisdiction',
    'documentation' => 'export_certificate.pdf'
];
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Wrong tax rate | Priority issue | Check rule priority |
| Tax not applied | Rule not matching | Verify country/state |
| Compound tax wrong | Order issue | Check calculation order |
| Tax override failed | Permission denied | Check admin permissions |

### Debug Commands

```bash
# List tax rules
whmcscli tax rules --country=US --state=TX

# Calculate tax
whmcscli tax calculate --amount=100 --country=US --state=TX

# Debug tax matching
whmcscli tax debug --country=US --state=TX --customer=12345

# Test compound tax
whmcscli tax compound-test --amount=100 --tax1=10 --tax2=5
```

## Best Practices

1. **Use clear naming** - Name rules by jurisdiction and type
2. **Set priorities** - Higher priority rules apply first
3. **Test thoroughly** - Verify calculations for all scenarios
4. **Document overrides** - Track tax exemptions and reasons
5. **Stay updated** - Keep rates current with changes
6. **Use tax zones** - For complex geographic configurations

## See Also

- [VAT Handling](./whmcs-vat-handling.md)
- [GST Processing](./whmcs-gst-processing.md)
- [Tax Exempt](./whmcs-tax-exempt.md)
