# WHMCS Client Audit Workflow

## Purpose
Conduct comprehensive client audit

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Get Client Count

```sql
SELECT 
    COUNT(*) as total_clients,
    SUM(CASE WHEN status = 'Active' THEN 1 ELSE 0 END) as active,
    SUM(CASE WHEN status = 'Inactive' THEN 1 ELSE 0 END) as inactive
FROM tblclients;
```

## Step 2: Analyze Client Acquisition

```sql
SELECT 
    DATE_FORMAT(createdat, '%Y-%m') as month,
    COUNT(*) as new_clients
FROM tblclients
GROUP BY DATE_FORMAT(createdat, '%Y-%m')
ORDER BY month DESC
LIMIT 12;
```

## Step 3: Identify Inactive Clients

```sql
SELECT 
    c.id,
    c.email,
    c.createdat,
    MAX(i.date) as last_invoice_date
FROM tblclients c
LEFT JOIN tblinvoices i ON c.id = i.userid
GROUP BY c.id, c.email, c.createdat
HAVING MAX(i.date) < DATE_SUB(NOW(), INTERVAL 12 MONTH)
OR MAX(i.date) IS NULL;
```

## Step 4: Analyze Client Value

```sql
SELECT 
    c.id,
    c.email,
    SUM(i.total) as lifetime_value,
    COUNT(i.id) as total_invoices
FROM tblclients c
JOIN tblinvoices i ON c.id = i.userid
WHERE i.status = 'Paid'
GROUP BY c.id, c.email
ORDER BY lifetime_value DESC
LIMIT 50;
```

## Step 5: Client Geographic Distribution

```sql
SELECT 
    country,
    COUNT(*) as clients
FROM tblclients
GROUP BY country
ORDER BY clients DESC;
```

## Step 6: Identify Duplicate Accounts

```sql
SELECT 
    email,
    COUNT(*) as count
FROM tblclients
GROUP BY email
HAVING COUNT(*) > 1;
```

## Step 7: Review Client Notes

Navigate to: Clients > [Client] > Notes

Review:
- Important notes
- Account history
- Special instructions

## Step 8: Verify Client Data Quality

Check for:
- Incomplete profiles
- Invalid emails
- Missing phone numbers
- Outdated information

## Step 9: Generate Client Audit Report

```
CLIENT AUDIT REPORT

TOTAL CLIENTS: XXX
- Active: XXX
- Inactive: XXX

NEW CLIENTS (YTD): XXX
AVERAGE CLIENT VALUE: $XXX

TOP 10 CLIENTS BY VALUE:
1. Client A: $XXX
2. Client B: $XXX
...

INACTIVE CLIENTS (12+ months): XXX
DUPLICATE ACCOUNTS: XX

DATA QUALITY ISSUES:
- Invalid emails: XX
- Missing phone: XX
- Incomplete profiles: XX
```

## Client Audit Checklist

- [ ] Client count obtained
- [ ] Acquisition analyzed
- [ ] Inactive clients identified
- [ ] Client value analyzed
- [ ] Geographic distribution mapped
- [ ] Duplicates identified
- [ ] Notes reviewed
- [ ] Data quality checked
- [ ] Audit report generated
