# WHMCS VAT Handling Documentation

## Overview

VAT (Value Added Tax) handling in WHMCS manages European Union VAT requirements, including VAT number validation, reverse charge mechanisms, and compliance with EU VAT directives.

## Configuration

### Enable VAT Handling

Navigate to: **Configuration > General Settings > Tax Settings**

```php
// VAT Configuration
$vatConfig = [
    'enabled' => true,
    'eu_countries' => [
        'AT', 'BE', 'BG', 'HR', 'CY', 'CZ', 'DK', 'EE',
        'FI', 'FR', 'DE', 'GR', 'HU', 'IE', 'IT', 'LV',
        'LT', 'LU', 'MT', 'NL', 'PL', 'PT', 'RO', 'SK',
        'SI', 'ES', 'SE'
    ],

    // VAT Rates
    'default_vat_rate' => null,        // Use country-specific rates
    'reduced_vat_rate' => null,         // For qualifying products
    'super_reduced_vat_rate' => null,   // For specific categories

    // VAT Number Validation
    'validate_vat_numbers' => true,
    'validation_service' => 'vi_es',   // VAT Information Exchange System
    'auto_validate_on_save' => true,
    'cache_validation' => true,
    'cache_hours' => 24,

    // Reverse Charge
    'enable_reverse_charge' => true,
    'reverse_charge_threshold' => 10000,  // B2B threshold

    // Customer Handling
    'default_customer_vat_status' => 'personal',
    'detect_business_customer' => true
];
```

### VAT Rates by Country

```php
// Current EU VAT rates (standard rates)
$vatRates = [
    'AT' => ['standard' => 20, 'reduced' => 10, 'super_reduced' => null],
    'BE' => ['standard' => 21, 'reduced' => 12, 'super_reduced' => 6],
    'BG' => ['standard' => 20, 'reduced' => 9, 'super_reduced' => null],
    'HR' => ['standard' => 25, 'reduced' => 13, 'super_reduced' => null],
    'CY' => ['standard' => 19, 'reduced' => 9, 'super_reduced' => 5],
    'CZ' => ['standard' => 21, 'reduced' => 15, 'super_reduced' => null],
    'DK' => ['standard' => 25, 'reduced' => null, 'super_reduced' => null],
    'EE' => ['standard' => 22, 'reduced' => 9, 'super_reduced' => null],
    'FI' => ['standard' => 24, 'reduced' => 14, 'super_reduced' => null],
    'FR' => ['standard' => 20, 'reduced' => 10, 'super_reduced' => 2.1],
    'DE' => ['standard' => 19, 'reduced' => 7, 'super_reduced' => null],
    'GR' => ['standard' => 24, 'reduced' => 13, 'super_reduced' => null],
    'HU' => ['standard' => 27, 'reduced' => 18, 'super_reduced' => 5],
    'IE' => ['standard' => 23, 'reduced' => 13.5, 'super_reduced' => 4.8],
    'IT' => ['standard' => 22, 'reduced' => 10, 'super_reduced' => 4],
    'LV' => ['standard' => 21, 'reduced' => 12, 'super_reduced' => null],
    'LT' => ['standard' => 21, 'reduced' => 9, 'super_reduced' => 5],
    'LU' => ['standard' => 17, 'reduced' => 14, 'super_reduced' => 3],
    'MT' => ['standard' => 18, 'reduced' => 7, 'super_reduced' => 5],
    'NL' => ['standard' => 21, 'reduced' => 9, 'super_reduced' => null],
    'PL' => ['standard' => 23, 'reduced' => 8, 'super_reduced' => 5],
    'PT' => ['standard' => 23, 'reduced' => 13, 'super_reduced' => 6],
    'RO' => ['standard' => 19, 'reduced' => 9, 'super_reduced' => 5],
    'SK' => ['standard' => 20, 'reduced' => 10, 'super_reduced' => null],
    'SI' => ['standard' => 22, 'reduced' => 9.5, 'super_reduced' => null],
    'ES' => ['standard' => 21, 'reduced' => 10, 'super_reduced' => 4],
    'SE' => ['standard' => 25, 'reduced' => 12, 'super_reduced' => 6]
];
```

## VAT Number Validation

### Validate VAT Number

```http
POST /billing/vat/validate
```

**Request Body:**

```json
{
  "vat_number": "DE123456789",
  "country_code": "DE",
  "company_name": "Example GmbH",
  "validate_with_name" => false
}
```

**Response:**

```json
{
  "valid": true,
  "vat_number": "DE123456789",
  "country_code": "DE",
  "company_name": "Example GmbH",
  "company_address": "Street Name 1, City, Postal Code, Country",
  "registration_date": "2020-01-15",
  "vat_exemption_applies": false,
  "valid_at" => "2024-01-15T10:30:00Z",
  "source" => "VIES"
}
```

### VAT Validation Settings

```php
// VAT number validation configuration
$vatValidation = [
    'enabled' => true,
    'service' => 'vi_es',
    'vi_es_url' => 'https://ec.europa.eu/taxation_customs/vies',
    'validate_on_save' => true,
    'validate_on_checkout' => true,
    'strict_mode' => false,           // Require exact name match
    'name_match_threshold' => 0.7,    // 70% similarity
    'allow_pending' => true,           // Allow if validation pending
    'cache_results' => true,
    'cache_duration' => 86400          // 24 hours
];
```

## VAT Calculation

### B2C (Business to Consumer)

When selling to a consumer in an EU country:

```php
// B2C VAT calculation
$b2cCalc = [
    'customer_type' => 'individual',
    'customer_country' => 'DE',
    'supplier_country' => 'IE',
    'vat_rule' => 'customer_location',
    'vat_rate' => 19,                  // Germany's rate
    'tax_amount' => $amount * 0.19
];
```

### B2B with Valid VAT Number (EU)

When selling to a business with a valid EU VAT number:

```php
// B2B VAT calculation (Reverse Charge)
$b2bCalc = [
    'customer_type' => 'business',
    'customer_country' => 'DE',
    'supplier_country' => 'IE',
    'vat_number' => 'DE123456789',
    'vat_valid' => true,
    'vat_rule' => 'reverse_charge',
    'vat_amount' => 0,                 // Customer pays own VAT
    'invoice_note' => 'Reverse charge: Customer VAT number DE123456789'
];
```

### B2B Outside EU

```php
// B2B outside EU - No VAT
$b2bOutsideEu = [
    'customer_type' => 'business',
    'customer_country' => 'US',
    'supplier_country' => 'IE',
    'vat_rule' => 'outside_scope',
    'vat_amount' => 0,
    'tax_note' => 'Outside scope of EU VAT'
];
```

### OSS (One-Stop Shop)

For digital services to EU consumers:

```php
// OSS VAT handling
$ossConfig = [
    'enabled' => true,
    'scheme' => 'non_union_oss',       // non_union, union, import
    'registration_country' => 'IE',
    'registration_number' => 'IM1234567890',
    'auto_determine_rate' => true,
    'report_frequency' => 'quarterly'
];
```

## Reverse Charge Mechanism

### Enable Reverse Charge

```php
// Reverse charge configuration
$reverseCharge = [
    'enabled' => true,
    'threshold' => 0,                   // 0 = always apply for B2B
    'applies_to' => ['b2b_with_vat', 'b2b_government'],
    'requires_valid_vat' => true,
    'display_on_invoice' => true,
    'invoice_note' => 'Reverse charge - Customer to account for VAT'
];
```

### Reverse Charge Invoice

```php
// Invoice with reverse charge
$reverseChargeInvoice = [
    'items' => [
        ['description' => 'Cloud Services', 'amount' => 1000.00, 'tax' => 0]
    ],
    'subtotal' => 1000.00,
    'vat' => [
        'rate' => 0,
        'amount' => 0,
        'mechanism' => 'reverse_charge',
        'note' => 'Reverse charge applied - EU B2B'
    ],
    'total' => 1000.00,
    'invoice_note' => 'VAT reverse charge: According to Art. 196 Directive 2006/112/EC,
                       the customer is liable to account for VAT.'
];
```

## Customer VAT Settings

### Store VAT Information

```php
// Client VAT data
$clientVatData = [
    'client_id' => 12345,
    'vat_number' => 'DE123456789',
    'vat_validated' => true,
    'validated_at' => '2024-01-15T10:30:00Z',
    'company_name' => 'Example GmbH',
    'company_address' => 'Street 1, City, 12345',
    'customer_type' => 'business',     // business, individual
    'tax_exempt' => false,
    'vi_es_response' => [...],
    'tax_jurisdiction' => 'EU'
];
```

### VAT Customer Classification

```php
// Customer classification rules
$classificationRules = [
    'business' => [
        'has_valid_vat_number' => true,
        'verify_vat_number' => true
    ],
    'individual' => [
        'has_valid_vat_number' => false,
        'tax_at_customer_location' => true
    ],
    'government' => [
        'vat_exempt_check' => true,
        'reverse_charge' => true
    ],
    'non_profit' => [
        'vat_exempt_check' => true,
        'reduced_rate_check' => true
    ]
];
```

## API Reference

### Get VAT Rate

```http
GET /billing/vat/rate/{country_code}
```

**Response:**

```json
{
  "country_code": "DE",
  "country_name": "Germany",
  "standard_rate" => 19,
  "reduced_rate" => 7,
  "super_reduced_rate" => null,
  "parking_rate" => null,
  "effective_date" => "2024-01-01"
}
```

### Calculate VAT

```http
POST /billing/vat/calculate
```

**Request Body:**

```json
{
  "amount" => 1000.00,
  "country_code" => "DE",
  "customer_type" => "business",
  "vat_number" => "DE123456789",
  "product_type" => "digital"          // digital, physical, service
}
```

**Response:**

```json
{
  "amount_excluding_vat" => 1000.00,
  "vat_rate" => 0,
  "vat_amount" => 0,
  "amount_including_vat" => 1000.00,
  "vat_rule" => "reverse_charge",
  "customer_location" => "DE",
  "vat_number_valid" => true
}
```

### Update Client VAT

```http
PATCH /clients/{client_id}/vat
```

**Request Body:**

```json
{
  "vat_number" => "DE123456789",
  "validate" => true,
  "customer_type" => "business",
  "company_name" => "Example GmbH"
}
```

## OSS Reporting

### OSS Registration

```php
// One-Stop Shop configuration
$ossConfig = [
    'enabled' => true,
    'registration_country' => 'IE',
    'oss_number' => 'IM1234567890',
    'scheme' => 'union_oss',
    'reporting_period' => 'quarterly',
    'auto_submit' => false
];
```

### OSS Sales Report

```http
GET /billing/reports/oss
```

**Response:**

```json
{
  "period" => "2024-Q1",
  "oss_number" => "IM1234567890",
  "sales" => [
    {
      "country_code" => "DE",
      "vat_rate" => 19,
      "total_sales" => 50000.00,
      "vat_collected" => 9500.00,
      "customer_count" => 150
    },
    {
      "country_code" => "FR",
      "vat_rate" => 20,
      "total_sales" => 30000.00,
      "vat_collected" => 6000.00,
      "customer_count" => 80
    }
  ],
  "totals" => {
    "total_sales" => 80000.00,
    "total_vat" => 15500.00
  }
}
```

## Compliance

### Invoice Requirements

```php
// VAT-compliant invoice requirements
$vatInvoiceRequirements = [
    'show_vat_number' => true,
    'show_customer_vat_number' => true,
    'show_vat_rate' => true,
    'show_reverse_charge_notice' => true,
    'show_country_of_supply' => true,
    'show_oss_number' => true,         // If using OSS
    'required_fields' => [
        'supplier_vat',
        'customer_vat (if B2B)',
        'vat_rate',
        'vat_amount',
        'reverse_charge_notice (if applicable)'
    ]
];
```

### Record Keeping

```php
// VAT record keeping
$vatRecordKeeping = [
    'keep_all_transactions' => true,
    'retention_years' => 10,
    'log_vat_calculations' => true,
    'store_vat_validations' => true,
    'quarterly_backup' => true
];
```

## Customer Portal

### VAT Number Entry

**Client Area > Checkout > Billing Details**

```
+------------------------------------------+
| Business Details (Optional)              |
+------------------------------------------+
| Country: [Germany___________________]    |
| VAT Number: [DE 123456789___________]   |
| [Validate VAT Number]                    |
|                                          |
| Status: Valid - Example GmbH             |
| [Update]                                 |
+------------------------------------------+
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| VAT number rejected | Invalid format | Check number format |
| VIES timeout | Service unavailable | Retry later |
| Wrong VAT applied | Classification error | Update customer type |
| Reverse charge not applied | Missing VAT number | Validate VAT number |

### Debug Commands

```bash
# Validate VAT number
whmcscli vat validate --number=DE123456789

# Check VAT calculation
whmcscli vat calculate --amount=1000 --country=DE --type=business

# View VIES status
whmcscli vat vies-status

# Force VAT recalculation
whmcscli invoice recalculate-vat --invoice_id=INV-12345
```

## See Also

- [Tax Exempt](./whmcs-tax-exempt.md)
- [GST Processing](./whmcs-gst-processing.md)
- [Custom Taxes](./whmcs-custom-taxes.md)
