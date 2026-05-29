# WHMCS Payment Debug Workflow

## Overview
This workflow guides you through debugging payment processing issues.

## Prerequisites
- Payment gateway logs
- Transaction records
- WHMCS admin access

## Step-by-Step Guide

### Step 1: Check Transaction Status
```sql
-- Find the transaction
SELECT * FROM tblaccounts WHERE transactionid = 'TXN123';

-- Check associated invoice
SELECT i.*, a.transactionid, a.amount
FROM tblinvoices i
LEFT JOIN tblaccounts a ON i.id = a.invoiceid
WHERE i.id = (SELECT invoiceid FROM tblaccounts WHERE transactionid = 'TXN123');
```

### Step 2: Enable Payment Logging
```php
// In gateway callback
logModuleCall(
    'yourgateway',
    'payment_callback',
    print_r($_POST, true),
    'Callback received'
);
```

### Step 3: Test Payment Flow
```bash
# 1. Create test order
# WHMCS Admin > Orders > Create New Order

# 2. Set invoice to Pending
# Update invoice status to Unpaid

# 3. Submit test payment
curl -X POST "http://localhost/modules/gateways/yourgateway/callback.php" \
    -d "invoice_id=INV-1001" \
    -d "transaction_id=test_txn_123" \
    -d "amount=99.99" \
    -d "status=completed"
```

### Step 4: Common Payment Issues
```php
// Issue: Invoice not found
// Fix: Verify invoice ID format and existence

// Issue: Amount mismatch
// Fix: Compare callback amount with invoice total

// Issue: Duplicate transaction
// Fix: Check tblaccounts for existing transaction

// Issue: Gateway not configured
// Fix: Verify gateway is activated in WHMCS
```

## Payment Debug Checklist

### Investigation
- [ ] Transaction found
- [ ] Invoice status checked
- [ ] Gateway logs reviewed
- [ ] Amounts compared

### Resolution
- [ ] Transaction recorded
- [ ] Invoice updated
- [ ] Payment applied
- [ ] User notified
