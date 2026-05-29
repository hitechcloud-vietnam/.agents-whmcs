# WHMCS Tax Exemption Documentation

## Overview

Tax exemption handling in WHMCS manages customers who are exempt from certain taxes, including VAT, GST, and sales tax, based on their status, location, or certificate.

## Configuration

### Enable Tax Exemption

Navigate to: **Configuration > General Settings > Tax Settings**

```php
// Tax Exemption Configuration
$taxExemptConfig = [
    'enabled' => true,
    'default_exempt' => false,

    // Exemption Types
    'exemption_types' => [
        'vat_exempt' => [
            'enabled' => true,
            'requires_certificate' => true,
            'auto_detect' => true,
            'eu_intra' => true
        ],
        'gst_exempt' => [
            'enabled' => true,
            'requires_certificate' => true
        ],
        'sales_tax_exempt' => [
            'enabled' => true,
            'requires_certificate' => true,
            'state_based' => true
        ],
        'non_profit' => [
            'enabled' => true,
            'requires_certificate' => true,
            'verification_required' => true
        ]
    ],

    // Certificate Settings
    'certificate_required' => true,
    'certificate_validity_years' => 1,
    'auto_expire_warning_days' => 30,

    // Validation
    'validate_exemptions' => true,
    'recheck_frequency' => 'monthly',
    'enforce_expiry' => true
];
```

### Tax Rules for Exempt Customers

```php
// Tax rules for exempt customers
$exemptTaxRules = [
    'vat_exempt' => [
        'exempt_from' => ['vat', 'eu_vat'],
        'still_applicable' => ['gst', 'us_sales_tax']
    ],
    'gst_exempt' => [
        'exempt_from' => ['gst'],
        'still_applicable' => ['vat', 'us_sales_tax']
    ],
    'sales_tax_exempt' => [
        'exempt_from' => ['state_sales_tax'],
        'still_applicable' => ['vat', 'gst', 'federal_tax']
    ]
];
```

## Client Tax Exemption

### Mark Client as Tax Exempt

```php
// Update client exemption status
$client = WHMCS\User\Client::find(12345);
$client->tax_exempt = true;
$client->tax_exempt_type = 'vat_exempt';
$client->tax_exempt_certificate = 'path/to/certificate.pdf';
$client->tax_exempt_expiry = '2025-01-15';
$client->save();
```

### Exemption Types

```php
// Available exemption types
$exemptionTypes = [
    'vat_exempt' => [
        'name' => 'VAT Exempt',
        'description' => 'Exempt from VAT charges',
        'applies_to' => ['EU VAT'],
        'common_reasons' => [
            'Business outside EU',
            'EU business with valid VAT number',
            'Government entity',
            'Non-profit organization'
        ]
    ],
    'gst_exempt' => [
        'name' => 'GST Exempt',
        'description' => 'Exempt from GST charges',
        'applies_to' => ['Australia GST', 'Canada GST/HST', 'India GST'],
        'common_reasons' => [
            'Non-resident',
            'Export supplies',
            'Government entity'
        ]
    ],
    'sales_tax_exempt' => [
        'name' => 'Sales Tax Exempt',
        'description' => 'Exempt from state sales tax',
        'applies_to' => ['US State Sales Tax'],
        'common_reasons' => [
            'Resale certificate',
            'Government entity',
            'Religious organization',
            'Educational institution'
        ]
    ],
    'non_profit' => [
        'name' => 'Non-Profit Exemption',
        'description' => 'Tax exempt non-profit organization',
        'applies_to' => ['all_taxes'],
        'common_reasons' => [
            'Registered charity',
            '501(c)(3) organization',
            'Educational institution'
        ]
    ]
];
```

## Certificate Management

### Upload Certificate

```http
POST /clients/{client_id}/tax-certificates
```

**Request Body:**

```json
{
  "type": "vat_exempt",
  "certificate_number": "VAT123456789",
  "issue_date": "2024-01-15",
  "expiry_date": "2025-01-15",
  "issuing_authority": "Tax Authority",
  "document" => "base64_encoded_pdf"
}
```

### Certificate Verification

```php
// Certificate verification settings
$verificationConfig = [
    'auto_verify' => true,
    'verification_api' => 'vi_es',
    'verify_on_upload' => true,
    'reverify_frequency' => 90,        // days
    'verification_timeout' => 30
];
```

### VAT Number Validation (EU)

```http
POST /billing/tax/validate-vat
```

**Request Body:**

```json
{
  "vat_number": "DE123456789",
  "country_code" => "DE"
}
```

**Response:**

```json
{
  "valid": true,
  "company_name": "Example GmbH",
  "company_address": "Example Street 1, 12345 Berlin",
  "country_code": "DE",
  "vat_number": "DE123456789",
  "valid_from" => "2020-01-01",
  "verified_at": "2024-01-15T10:30:00Z"
}
```

## Tax Exemption Rules

### Geographic Rules

```php
// Geographic exemption rules
$geoExemptions = [
    // EU VAT exemptions
    'eu_vat_exempt_countries' => ['GB', 'NO', 'CH'],  // Non-EU but similar

    // US State exemptions
    'us_tax_exempt_states' => [
        'OR' => ['non_profit', 'resale'],
        'AK' => ['non_profit'],
        'DE' => ['non_profit', 'government']
    ],

    // Canadian exemptions
    'canada_gst_exempt' => [
        'export' => true,
        'first_nations' => true,
        'non_profit' => true
    ]
];
```

### Customer-Based Rules

```php
// Customer type exemptions
$customerExemptions = [
    'government' => [
        'exempt_from' => ['vat', 'gst', 'sales_tax'],
        'requires_documentation' => true,
        'verify_status' => true
    ],
    'educational' => [
        'exempt_from' => ['vat', 'gst', 'sales_tax'],
        'requires_documentation' => true,
        'limited_to_education' => true
    ],
    'healthcare' => [
        'exempt_from' => ['vat', 'gst', 'sales_tax'],
        'region_specific' => true
    ],
    'non_profit' => [
        'exempt_from' => ['vat', 'gst', 'sales_tax'],
        'requires_registration' => true,
        'annual_reverification' => true
    ]
];
```

## Product/Service Tax Exemption

### Configure Tax Exempt Products

```php
// Product tax settings
$productTaxConfig = [
    'tax_exempt_available' => true,
    'default_taxable' => true,
    'exempt_product_categories' => [
        'healthcare',
        'education',
        'financial_services'
    ]
];
```

### Override Tax Settings

```php
// Per-product override
$product = WHMCS\Product\Product::find(67890);
$product->tax_override = true;
$product->tax_exempt = true;
$product->tax_override_notes = 'Healthcare service - tax exempt';
$product->save();
```

## Invoice Handling

### Exempt Invoice Calculation

When a tax-exempt customer places an order:

```php
// Invoice calculation for exempt customer
$invoiceCalc = [
    'items' => [
        ['description' => 'Premium Service', 'amount' => 100.00, 'tax' => 0],
        ['description' => 'Add-on', 'amount' => 50.00, 'tax' => 0]
    ],
    'subtotal' => 150.00,
    'tax' => [
        'vat' => ['rate' => 0.19, 'amount' => 0, 'exempt' => true, 'reason' => 'VAT Exempt'],
        'gst' => ['rate' => 0.10, 'amount' => 0, 'exempt' => true, 'reason' => 'GST Exempt']
    ],
    'total' => 150.00,
    'exemption_info' => [
        'type' => 'vat_exempt',
        'certificate_number' => 'VAT123456789',
        'valid_until' => '2025-01-15'
    ]
];
```

### Invoice Display

```html
<!-- Tax Exempt Notice on Invoice -->
<div class="tax-exempt-notice">
    <p class="exempt-status">Tax Exempt Customer</p>
    <p class="exempt-type">VAT Exemption Certificate: VAT123456789</p>
    <p class="exempt-valid">Valid Until: January 15, 2025</p>
    <p class="exempt-note">Tax has not been charged due to valid exemption status.</p>
</div>
```

## API Reference

### Update Client Exemption

```http
PATCH /clients/{client_id}/tax-exemption
```

**Request Body:**

```json
{
  "exempt": true,
  "type": "vat_exempt",
  "reason": "EU Business Customer",
  "certificate_number": "VAT123456789",
  "expiry_date": "2025-01-15",
  "documentation" => "base64_encoded_certificate"
}
```

### List Exempt Clients

```http
GET /billing/tax-exempt-clients
```

**Query Parameters:**
- `exemption_type`: Filter by type
- `expiring_soon`: Expiring within N days
- `expired`: Already expired

**Response:**

```json
{
  "clients": [
    {
      "client_id": 12345,
      "client_name": "Acme Corp",
      "exemption_type": "vat_exempt",
      "certificate_number": "VAT123456789",
      "expiry_date": "2025-01-15",
      "status" => "active"
    }
  ]
}
```

### Validate Exemption

```http
POST /billing/tax/validate-exemption
```

**Request Body:**

```json
{
  "client_id": 12345,
  "tax_type": "vat",
  "jurisdiction": "EU"
}
```

## Certificate Renewal

### Expiry Notifications

```php
// Expiry notification settings
$expiryNotifications = [
    'enabled' => true,
    'notify_days_before' => [30, 14, 7, 1],
    'notify_client' => true,
    'notify_admin' => true,
    'auto_expire' => true,
    'grace_period_days' => 0
];
```

### Renewal Process

```php
// Certificate renewal workflow
$renewalWorkflow = [
    'steps' => [
        'expiry_reminder_sent',
        'client_submits_renewal',
        'admin_reviews_certificate',
        'approval_extends_exemption'
    ],
    'auto_process' => false,
    'require_new_certificate' => true
];
```

## Reporting

### Tax Exemption Report

```http
GET /billing/reports/tax-exemptions
```

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_exempt_invoices": 150,
    "total_exempt_amount": 45000.00,
    "tax_saved" => 4500.00,
    "active_exemptions" => 50,
    "expiring_soon" => 5,
    "expired" => 2
  },
  "by_type" => [
    {"type" => "vat_exempt", "count" => 30, "amount" => 25000},
    {"type" => "gst_exempt", "count" => 10, "amount" => 10000},
    {"type" => "sales_tax_exempt", "count" => 110, "amount" => 10000}
  ]
}
```

## Customer Portal

### Tax Exemption Management

**Client Area > Account > Tax Exemption**

```
+------------------------------------------------------------------+
|  Tax Exemption Status                                             |
+------------------------------------------------------------------+
|                                                                  |
|  Current Status: VAT Exempt                                     |
|  Certificate: VAT123456789                                      |
|  Valid Until: January 15, 2025                                  |
|  Status: Active (45 days remaining)                             |
|                                                                  |
|  Documents on File:                                              |
|  - VAT Certificate (uploaded Jan 15, 2024)                       |
|                                                                  |
|  Actions:                                                       |
|  [Update Certificate] [Remove Exemption]                         |
|                                                                  |
+------------------------------------------------------------------+
```

### Upload New Certificate

**Client Area > Account > Tax Exemption > Upload**

```
+------------------------------------------+
| Upload Tax Exemption Certificate          |
+------------------------------------------+
| Exemption Type: [VAT Exempt________]    |
| Certificate Number: [VAT123456789____]   |
| Issuing Authority: [Tax Authority______]|
| Issue Date:      [2024-01-15__________]|
| Expiry Date:    [2025-01-15__________] |
| Certificate:    [Choose File__________] |
|                                          |
| [Upload Certificate]                     |
+------------------------------------------+
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Tax still charged | Exemption expired | Renew certificate |
| Wrong exemption type | Type misconfigured | Verify exemption type |
| Certificate rejected | Invalid document | Upload valid certificate |
| Audit finding | Exemption not documented | Maintain certificate records |

### Debug Commands

```bash
# Check client exemption status
whmcscli client tax-exempt --client_id=12345

# Verify certificate
whmcscli tax certificate-verify --certificate=VAT123456789

# List expiring exemptions
whmcscli tax expiring --days=30

# Force recalculate invoice
whmcscli invoice recalculate --invoice_id=INV-12345
```

## Compliance

### Record Keeping

```php
// Required records for tax compliance
$recordKeeping = [
    'keep_certificates' => true,
    'certificate_retention_years' => 7,
    'log_exemption_checks' => true,
    'audit_trail' => true,
    'annual_reverification' => false
];
```

## See Also

- [VAT Handling](./whmcs-vat-handling.md)
- [GST Processing](./whmcs-gst-processing.md)
- [Custom Taxes](./whmcs-custom-taxes.md)
