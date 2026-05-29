# WHMCS Early Termination Fees Documentation

## Overview

Early termination fees (ETF) are charges applied when customers cancel their services before the end of their contract term. This helps recover setup costs and compensates for lost revenue.

## Configuration

### Enable Early Termination Fees

Navigate to: **Configuration > General Settings > Termination Fees**

```php
// Early Termination Fee Configuration
$etfConfig = [
    'enabled' => true,
    'default_fee_type' => 'percentage',  // fixed, percentage, prorated

    // Fee Calculation
    'calculate_from_remaining' => true,
    'minimum_fee' => 0,
    'maximum_fee' => null,

    // Applies To
    'apply_to_all_products' => false,
    'products_exempt' => [],
    'products_included' => ['hosting', 'dedicated', 'vps'],

    // Notice Requirements
    'require_advance_notice' => true,
    'minimum_notice_days' => 30,
    'notice_waiver_fee' => 0
];
```

## Fee Calculation Methods

### Fixed Fee

```php
// Fixed early termination fee
$fixedETF = [
    'type' => 'fixed',
    'amount' => 99.00,
    'description' => 'Early cancellation fee',
    'prorate_remaining' => false
];
```

### Percentage of Remaining

```php
// Percentage of remaining contract value
$percentageETF = [
    'type' => 'percentage',
    'percentage' => 50,               // 50% of remaining value
    'of_remaining' => true,            // Of remaining contract value
    'minimum_fee' => 25.00,
    'maximum_fee' => 200.00
];

// Example:
// Contract: $100/month for 12 months = $1200 total
// Cancelled after 3 months ($300 paid)
// Remaining: $900 (9 months)
// ETF at 50%: $450
```

### Flat + Percentage

```php
// Combination fee
$combinedETF = [
    'type' => 'combined',
    'flat_fee' => 50.00,
    'percentage' => 25,               // 25% of remaining
    'minimum_fee' => 50.00,
    'maximum_fee' => 300.00
];
```

### Monthly Value Equivalent

```php
// Fee equals remaining months (up to max)
$monthlyEquivalentETF = [
    'type' => 'monthly_equivalent',
    'months_remaining' => 'all',      // All remaining months
    'max_months' => 3,               // Cap at 3 months
    'prorate_partial_month' => true
];

// Example:
// Remaining: 6 months @ $50/month = $300
// ETF: Min(6, 3) months = $150
```

## Contract Types

### Annual Contracts

```php
// Annual contract termination
$annualContractETF = [
    'contract_length' => 12,           // months
    'setup_fee' => 99.00,            // Non-refundable
    'early_termination_fee' => [
        'type' => 'percentage',
        'months' => [
            ['months_remaining' => 12, 'percentage' => 100],
            ['months_remaining' => 9, 'percentage' => 75],
            ['months_remaining' => 6, 'percentage' => 50],
            ['months_remaining' => 3, 'percentage' => 25],
            ['months_remaining' => 0, 'percentage' => 0]
        ]
    ],
    'waiver_period_days' => 30,       // Days to cancel without fee
    'pro_rata_setup_refund' => false
];
```

### Multi-Year Contracts

```php
// Multi-year contract
$multiYearETF = [
    'contract_length' => 24,          // months
    'discount' => 20,                // 20% discount vs monthly
    'early_termination' => [
        'within_year_1' => [
            'fee_type' => 'remaining_year_1',
            'minimum_months_charged' => 12
        ],
        'within_year_2' => [
            'fee_type' => 'remaining_contract',
            'percentage' => 50
        ]
    ]
];
```

### Month-to-Month

```php
// Month-to-month (no ETF)
$monthToMonthETF = [
    'contract_type' => 'monthly',
    'termination_fee' => 0,
    'notice_required' => true,
    'notice_days' => 30,
    'billing_end_date' => 'end_of_notice_period'
];
```

## Product-Specific Fees

### Hosting ETF

```php
// Hosting product termination fees
$hostingETF = [
    'product_type' => 'hosting',
    'setup_fee' => 0,
    'contract_options' => [
        'monthly' => ['termination_fee' => 0],
        'annual' => [
            'discount' => 20,
            'termination_fee' => [
                'type' => 'percentage',
                'percentage' => 50,
                'of_remaining' => true,
                'minimum' => 25
            ]
        ]
    ]
];
```

### Dedicated Server ETF

```php
// Dedicated server termination fees
$dedicatedServerETF = [
    'product_type' => 'dedicated',
    'setup_fee' => 199.00,
    'contract_options' => [
        'monthly' => [
            'termination_fee' => 99.00,
            'notice_days' => 30
        ],
        'annual' => [
            'discount' => 15,
            'termination_fee' => [
                'type' => 'combined',
                'flat' => 199.00,
                'percentage' => 25,
                'of_remaining' => true
            ]
        )
    ],
    'hardware_cost_recovery' => true,
    'hardware_cost' => 500.00,
    'hardware_recovery_years' => 2
];
```

### SSL Certificate ETF

```php
// SSL certificate termination
$sslETF = [
    'product_type' => 'ssl',
    'termination_fee' => 'full_price',  // No refund
    'reason_exceptions' => [
        'issuer_revocation' => 'full_refund',
        'technical_issue' => 'pro_rata_refund'
    ]
];
```

## API Reference

### Calculate Early Termination Fee

```http
POST /billing/termination-fee/calculate
```

**Request Body:**

```json
{
  "service_id": 67890,
  "termination_date": "2024-02-01",
  "include_setup_refund" => false
}
```

**Response:**

```json
{
  "service_id": 67890,
  "contract": {
    "type" => "annual",
    "start_date" => "2024-01-01",
    "end_date" => "2024-12-31",
    "monthly_value" => 99.00,
    "total_contract_value" => 1188.00
  },
  "termination": {
    "requested_date" => "2024-02-01",
    "days_remaining" => 334,
    "months_remaining" => 11,
    "remaining_value" => 1089.00
  },
  "termination_fee": {
    "type" => "percentage",
    "percentage" => 50,
    "calculation" => "remaining_value * percentage",
    "amount" => 544.50,
    "minimum_applied" => false,
    "maximum_applied" => false,
    "final_fee" => 544.50
  },
  "credits" => {
    "setup_fee_credit" => 0,
    "total_credits" => 0
  },
  "summary": {
    "amount_due" => 544.50,
    "service_end_date" => "2024-02-01"
  }
}
```

### Process Early Termination

```http
POST /billing/termination/early
```

**Request Body:**

```json
{
  "service_id": 67890,
  "termination_date": "2024-02-01",
  "reason" => "customer_request",
  "create_invoice" => true,
  "apply_credits" => true,
  "backup_requested" => true,
  "customer_confirmed" => true
}
```

## Customer Portal

### View Termination Fee

**Client Area > My Services > Cancel Service**

```
+------------------------------------------------------------------+
|  Cancel Premium Hosting                                           |
+------------------------------------------------------------------+
|                                                                  |
|  Current Contract: Annual (12 months)                             |
|  Contract Period: January 1, 2024 - December 31, 2024            |
|  Monthly Value: $99.00                                          |
|  Amount Paid: $297.00 (3 months)                               |
|                                                                  |
|  +--------------------------------------------------------------+|
|  | EARLY TERMINATION CALCULATION                                ||
|  |                                                              ||
|  | Remaining Contract: 9 months                                ||
|  | Remaining Value: $891.00                                    ||
|  |                                                              ||
|  | Early Termination Fee (50% of remaining): $445.50          ||
|  |                                                              ||
|  | Final Amount Due: $445.50                                   ||
|  |                                                              ||
|  | Your service will end on February 1, 2024                   ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  [Confirm Cancellation] [Keep Service]                           |
+------------------------------------------------------------------+
```

### Cancellation Flow

```
1. Customer requests cancellation
2. System checks contract type
3. If contract: Calculate ETF
4. Display ETF to customer
5. Customer confirms
6. Generate termination invoice
7. Customer pays ETF
8. Schedule termination
9. Execute termination
10. Send confirmation
```

## Waiver Policies

### Waive ETF Conditions

```php
// Early termination fee waiver
$waiverPolicy = [
    'enabled' => true,

    // Automatic Waivers
    'auto_waive_conditions' => [
        'service_issue_unresolved' => [
            'enabled' => true,
            'open_ticket_required' => true,
            'ticket_age_days' => 14
        ],
        'provider_breach' => [
            'enabled' => true,
            'sla_violation_required' => true
        ],
        'price_increase' => [
            'enabled' => true,
            'increase_threshold' => 10  // percent
        ],
        'force_majeure' => [
            'enabled' => true,
            'requires_documentation' => true
        ]
    ],

    // Manual Waiver
    'manual_waiver_allowed' => true,
    'waiver_approval_required' => true,
    'waiver_approval_role' => 'billing_manager',

    // Partial Waiver
    'partial_waiver_allowed' => true,
    'minimum_waiver_percentage' => 25,
    'maximum_waiver_percentage' => 100
];
```

### Request Waiver

```http
POST /billing/termination-fee/waive-request
```

**Request Body:**

```json
{
  "service_id": 67890,
  "reason" => "service_issue_unresolved",
  "ticket_id" => "TICK-12345",
  "explanation" => "Website has been down for 3 weeks"
}
```

## Contract Enforcement

### Contract Display

```php
// Contract information on product
$contractDisplay = [
    'show_on_product_page' => true,
    'show_on_invoice' => true,
    'show_in_client_area' => true,
    'terms_link' => '/terms/contracts',

    'display_format' => [
        'type' => 'Annual Contract',
        'term' => '12 months',
        'monthly_value' => '$99.00/month',
        'total_value' => '$1,188.00 billed annually',
        'early_termination' => '50% of remaining value'
    ]
];
```

### Contract Confirmation

```php
// Contract signup confirmation
$contractConfirmation = [
    'require_acknowledgment' => true,
    'acknowledgment_text' => 'I understand that early termination fees
                              may apply if I cancel before the end of
                              my contract term.',

    'display_summary' => true,
    'summary_includes' => [
        'contract_length',
        'monthly_value',
        'total_value',
        'termination_fee',
        'cancellation_terms'
    ]
];
```

## Pro-Rated Credits

### Calculate Pro-Rated Credit

```php
// Credit for unused service
$proRateCredit = [
    'enabled' => true,
    'calculate_from' => 'termination_date',
    'include_setup_fee' => false,      // Usually non-refundable
    'prorate_method' => 'daily',

    // Example
    'example' => [
        'monthly_price' => 99.00,
        'days_in_month' => 30,
        'days_used' => 10,
        'days_remaining' => 20,
        'credit_amount' => (99.00 / 30) * 20 = 66.00
    ]
];
```

### Full Settlement

```php
// Settlement calculation
$fullSettlement = [
    'remaining_value' => 891.00,       // 9 months remaining
    'early_termination_fee' => 445.50,  // 50%
    'setup_fee_credit' => 0,          // Non-refundable
    'pro_rated_credit' => 0,          // ETF replaces credit

    'total_due' => 445.50
];
```

## Exceptions

### Geographic Exceptions

```php
// Region-specific rules
$geoExceptions = [
    'EU' => [
        'cancellation_rights' => true,  // Consumer protection
        'cooling_off_days' => 14,
        'termination_fee_rules' => [
            'max_fee_limited' => true,
            'max_fee_percentage' => 10   // Max 10% of remaining
        ]
    ],
    'California' => [
        'consumer_protection' => true,
        'maximum_etf' => [
            'hosting' => 75.00,
            'other' => 100.00
        ]
    ]
];
```

### Customer Segment Exceptions

```php
// Customer segment exceptions
$customerExceptions = [
    'enterprise' => [
        'negotiated_terms' => true,
        'custom_etf' => true,
        'no_etf' => false               // Still has ETF but negotiable
    ],
    'government' => [
        'no_etf' => true,
        'require_approval' => false
    ],
    'nonprofit' => [
        'reduced_etf' => true,
        'reduction_percentage' => 50
    ]
];
```

## Reporting

### ETF Report

```http
GET /billing/reports/termination-fees
```

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_etf_assessed" => 5000.00,
    "total_etf_collected" => 3500.00,
    "total_etf_waived" => 1500.00,
    "early_cancellations" => 25
  },
  "by_product" => [
    {"product" => "hosting", "assessed" => 3000, "collected" => 2500, "waived" => 500},
    {"product" => "dedicated", "assessed" => 2000, "collected" => 1000, "waived" => 1000}
  ],
  "waiver_reasons" => [
    {"reason" => "service_issue", "amount" => 1000},
    {"reason" => "price_increase", "amount" => 300},
    {"reason" => "manual_waiver", "amount" => 200}
  ]
}
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| ETF not calculating | Product not configured | Enable ETF for product |
| Wrong percentage | Configuration error | Verify fee settings |
| Customer disputes ETF | Unclear policy | Update terms display |
| ETF not collecting | Payment failure | Check payment processing |

### Debug Commands

```bash
# Calculate ETF
whmcscli etf calculate --service_id=67890

# View ETF configuration
whmcscli etf config --product_id=100

# Apply waiver
whmcscli etf waive --service_id=67890 --reason="service_issue"

# Generate termination invoice
whmcscli etf invoice --service_id=67890
```

## Best Practices

1. **Be transparent** - Clearly display ETF before purchase
2. **Set reasonable fees** - Balance recovery with customer experience
3. **Have waiver policies** - Handle exceptional circumstances
4. **Communicate early** - Remind of contract end dates
5. **Offer alternatives** - Downsells before termination
6. **Document everything** - Maintain clear records

## See Also

- [Termination Policy](./whmcs-termination-policy.md)
- [Refund Policy](./whmcs-refund-policy.md)
- [Credit Policy](./whmcs-credit-policy.md)
