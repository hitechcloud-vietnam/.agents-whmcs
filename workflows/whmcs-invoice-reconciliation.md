# WHMCS Invoice Reconciliation Workflow

## Purpose
Reconcile WHMCS invoices with accounting records

## Prerequisites
- WHMCS installed
- Admin access
- Accounting system access

## Step 1: Export Invoice Data

Navigate to: Billing > Invoices

Export to CSV:
```
Invoice Number, Date, Client, Amount, Status, Payment Date, Payment Method
```

Or use SQL:
```sql
SELECT 
    i.id,
    i.invoicenum,
    i.date,
    c.firstname,
    c.lastname,
    i.total,
    i.status,
    i.duedate,
    i.datepaid,
    i.paymentmethod
FROM tblinvoices i
JOIN tblclients c ON i.userid = c.id
WHERE i.date >= '2024-01-01'
ORDER BY i.date DESC;
```

## Step 2: Export Payment Data

Navigate to: Billing > Transactions

Export transactions to match with invoices.

## Step 3: Reconcile Paid Invoices

Match WHMCS payments with accounting records:
- Invoice number
- Amount paid
- Payment date
- Payment method

## Step 4: Identify Discrepancies

Common issues:
- Unpaid invoices marked paid
- Partial payments
- Wrong amounts
- Missing transactions

### SQL to Find Issues
```sql
-- Invoices marked paid but no payment record
SELECT i.*
FROM tblinvoices i
LEFT JOIN tblaccounts a ON i.id = a.invoiceid
WHERE i.status = 'Paid' AND a.id IS NULL;
```

## Step 5: Correct Discrepancies

For each issue:
1. Investigate root cause
2. Update WHMCS if needed
3. Document correction
4. Update accounting records

## Step 6: Reconcile Refunds

```sql
SELECT 
    i.*,
    r.refundtype,
    r.refundamount,
    r.refunddate
FROM tblinvoices i
JOIN tblrefunds r ON i.id = r.invoiceid
WHERE i.date >= '2024-01-01';
```

## Step 7: Generate Reconciliation Report

Create report:
```
Reconciliation Date: [date]
Period: [start] to [end]

Total Invoices: XX
Total Invoice Value: $XXX

Paid Invoices: XX
Paid Value: $XXX

Outstanding Invoices: XX
Outstanding Value: $XXX

Overdue Invoices: XX
Overdue Value: $XXX

Refunds: XX
Refund Value: $XXX

Discrepancies Found: X
Discrepancies Resolved: X
```

## Step 8: Export for Accounting

Export reconciled data to:
- QuickBooks
- Xero
- FreshBooks
- Manual accounting system

## Invoice Reconciliation Checklist

- [ ] Invoice data exported
- [ ] Payment data exported
- [ ] Paid invoices reconciled
- [ ] Discrepancies identified
- [ ] Discrepancies corrected
- [ ] Refunds reconciled
- [ ] Report generated
- [ ] Data exported for accounting
