# WHMCS Refund Policy Documentation

## Overview

Refund policies define how WHMCS handles customer refund requests, including processing timeframes, refund methods, and automation rules.

## Configuration

### Enable Refunds

Navigate to: **Configuration > General Settings > Refund Settings**

```php
// Refund Configuration
$refundConfig = [
    'enabled' => true,
    'allow_refunds' => true,

    // Timeframe
    'refund_window_days' => 30,
    'refund_window_override' => true,

    // Methods
    'refund_methods' => [
        'original_payment',
        'credit_balance',
        'store_credit',
        'bank_transfer'
    ],
    'default_refund_method' => 'original_payment',

    // Automation
    'auto_approve_small_refunds' => true,
    'small_refund_threshold' => 10.00,
    'require_approval_above' => 100.00
];
```

## Refund Rules

### Standard Refund Rules

```php
// Standard refund policy
$standardRefund = [
    'name' => 'Standard Refund',
    'timeframe_days' => 30,
    'approval_required' => false,
    'processing_time_days' => 5,
    'refund_methods' => ['original_payment', 'credit_balance'],

    // What can be refunded
    'refundable_items' => [
        'products' => true,
        'setup_fees' => true,
        'addons' => true,
        'domains' => false,
        'ssl' => false,
        'custom_items' => true
    ],

    // Non-refundable
    'non_refundable' => [
        'late_fees' => true,
        'domain_renewals' => false,
        'setup_fee_if_used' => true,
        'custom_development' => false
    ]
];
```

### Product-Specific Refunds

```php
// Product-specific refund policies
$productRefundPolicies = [
    'hosting' => [
        'refund_window_days' => 30,
        'approval_required' => false,
        'setup_fee_refundable' => true,
        'setup_fee_refund_window_days' => 7,
        'prorate_on_refund' => true
    ],
    'domain' => [
        'refund_window_days' => 5,
        'approval_required' => true,
        'registry_refund_policy' => true,
        'restocking_fee' => 0
    ],
    'ssl' => [
        'refund_window_days' => 7,
        'approval_required' => true,
        'certificate_issued' => [
            'refundable' => false,
            'reason_required' => 'technical_issues'
        ]
    ],
    'software' => [
        'refund_window_days' => 14,
        'approval_required' => true,
        'license_revoked' => true
    ]
];
```

## Refund Processing

### Request Refund

```http
POST /billing/refunds
```

**Request Body:**

```json
{
  "invoice_id": "INV-12345",
  "items": [
    {
      "item_id": "ITEM-001",
      "amount": 99.99
    }
  ],
  "reason": "service_issue",
  "reason_details": "Service was unavailable for 3 days",
  "customer_notes": "Please process my refund"
}
```

### Process Refund (Admin)

```http
POST /billing/refunds/{refund_id}/process
```

**Request Body:**

```json
{
  "action": "approve",
  "amount": 99.99,
  "method": "original_payment",
  "notes": "Approved due to service outage",
  "notify_customer": true
}
```

### Refund Response

```json
{
  "success": true,
  "refund_id": "REF-12345",
  "status": "approved",
  "amount": 99.99,
  "method": "original_payment",
  "processing_date": "2024-01-15",
  "estimated_completion": "2024-01-20",
  "credit_balance_updated": false
}
```

## Refund Types

### Full Refund

```php
// Full refund of order
$fullRefund = [
    'type' => 'full',
    'invoice_id' => 'INV-12345',
    'amount' => 299.99,
    'items' => [
        ['product_id' => 100, 'amount' => 199.99],
        ['addon_id' => 50, 'amount' => 50.00],
        ['setup_fee_id' => 25, 'amount' => 50.00]
    ],
    'include_tax' => true,
    'tax_amount' => 24.99
];
```

### Partial Refund

```php
// Partial refund
$partialRefund = [
    'type' => 'partial',
    'invoice_id' => 'INV-12345',
    'reason' => 'service_credit',
    'items' => [
        ['product_id' => 100, 'amount' => 50.00]
    ],
    'original_amount' => 299.99,
    'refund_amount' => 50.00,
    'remaining_balance' => 249.99
];
```

### Pro-Rated Refund

```php
// Pro-rated refund
$proRatedRefund = [
    'type' => 'pro_rated',
    'service_id' => 67890,
    'reason' => 'cancellation',
    'calculation' => [
        'monthly_price' => 99.00,
        'days_in_month' => 30,
        'days_used' => 10,
        'days_remaining' => 20,
        'daily_rate' => 3.30,
        'credit_amount' => 66.00
    ],
    'refund_amount' => 66.00
];
```

## Refund Workflow

### Automatic Refund Workflow

```
1. Customer requests refund
2. System validates:
   - Within refund window?
   - Products refundable?
   - Amount under threshold?
3. Auto-approve (if eligible)
   - Create refund record
   - Process payment refund
   - Update invoice status
   - Send confirmation
4. Manual approval (if required)
   - Notify admin
   - Admin reviews request
   - Admin approves/rejects
   - Process if approved
```

### Manual Approval Workflow

```php
// Manual approval workflow
$manualApproval = [
    'required_for' => [
        'amount_above' => 100.00,
        'first_refund' => false,
        'specific_products' => ['custom_development']
    ],
    'approval_roles' => ['billing_manager', 'admin'],
    'approval_timeframe_days' => 3,
    'auto_deny_after_days' => 14,
    'auto_cancel_if_no_response' => true
];
```

## Refund Reasons

### Common Reasons

```php
// Predefined refund reasons
$refundReasons = [
    'service_issue' => [
        'name' => 'Service Issue',
        'requires_documentation' => true,
        'auto_approve' => true,
        'approval_threshold' => 100.00
    ],
    'customer_request' => [
        'name' => 'Customer Request',
        'requires_documentation' => false,
        'auto_approve' => false,
        'within_window_required' => true
    ],
    'billing_error' => [
        'name' => 'Billing Error',
        'requires_documentation' => true,
        'auto_approve' => true
    ],
    'technical_issue' => [
        'name' => 'Technical Issue',
        'requires_documentation' => true,
        'ticket_reference_required' => true,
        'auto_approve' => false
    ],
    'product_not_as_described' => [
        'name' => 'Product Not As Described',
        'requires_documentation' => true,
        'auto_approve' => false
    ]
];
```

## Customer Portal

### Request Refund

**Client Area > Billing > Request Refund**

```
+------------------------------------------------------------------+
|  Request Refund                                                   |
+------------------------------------------------------------------+
|                                                                  |
|  Invoice: INV-12345                                             |
|  Amount: $299.99                                                |
|  Date: January 1, 2024                                          |
|  Refund Window: 30 days (expires January 31, 2024)              |
|                                                                  |
|  Items to Refund:                                               |
|  +--------------------------------------------------------------+|
|  | Item                      | Amount      | Refund             ||
|  |---------------------------|-------------|-------------------||
|  | [x] Premium Hosting      | $199.99    | [Full $199.99__] ||
|  | [x] SSL Add-on           | $50.00     | [Full $50.00___] ||
|  | [ ] Setup Fee            | $50.00     | [Not selected]   ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  Total Refund: $249.99                                         |
|                                                                  |
|  Reason: [Select reason_____________________]                    |
|  Details: [_______________________________________________]       |
|                                                                  |
|  [Submit Refund Request]                                        |
+------------------------------------------------------------------+
```

## API Reference

### List Refund Requests

```http
GET /billing/refunds
```

**Query Parameters:**
- `status`: pending, approved, rejected, processed
- `invoice_id`: Filter by invoice
- `client_id`: Filter by client

**Response:**

```json
{
  "refunds": [
    {
      "id": "REF-12345",
      "invoice_id": "INV-12345",
      "client_id": 12345,
      "status": "pending",
      "amount": 99.99,
      "reason": "service_issue",
      "requested_at": "2024-01-15T10:30:00Z",
      "requested_by": "customer"
    }
  ]
}
```

### Get Refund Details

```http
GET /billing/refunds/{refund_id}
```

### Approve Refund

```http
POST /billing/refunds/{refund_id}/approve
```

### Reject Refund

```http
POST /billing/refunds/{refund_id}/reject
```

**Request Body:**

```json
{
  "reason": "Outside refund window",
  "notes": "Refund request denied - outside 30-day window",
  "notify_customer": true
}
```

## Reporting

### Refund Report

```http
GET /billing/reports/refunds
```

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_refunds" => 50,
    "total_refunded" => 5000.00,
    "pending_refunds" => 5,
    "pending_amount" => 500.00,
    "approved_refunds" => 45,
    "approved_amount" => 4500.00,
    "rejected_refunds" => 5,
    "rejected_amount" => 500.00
  },
  "by_reason" => [
    {"reason" => "service_issue", "count" => 20, "amount" => 2000.00},
    {"reason" => "customer_request", "count" => 15, "amount" => 1500.00},
    {"reason" => "billing_error", "count" => 10, "amount" => 1000.00}
  ],
  "by_product" => [
    {"product" => "hosting", "count" => 30, "amount" => 3000.00},
    {"product" => "ssl", "count" => 10, "amount" => 1000.00}
  ]
}
```

## Non-Refundable Items

### Default Non-Refundable

```php
// Items that cannot be refunded
$nonRefundable = [
    // Fees
    'late_payment_fees',
    'setup_fees_after_use',
    'domain_renewal_fees',
    'transfer_fees',

    // Time-Sensitive
    'used_service_time',
    'consumed_resources',
    'bandwidth_used',

    // Products
    'issued_ssl_certificates',
    'completed_custom_work',
    'downloaded_digital_goods',
    'registered_domains',

    // Regulatory
    'government_taxes',
    'regulatory_fees'
];
```

## Credit Balance

### Refund to Credit Balance

```php
// Refund to store credit
$creditRefund = [
    'method' => 'credit_balance',
    'add_to_balance' => true,
    'credit_expires_days' => 365,
    'notify_customer' => true,
    'auto_apply_to_invoices' => false
];
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Refund not processing | Payment gateway issue | Check gateway logs |
| Amount mismatch | Tax not included | Check tax calculation |
| Request pending | Approval required | Check approval workflow |
| Customer disputes | Unclear policy | Update refund terms |

### Debug Commands

```bash
# List pending refunds
whmcscli refund list --status=pending

# Process refund
whmcscli refund process --refund_id=REF-12345

# View refund details
whmcscli refund view --refund_id=REF-12345
```

## Best Practices

1. **Clear policy** - Display refund terms clearly
2. **Quick processing** - Aim for same-day refunds
3. **Communicate status** - Keep customers informed
4. **Fair evaluation** - Consider customer history
5. **Document decisions** - Maintain audit trail
6. **Monitor patterns** - Track refund reasons

## See Also

- [Credit Policy](./whmcs-credit-policy.md)
- [Early Termination](./whmcs-early-termination.md)
- [Discount Limits](./whmcs-discount-limits.md)
