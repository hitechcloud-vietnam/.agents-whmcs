# WHMCS Callback Debugging Workflow

## Overview
This workflow guides you through debugging payment gateway callback issues.

## Prerequisites
- Gateway logs
- Webhook inspection
- WHMCS gateway configuration

## Step-by-Step Guide

### Step 1: Check Gateway Logs
```bash
# View WHMCS gateway logs
tail -100 /var/www/whmcs/admin/logs/gateway.log

# Check module-specific logs
tail -100 /var/www/whmcs/storage/logs/gateway_yourmodule.log
```

### Step 2: Enable Debug Logging
```php
// In your gateway callback
logModuleCall(
    'yourmodule',
    'callback_received',
    print_r($_POST, true),
    print_r($_GET, true)
);
```

### Step 3: Test Callback Locally
```bash
# Simulate callback
curl -X POST "http://localhost/modules/gateways/yourmodule/callback.php" \
    -d "transaction_id=txn_test_123" \
    -d "invoice_id=INV-1001" \
    -d "amount=99.99" \
    -d "status=completed"
```

### Step 4: Check WHMCS Transaction
```sql
SELECT * FROM tblaccounts WHERE transactionid = 'txn_test_123';
SELECT * FROM tblinvoices WHERE id = 1001;
```

### Step 5: Common Issues
```php
// Issue: Callback not reaching WHMCS
// Fix: Check firewall, URL configuration

// Issue: Double processing
// Fix: Check for duplicate transaction_id

// Issue: Amount mismatch
// Fix: Compare callback amount with invoice total
```

## Callback Debugging Checklist

### Investigation
- [ ] Callback logs reviewed
- [ ] Transaction verified
- [ ] Amounts compared
- [ ] Timing checked

### Resolution
- [ ] URL corrected
- [ ] Signature verified
- [ ] Idempotency implemented
- [ ] Transaction recorded
