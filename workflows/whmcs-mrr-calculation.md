# WHMCS MRR (Monthly Recurring Revenue) Calculation Workflow

## Purpose
Calculate and track monthly recurring revenue

## Prerequisites
- WHMCS installed
- Admin access
- SQL/database access

## Step 1: Calculate Current MRR

```sql
SELECT 
    SUM(monthly_amount) as total_mrr
FROM (
    SELECT 
        h.id,
        CASE h.billingcycle
            WHEN 'Monthly' THEN h.amount
            WHEN 'Quarterly' THEN h.amount / 3
            WHEN 'Semi-Annual' THEN h.amount / 6
            WHEN 'Annual' THEN h.amount / 12
            WHEN 'Biennial' THEN h.amount / 24
            WHEN 'Triennial' THEN h.amount / 36
            ELSE h.amount
        END as monthly_amount
    FROM tblhosting h
    WHERE h.domainstatus = 'Active'
) as services;
```

## Step 2: Calculate MRR by Product

```sql
SELECT 
    p.name as product,
    COUNT(h.id) as active_services,
    SUM(
        CASE h.billingcycle
            WHEN 'Monthly' THEN h.amount
            WHEN 'Quarterly' THEN h.amount / 3
            WHEN 'Annual' THEN h.amount / 12
            ELSE h.amount
        END
    ) as mrr
FROM tblhosting h
JOIN tblproducts p ON h.packageid = p.id
WHERE h.domainstatus = 'Active'
GROUP BY p.name
ORDER BY mrr DESC;
```

## Step 3: Calculate MRR by Client

```sql
SELECT 
    c.id,
    c.email,
    SUM(
        CASE h.billingcycle
            WHEN 'Monthly' THEN h.amount
            WHEN 'Annual' THEN h.amount / 12
            ELSE h.amount
        END
    ) as client_mrr
FROM tblhosting h
JOIN tblclients c ON h.userid = c.id
WHERE h.domainstatus = 'Active'
GROUP BY c.id, c.email
ORDER BY client_mrr DESC
LIMIT 50;
```

## Step 4: Calculate MRR Movements

### New MRR
```sql
SELECT 
    SUM(monthly_amount) as new_mrr
FROM (
    SELECT 
        CASE h.billingcycle
            WHEN 'Monthly' THEN h.amount
            WHEN 'Annual' THEN h.amount / 12
            ELSE h.amount
        END as monthly_amount
    FROM tblhosting h
    WHERE h.domainstatus = 'Active'
    AND h.regdate >= DATE_SUB(NOW(), INTERVAL 30 DAY)
) as new_services;
```

### Expansion MRR (Upgrades)
```sql
SELECT 
    SUM(amount_difference) as expansion_mrr
FROM (
    SELECT 
        (new.amount - old.amount) as amount_difference
    FROM tblhosting new
    JOIN tblhosting old ON new.id = old.id
    WHERE new.domainstatus = 'Active'
    AND new.regdate >= DATE_SUB(NOW(), INTERVAL 30 DAY)
    AND new.amount > old.amount
) as upgrades;
```

### Churned MRR
```sql
SELECT 
    SUM(monthly_amount) as churned_mrr
FROM (
    SELECT 
        CASE h.billingcycle
            WHEN 'Monthly' THEN h.amount
            WHEN 'Annual' THEN h.amount / 12
            ELSE h.amount
        END as monthly_amount
    FROM tblhosting h
    WHERE h.domainstatus IN ('Terminated', 'Cancelled')
    AND h.termination_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
) as churned;
```

## Step 5: Calculate Net New MRR

```
Net New MRR = New MRR + Expansion MRR - Churned MRR - Contraction MRR
```

## Step 6: Track MRR Over Time

```sql
SELECT 
    DATE_FORMAT(regdate, '%Y-%m') as month,
    COUNT(*) as services_added,
    SUM(
        CASE billingcycle
            WHEN 'Monthly' THEN amount
            WHEN 'Annual' THEN amount / 12
            ELSE amount
        END
    ) as mrr_added
FROM tblhosting
WHERE domainstatus = 'Active'
GROUP BY DATE_FORMAT(regdate, '%Y-%m')
ORDER BY month DESC
LIMIT 12;
```

## Step 7: Generate MRR Report

```
MRR REPORT
Date: [current date]

CURRENT MRR: $XXX

MRR MOVEMENTS (30 DAYS):
New MRR: +$XXX
Expansion MRR: +$XXX
Contraction MRR: -$XXX
Churned MRR: -$XXX
Net New MRR: $XXX

MRR BY PRODUCT:
1. Product A: $XXX (XX%)
2. Product B: $XXX (XX%)

TOP MRR CLIENTS:
1. Client A: $XXX
2. Client B: $XXX
```

## MRR Calculation Checklist

- [ ] Current MRR calculated
- [ ] MRR by product calculated
- [ ] MRR by client calculated
- [ ] New MRR calculated
- [ ] Expansion MRR calculated
- [ ] Churned MRR calculated
- [ ] Net new MRR calculated
- [ ] MRR tracked over time
- [ ] MRR report generated
