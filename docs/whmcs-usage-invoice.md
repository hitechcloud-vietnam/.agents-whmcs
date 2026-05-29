# WHMCS Usage-Based Invoicing Documentation

## Overview

Usage-based invoicing (also known as metered or consumption billing) calculates charges based on actual resource consumption, enabling flexible pricing for cloud services, bandwidth, storage, and other metered resources.

## Configuration

### Enable Usage Billing

Navigate to: **Configuration > Products/Services > Usage Billing**

```php
// Usage Billing Configuration
$usageConfig = [
    'enabled' => true,
    'billing_model' => ' arrears',     // arrears, in_advance, hybrid
    'metering_enabled' => true,
    'threshold_alerts' => true,

    // Invoice Generation
    'auto_generate' => true,
    'generation_day' => 1,            // Day of month
    'cutoff_day' => 0,                 // 0 = end of previous day

    // Pricing
    'pricing_model' => 'tiered',       // flat, tiered, graduated
    'round_quantities' => true,
    'quantity_decimals' => 0,
    'minimum_charge' => 0.00,

    // Credits
    'carry_over_credits' => true,
    'credit_expiry_days' => 90,
    'never_expire_credits' => false
];
```

### Product Usage Configuration

```php
// Product usage settings
$productUsage = [
    'usage_billing_enabled' => true,
    'bill_in_advance' => false,
    'included_quantity' => 1000,       // Included in base price
    'overage_enabled' => true,
    'minimum_quantity' => 0,
    'maximum_quantity' => null,       // null = unlimited
    'usage_units' => 'API calls',
    'unit_short' => 'calls'
];
```

## Usage Types

### Bandwidth Usage

```php
// Bandwidth billing
$bandwidthUsage = [
    'metric' => 'bandwidth',
    'unit' => 'GB',
    'aggregation' => 'monthly',
    'pricing' => [
        'included' => 100,             // GB included
        'overage_per_gb' => 0.10,
        'tiers' => [
            ['max' => 100, 'price' => 0],
            ['max' => 500, 'price' => 0.10],
            ['max' => null, 'price' => 0.08]
        ]
    ],
    'reset_on_billing' => false,
    'carry_forward' => false
];
```

### Storage Usage

```php
// Storage billing
$storageUsage = [
    'metric' => 'storage',
    'unit' => 'GB',
    'aggregation' => 'daily_average',
    'pricing' => [
        'included' => 50,
        'overage_per_gb' => 0.05,
        'tiers' => [
            ['max' => 50, 'price' => 0],
            ['max' => 200, 'price' => 0.05],
            ['max' => null, 'price' => 0.03]
        ]
    ]
];
```

### API Calls / Requests

```php
// API call billing
$apiUsage = [
    'metric' => 'api_calls',
    'unit' => 'calls',
    'aggregation' => 'total',
    'pricing' => [
        'included' => 10000,
        'tiers' => [
            ['max' => 10000, 'price' => 0],
            ['max' => 100000, 'price' => 0.001],
            ['max' => 1000000, 'price' => 0.0005],
            ['max' => null, 'price' => 0.0001]
        ]
    ]
];
```

### Compute Hours

```php
// Compute time billing
$computeUsage = [
    'metric' => 'compute_hours',
    'unit' => 'hours',
    'aggregation' => 'sum',
    'pricing' => [
        'included' => 0,
        'flat_rate_per_hour' => 0.05,
        'tiers' => [
            ['max' => 100, 'price' => 0.05],
            ['max' => 500, 'price' => 0.04],
            ['max' => null, 'price' => 0.03]
        ]
    ]
];
```

## Pricing Models

### Flat Rate Overage

```php
// Flat rate for all overage
$flatOverage = [
    'model' => 'flat',
    'included_quantity' => 1000,
    'overage_price' => 0.01,
    'example' => '1000 included + 500 overage = $5.00'
];
```

### Tiered Pricing

```php
// Volume-based tiers
$tieredPricing = [
    'model' => 'tiered',
    'tiers' => [
        ['min' => 0, 'max' => 1000, 'price' => 0, 'type' => 'included'],
        ['min' => 1001, 'max' => 5000, 'price' => 0.008, 'type' => 'tier'],
        ['min' => 5001, 'max' => 10000, 'price' => 0.006, 'type' => 'tier'],
        ['min' => 10001, 'max' => null, 'price' => 0.004, 'type' => 'tier']
    ]
];

// Example: 7500 units
// Tier 1: 1000 * $0 = $0
// Tier 2: 4000 * $0.008 = $32
// Tier 3: 2500 * $0.006 = $15
// Total: $47
```

### Graduated Pricing

```php
// Each unit priced according to its tier
$graduatedPricing = [
    'model' => 'graduated',
    'tiers' => [
        ['min' => 0, 'max' => 1000, 'price' => 0.001],
        ['min' => 1000, 'max' => 5000, 'price' => 0.002],
        ['min' => 5000, 'max' => null, 'price' => 0.003]
    ]
];
```

## Metering & Collection

### Usage Data Collection

```php
// Usage collection configuration
$collectionConfig = [
    'method' => 'push',              // push, pull
    'interval' => 300,              // 5 minutes
    'batch_size' => 100,
    'compression' => true,

    // Push endpoint
    'push_endpoint' => 'https://api.whmcs.com/usage',
    'push_auth' => 'api_key',

    // Pull configuration
    'pull_sources' => [
        ['name' => 'cloud_provider', 'type' => 'api', 'config' => [...]],
        ['name' => 'custom_app', 'type' => 'database', 'config' => [...]]
    ]
];
```

### Record Usage

```http
POST /billing/usage/record
```

**Request Body:**

```json
{
  "user_id": 12345,
  "product_id": 67890,
  "metric": "bandwidth",
  "quantity": 1.5,
  "timestamp": "2024-01-15T10:30:00Z",
  "metadata": {
    "server_id": "srv-001",
    "region": "us-east"
  }
}
```

### Batch Record Usage

```http
POST /billing/usage/record/batch
```

**Request Body:**

```json
{
  "records": [
    {"user_id": 12345, "product_id": 67890, "metric": "bandwidth", "quantity": 1.5},
    {"user_id": 12345, "product_id": 67890, "metric": "storage", "quantity": 0.5},
    {"user_id": 67890, "product_id": 11111, "metric": "api_calls", "quantity": 100}
  ]
}
```

### Usage Query

```http
GET /billing/usage/{user_id}/{product_id}
```

**Query Parameters:**
- `metric`: Filter by metric
- `from`: Start date
- `to`: End date
- `aggregation`: hourly, daily, monthly

**Response:**

```json
{
  "user_id": 12345,
  "product_id": 67890,
  "period": {
    "from": "2024-01-01",
    "to": "2024-01-15"
  },
  "usage": [
    {
      "metric": "bandwidth",
      "unit": "GB",
      "total": 150.5,
      "breakdown": {
        "included": 50,
        "overage": 100.5
      },
      "cost": 10.05
    }
  ]
}
```

## Invoice Generation

### Usage Invoice Line Items

```php
// Usage invoice structure
$usageInvoice = [
    'items' => [
        [
            'description' => 'Bandwidth Usage - January 2024',
            'type' => 'usage',
            'metric' => 'bandwidth',
            'quantity' => 150.5,
            'unit' => 'GB',
            'breakdown' => [
                'included' => 100,
                'billable' => 50.5
            ],
            'pricing' => [
                'included_cost' => 0,
                'overage_cost' => 5.05
            ],
            'total' => 5.05
        ],
        [
            'description' => 'API Calls - January 2024',
            'type' => 'usage',
            'metric' => 'api_calls',
            'quantity' => 25000,
            'unit' => 'calls',
            'breakdown' => [...],
            'total' => 15.00
        ]
    ],
    'totals' => [
        'subtotal' => 20.05,
        'tax' => 1.80,
        'total' => 21.85
    ]
];
```

### Usage Invoice Template

```html
<!-- Usage Invoice Line Item Display -->
<div class="usage-item">
    <h4>Bandwidth Usage - January 2024</h4>
    <table class="usage-details">
        <tr>
            <td>Total Usage:</td>
            <td>150.5 GB</td>
        </tr>
        <tr>
            <td>Included in Plan:</td>
            <td>100 GB</td>
        </tr>
        <tr>
            <td>Overage:</td>
            <td>50.5 GB @ $0.10/GB</td>
        </tr>
        <tr class="total">
            <td>Bandwidth Total:</td>
            <td>$5.05</td>
        </tr>
    </table>
</div>
```

## API Reference

### Get Usage Summary

```http
GET /billing/usage/summary/{user_id}
```

**Response:**

```json
{
  "user_id": 12345,
  "current_period": {
    "start" => "2024-01-01",
    "end" => "2024-01-31",
    "days_remaining" => 16
  },
  "usage": [
    {
      "product_id": 67890,
      "product_name": "Cloud Server",
      "metrics" => [
        {
          "metric" => "bandwidth",
          "used" => 75.5,
          "included" => 100,
          "remaining" => 24.5,
          "projected" => 150,
          "projected_overage" => 50
        }
      ]
    }
  ]
}
```

### Get Usage Forecast

```http
GET /billing/usage/forecast/{user_id}/{product_id}
```

**Response:**

```json
{
  "user_id": 12345,
  "product_id": 67890,
  "metric": "bandwidth",
  "forecast": {
    "daily_average" => 5.03,
    "days_remaining" => 16,
    "projected_usage" => 155.5,
    "projected_overage" => 55.5,
    "projected_cost" => 5.55,
    "confidence" => 0.85
  }
}
```

### Set Usage Alert

```http
POST /billing/usage/alerts
```

**Request Body:**

```json
{
  "user_id": 12345,
  "product_id": 67890,
  "metric": "bandwidth",
  "threshold" => 80,
  "threshold_type" => "percentage",    // percentage, absolute
  "notification_channels" => ["email", "sms"],
  "recipients" => ["user@example.com"]
}
```

## Customer Portal

### Usage Dashboard

**Client Area > My Services > Usage**

```
+------------------------------------------------------------------+
|  Usage Dashboard - Cloud Server                                   |
+------------------------------------------------------------------+
|                                                                  |
|  Current Period: January 1 - January 31, 2024                   |
|  Days Remaining: 16                                             |
|                                                                  |
|  +--------------------------------------------------------------+|
|  | Bandwidth (GB)                                               ||
|  | Used: 75.5 / 100 GB                                        ||
|  | [===================                        ] 75.5%        ||
|  | Projected: 150 GB | Overage: ~50 GB ($5.00 est)             ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  +--------------------------------------------------------------+|
|  | API Calls                                                   ||
|  | Used: 15,000 / 10,000                                      ||
|  | [====================] 150%                                 ||
|  | Projected: 30,000 | Overage: 20,000 ($10.00 est)            ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  Estimated Bill: $15.00 + $5.00 + $10.00 = $30.00              |
|                                                                  |
|  [View Details] [Usage History] [Upgrade Plan]                  |
+------------------------------------------------------------------+
```

## Threshold Alerts

### Alert Configuration

```php
// Usage threshold alerts
$alertConfig = [
    'enabled' => true,
    'alert_levels' => [
        ['percentage' => 50, 'message' => 'You have used 50% of your allowance'],
        ['percentage' => 75, 'message' => 'Warning: 75% of allowance used'],
        ['percentage' => 90, 'message' => 'Critical: 90% of allowance used'],
        ['percentage' => 100, 'message' => 'You have exceeded your included allowance']
    ],
    'notification_methods' => ['email', 'sms'],
    'notification_frequency' => 'once',    // once, daily, always
    'include_forecast' => true
];
```

## Reporting

### Usage Reports

```http
GET /billing/reports/usage
```

**Query Parameters:**
- `period`: daily, weekly, monthly
- `group_by`: user, product, metric
- `date_from`: Start date
- `date_to`: End date

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_usage_cost": 45230.00,
    "total_quantity": 150000,
    "by_metric" => [
      {"metric" => "bandwidth", "quantity" => 50000, "cost" => 2500},
      {"metric" => "api_calls", "quantity" => 100000, "cost" => 500}
    ]
  },
  "top_users" => [
    {"user_id" => 12345, "cost" => 1500},
    {"user_id" => 67890, "cost" => 1200}
  ]
}
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Missing usage data | Collection failure | Check API credentials |
| Wrong totals | Aggregation error | Verify aggregation method |
| Overage not calculated | Pricing not configured | Set up pricing tiers |
| Alert not sent | Notification misconfigured | Check alert settings |

### Debug Commands

```bash
# Check usage data
whmcscli usage view --user_id=12345 --product_id=67890 --metric=bandwidth

# Verify collection
whmcscli usage collection-status --metric=bandwidth

# Calculate invoice
whmcscli usage invoice --user_id=12345 --period=2024-01
```

## See Also

- [Recurring Invoice](./whmcs-recurring-invoice.md)
- [Tiered Pricing](./whmcs-tiered-pricing.md)
- [Volume Discounts](./whmcs-volume-discounts.md)
