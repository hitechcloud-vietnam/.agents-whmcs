# WHMCS Tax Filing Preparation Workflow

## Purpose
Prepare WHMCS data for tax filing

## Prerequisites
- WHMCS installed
- Admin access
- Tax knowledge

## Step 1: Review Tax Settings

Navigate to: Setup > Payments > Tax Rules

Verify:
- Tax rates configured
- Tax exemptions
- Applicable regions

## Step 2: Export Sales Data

### All Invoices
```sql
SELECT 
    i.invoicenum,
    i.date,
    c.companyname,
    c.vatnumber,
    c.country,
    c.state,
    i.subtotal,
    i.tax,
    i.total,
    i.status
FROM tblinvoices i
JOIN tblclients c ON i.userid = c.id
WHERE i.date >= '2024-01-01'
AND i.date <= '2024-12-31'
ORDER BY i.date;
```

## Step 3: Calculate Tax Collected

```sql
SELECT 
    SUM(tax) as total_tax_collected
FROM tblinvoices
WHERE status = 'Paid'
AND date >= '2024-01-01'
AND date <= '2024-12-31';
```

## Step 4: Calculate Tax by Jurisdiction

```sql
SELECT 
    c.country,
    c.state,
    SUM(i.tax) as tax_collected,
    COUNT(i.id) as invoices
FROM tblinvoices i
JOIN tblclients c ON i.userid = c.id
WHERE i.status = 'Paid'
AND i.tax > 0
GROUP BY c.country, c.state;
```

## Step 5: Identify Tax-Exempt Sales

```sql
SELECT 
    c.id,
    c.companyname,
    c.vatnumber,
    SUM(i.total) as exempt_sales
FROM tblinvoices i
JOIN tblclients c ON i.userid = c.id
WHERE i.status = 'Paid'
AND c.taxexempt = 1
GROUP BY c.id, c.companyname, c.vatnumber;
```

## Step 6: Generate Tax Report

Create report:
```
TAX FILING REPORT
Year: 2024

Total Sales: $XXX
Taxable Sales: $XXX
Exempt Sales: $XXX
Total Tax Collected: $XXX

Tax by Jurisdiction:
- USA-CA: $XXX
- USA-NY: $XXX
- EU-DE: $XXX

Tax-Exempt Customers: XX
Exempt Amount: $XXX
```

## Step 7: Reconcile with Payments

Verify tax collected matches:
- Payment gateway totals
- Bank deposits
- Accounting records

## Step 8: Export for Tax Software

Export to CSV:
```
Invoice Date, Invoice #, Customer, Tax ID, Amount, Tax, Total, Payment Method
```

## Tax Filing Checklist

- [ ] Tax settings reviewed
- [ ] Sales data exported
- [ ] Tax collected calculated
- [ ] Tax by jurisdiction calculated
- [ ] Exempt sales identified
- [ ] Tax report generated
- [ ] Payments reconciled
- [ ] Data exported for filing
