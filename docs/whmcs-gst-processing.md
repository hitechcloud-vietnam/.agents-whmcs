# WHMCS GST Processing Documentation

## Overview

GST (Goods and Services Tax) processing in WHMCS handles tax compliance for countries that use GST systems, including Australia, Canada, India, New Zealand, and other GST jurisdictions.

## Supported Jurisdictions

### Countries with GST

| Country | GST Rate | Notes |
|---------|----------|-------|
| Australia (AU) | 10% | Standard rate |
| Canada (CA) | 5% GST + Provincial | Combined HST/GST |
| India (IN) | 18% | Standard rate |
| New Zealand (NZ) | 15% | Standard rate |
| Singapore (SG) | 9% | Standard rate |
| Malaysia (MY) | 6% | Standard rate |
| Japan (JP) | 10% | Consumption tax |
| South Korea (KR) | 10% | Value-added tax |

## Configuration

### Enable GST

Navigate to: **Configuration > General Settings > Tax Settings**

```php
// GST Configuration
$gstConfig = [
    'enabled' => true,
    'jurisdictions' => ['AU', 'CA', 'IN', 'NZ', 'SG'],

    // Tax Calculation
    'calculation_basis' => 'destination',    // origin, destination
    'compound_tax' => false,                 // For provinces with HST

    // Compliance
    'record_tax_number' => true,
    'require_tax_number' => false,
    'validate_tax_numbers' => true,

    // Reporting
    'jurisdiction_reporting' => true,
    'tax_inclusive_pricing' => false         // AU-style GST inclusive
];
```

### Country-Specific Settings

```php
// Australia GST
$auGst = [
    'enabled' => true,
    'rate' => 0.10,
    'registration_threshold' => 75000,      // AUD
    'registration_required' => true,
    'gst_free_categories' => [
        'healthcare',
        'education',
        'exports'
    ],
    'input_tax_credit' => true,
    'bas_reporting' => true                  // Business Activity Statement
];

// Canada GST/HST
$caGst = [
    'enabled' => true,
    'federal_gst' => 0.05,
    'provincial_rates' => [
        'ON' => 0.13,    // HST
        'BC' => 0.12,    // GST+PST
        'QC' => 0.14975, // GST+QST
        'AB' => 0.05,    // GST only
        'SK' => 0.11,    // GST+PST
        'MB' => 0.12,    // GST+PST
        'NS' => 0.15,    // HST
        'NB' => 0.15,    // HST
        'PE' => 0.15,    // HST
        'NL' => 0.15,    // HST
        'NT' => 0.05,    // GST only
        'YT' => 0.05,    // GST only
        'NU' => 0.05     // GST only
    ],
    'registration_threshold' => 30000,       // CAD
    'simplified_method' => false              // For small businesses
];

// India GST
$inGst = [
    'enabled' => true,
    'rates' => [
        'exempt' => 0,
        'reduced' => 0.05,
        'standard' => 0.18,
        'higher' => 0.28,
        'special' => 0.12
    ],
    'hsn_codes_required' => false,
    'registration_threshold' => 2000000,     // INR (20 lakhs)
    'composition_scheme' => false,
    'gstin_validation' => true,
    'intra_state' => true,                   // CGST+SGST
    'inter_state' => true,                   // IGST
    'tcs' => 0.001                           // Tax collected at source
];

// New Zealand GST
$nzGst = [
    'enabled' => true,
    'rate' => 0.15,
    'registration_threshold' => 60000,      // NZD
    'offshore_digital' => true,               // Since 2016
    'gst_number_format' => 'NNNNNNNNN'
];
```

## Tax Number Validation

### Australia ABN

```http
POST /billing/gst/validate/abn
```

**Request Body:**

```json
{
  "abn": "12345678901"
}
```

**Response:**

```json
{
  "valid": true,
  "abn": "12345678901",
  "entity_name": "Example Company Pty Ltd",
  "entity_type": "Pty Ltd",
  "status" => "Active",
  "gst_registered" => true,
  "state" => "VIC",
  "validated_at" => "2024-01-15T10:30:00Z"
}
```

### Canada GST Number

```http
POST /billing/gst/validate/canada
```

**Request Body:**

```json
{
  "gst_number": "123456789RT0001"
}
```

### India GSTIN

```http
POST /billing/gst/validate/gstin
```

**Request Body:**

```json
{
  "gstin": "27AABCU9603R1ZM"
}
```

**Response:**

```json
{
  "valid": true,
  "gstin": "27AABCU9603R1ZM",
  "legal_name": "EXAMPLE PRIVATE LIMITED",
  "trade_name": "Example",
  "gst_status" => "Active",
  "registration_date" => "2017-07-01",
  "constitution" => "Private Limited",
  "state_code" => "27",
  "validated_at" => "2024-01-15T10:30:00Z"
}
```

## Tax Calculation

### Destination-Based Calculation

```php
// Calculate GST based on customer location
function calculateGst($amount, $customerCountry, $customerState = null) {
    if ($customerCountry === 'AU') {
        return [
            'rate' => 0.10,
            'amount' => $amount,
            'gst' => $amount * 0.10,
            'total' => $amount * 1.10,
            'jurisdiction' => 'AU-GST'
        ];
    }

    if ($customerCountry === 'CA') {
        $provinceRate = $provincialRates[$customerState] ?? 0.05;
        return [
            'federal_gst' => 0.05,
            'provincial_rate' => $provinceRate - 0.05,
            'amount' => $amount,
            'gst' => $amount * 0.05,
            'provincial_tax' => $amount * ($provinceRate - 0.05),
            'total' => $amount * (1 + $provinceRate),
            'jurisdiction' => "CA-{$customerState}"
        ];
    }

    if ($customerCountry === 'NZ') {
        return [
            'rate' => 0.15,
            'amount' => $amount,
            'gst' => $amount * 0.15,
            'total' => $amount * 1.15,
            'jurisdiction' => 'NZ-GST'
        ];
    }
}
```

### GST-Free Categories

```php
// GST-free products/services
$gstFreeCategories = [
    'AU' => [
        'exports' => true,
        'healthcare' => ['medical', 'hospital', 'dental'],
        'education' => ['courses', 'training'],
        'food' => ['basic_food'],
        'religious' => true
    ],
    'NZ' => [
        'exports' => true,
        'education' => true,
        'healthcare' => true
    ]
];
```

## Invoice Display

### GST Invoice Format

```php
// GST-compliant invoice display
$gstInvoiceDisplay = [
    'show_gst_number' => true,
    'show_customer_gst' => true,
    'show_gst_rate' => true,
    'show_gst_amount' => true,
    'show_jurisdiction' => true,
    'gst_inclusive_label' => 'Total (GST Inclusive)',
    'gst_separate_label' => 'GST'
];
```

### Invoice Line Items

```json
{
  "items": [
    {
      "description": "Cloud Hosting - Monthly",
      "amount_ex_gst": 90.91,
      "gst_rate": 0.10,
      "gst_amount": 9.09,
      "amount": 100.00
    }
  ],
  "subtotal_ex_gst": 90.91,
  "gst": 9.09,
  "total": 100.00,
  "gst_number": "12 345 678 901",
  "invoice_note": "GST has been claimed on a GST-free sale" // If applicable
}
```

## Reporting

### GST Reports

```http
GET /billing/reports/gst
```

**Query Parameters:**
- `period`: monthly, quarterly, yearly
- `jurisdiction`: AU, CA, IN, NZ, SG
- `date_from`: Start date
- `date_to`: End date

**Response (Australia):**

```json
{
  "jurisdiction": "AU",
  "period": "2024-Q1",
  "gst_collected": 15000.00,
  "gst_paid": 5000.00,
  "net_gst_payable": 10000.00,
  "gst_free_sales": 25000.00,
  "export_sales": 10000.00,
  "total_sales": 50000.00,
  "by_category" => [
    {"category" => "standard", "sales" => 15000, "gst" => 1500},
    {"category" => "gst_free", "sales" => 25000, "gst" => 0}
  ]
}
```

### BAS Reporting (Australia)

```php
// Business Activity Statement
$basReport = [
    'g1_total_sales' => 50000.00,
    'gst_collected' => 5000.00,
    'g2_export_sales' => 10000.00,
    'g3_gst_free_sales' => 25000.00,
    'g7_inputs' => 20000.00,
    'gst_paid' => 2000.00,
    'net_gst' => 3000.00,
    'due_date' => '2024-05-28'
];
```

## API Reference

### Calculate GST

```http
POST /billing/gst/calculate
```

**Request Body:**

```json
{
  "amount": 1000.00,
  "jurisdiction": "AU",
  "customer_country": "AU",
  "customer_state": "VIC",
  "product_category": "standard",
  "gst_inclusive": false
}
```

**Response:**

```json
{
  "amount_ex_gst": 1000.00,
  "gst_rate": 0.10,
  "gst_amount": 100.00,
  "total": 1100.00,
  "jurisdiction": "AU-GST"
}
```

### Get GST Rates

```http
GET /billing/gst/rates/{jurisdiction}
```

**Response:**

```json
{
  "jurisdiction": "AU",
  "rates" => [
    {"type" => "standard", "rate" => 0.10, "description" => "Standard Rate"},
    {"type" => "gst_free", "rate" => 0, "description" => "GST Free"},
    {"type" => "input_taxed", "rate" => 0.10, "description" => "Input Taxed"}
  ]
}
```

### Update Customer GST

```http
PATCH /clients/{client_id}/gst
```

**Request Body:**

```json
{
  "jurisdiction": "AU",
  "gst_number": "12345678901",
  "validate" => true,
  "gst_registered" => true,
  "abn" => "12345678901"
}
```

## Customer Portal

### GST Number Entry

**Client Area > Account > Tax Information**

```
+------------------------------------------+
| Tax Information                          |
+------------------------------------------+
| Jurisdiction: Australia                  |
| ABN: [12 345 678 901_____________]      |
| [Validate ABN]                          |
|                                          |
| Status: Valid                           |
| Company: Example Company Pty Ltd         |
| GST Registered: Yes                     |
|                                          |
| [Save]                                   |
+------------------------------------------+
```

## Compliance

### Record Keeping Requirements

```php
// GST record keeping
$gstRecordKeeping = [
    'keep_all_invoices' => true,
    'retention_years' => 7,               // Varies by jurisdiction
    'record_gst_number' => true,
    'record_jurisdiction' => true,
    'log_calculations' => true,
    'store_validations' => true,
    'audit_trail' => true
];
```

### Required Invoice Fields

| Jurisdiction | Required Fields |
|--------------|----------------|
| AU | ABN, GST rate, GST amount |
| CA | GST/HST number, provincial rate |
| IN | GSTIN, HSN code (if required) |
| NZ | GST number |
| SG | GST number |

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Wrong GST rate | State not specified | Specify customer state for Canada |
| GST not calculated | Jurisdiction not enabled | Enable jurisdiction in config |
| Number validation failed | Invalid format | Check number format |
| GST-free not applied | Category not configured | Set product as GST-free |

### Debug Commands

```bash
# Validate GST number
whmcscli gst validate --jurisdiction=AU --number=12345678901

# Calculate GST
whmcscli gst calculate --amount=1000 --jurisdiction=AU --state=VIC

# Generate GST report
whmcscli gst report --jurisdiction=AU --period=quarterly

# Check configuration
whmcscli gst config --jurisdiction=AU
```

## See Also

- [VAT Handling](./whmcs-vat-handling.md)
- [Tax Exempt](./whmcs-tax-exempt.md)
- [Custom Taxes](./whmcs-custom-taxes.md)
