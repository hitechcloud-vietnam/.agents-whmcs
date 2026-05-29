# WHMCS Payment Reconciliation Workflow

## Purpose
Reconcile payments with payment gateway records

## Prerequisites
- WHMCS installed
- Admin access
- Payment gateway access

## Step 1: Export WHMCS Transactions

Navigate to: Billing > Transactions

Export all transactions:
```
Transaction ID, Date, Amount, Gateway, Invoice, Client
```

## Step 2: Export Gateway Transactions

### PayPal
1. Log into PayPal Business
2. Reports > Transaction Search
3. Export CSV

### Stripe
1. Log into Stripe Dashboard
2. Payments > Export
3. Download CSV

### Bank
1. Log into bank account
2. Download statement
3. Export to CSV

## Step 3: Match Transactions

Create matching report:
```
WHMCS ID | Gateway ID | Amount | Match | Status
12345 | PY-123456 | $99.00 | Yes | OK
12346 | - | $50.00 | No | Missing
- | STR-789 | $75.00 | No | Orphan
```

## Step 4: Identify Discrepancies

### Missing in WHMCS (Orphan Payments)
```sql
-- Gateway paid but not recorded
-- (compare external data)
```

### Missing in Gateway (Unrecorded)
```sql
-- WHMCS marked paid but not in gateway
SELECT i.*
FROM tblinvoices i
WHERE i.status = 'Paid'
AND i.paymentmethod = 'stripe'
AND i.datepaid >= '2024-01-01'
AND i.id NOT IN (SELECT invoiceid FROM tblgatewaytransactions WHERE gateway = 'stripe');
```

## Step 5: Investigate Issues

For each discrepancy:
1. Check gateway dashboard
2. Check WHMCS transaction log
3. Check client account
4. Document findings

## Step 6: Apply Corrections

### Record Missing Payments
Navigate to: Billing > Transactions > Add Transaction

Enter:
- Invoice number
- Amount
- Payment date
- Gateway reference

### Refund Excess Payments
Navigate to: Billing > Invoices > [Invoice] > Add Credit

Apply correction.

## Step 7: Reconcile by Gateway

Create per-gateway summary:
```
PayPal
- Total Received: $XXX
- Transactions: XX
- Fees: $XX
- Net: $XX

Stripe
- Total Received: $XXX
- Transactions: XX
- Fees: $XX
- Net: $XX
```

## Step 8: Generate Reconciliation Report

```
Payment Reconciliation Report
Period: [dates]

Total Payments Received: $XXX
Total Transactions: XX
Total Fees: $XX
Net Revenue: $XXX

Discrepancies: X
- Resolved: X
- Pending: X

Gateway Breakdown:
- PayPal: $XXX
- Stripe: $XXX
- Bank: $XXX
```

## Payment Reconciliation Checklist

- [ ] WHMCS transactions exported
- [ ] Gateway transactions exported
- [ ] Transactions matched
- [ ] Discrepancies identified
- [ ] Issues investigated
- [ ] Corrections applied
- [ ] Per-gateway reconciliation done
- [ ] Report generated
