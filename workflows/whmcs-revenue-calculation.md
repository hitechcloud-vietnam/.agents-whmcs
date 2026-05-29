# WHMCS Revenue Calculation Workflow

## Purpose
Calculate and analyze WHMCS revenue

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Get Total Revenue

Navigate to: Reports > Revenue > Monthly Revenue

### Calculate Total Revenue (SQL)
```sql
SELECT 
    SUM(total) as total_revenue
FROM tblinvoices
WHERE status = 'Paid'
AND date >= '2024-01-01'
AND date <= '2024-12-31';
```

## Step 2: Calculate Monthly Recurring Revenue (MRR)

```sql
SELECT 
    SUM(monthly) as mrr
FROM (
    SELECT 
        p.monthly,
        COUNT(h.id) as active_services
    FROM tblhosting h
    JOIN tblproducts p ON h.packageid = p.id
    WHERE h.domainstatus = 'Active'
    GROUP BY p.monthly
) as services;
```

## Step 3: Calculate Annual Recurring Revenue (ARR)

```sql
SELECT 
    SUM(annual) as arr
FROM (
    SELECT 
        p.annual,
        COUNT(h.id) as active_services
    FROM tblhosting h
    JOIN tblproducts p ON h.packageid = p.id
    WHERE h.domainstatus = 'Active'
    GROUP BY p.annual
) as services;
```

## Step 4: Calculate Revenue by Product

```sql
SELECT 
    p.name as product_name,
    COUNT(h.id) as active_services,
    SUM(h.amount) as monthly_revenue
FROM tblhosting h
JOIN tblproducts p ON h.packageid = p.id
WHERE h.domainstatus = 'Active'
GROUP BY p.name
ORDER BY monthly_revenue DESC;
```

## Step 5: Calculate Revenue by Client

```sql
SELECT 
    c.id,
    c.firstname,
    c.lastname,
    SUM(i.total) as lifetime_value
FROM tblclients c
JOIN tblinvoices i ON c.id = i.userid
WHERE i.status = 'Paid'
GROUP BY c.id, c.firstname, c.lastname
ORDER BY lifetime_value DESC
LIMIT 20;
```

## Step 6: Calculate Average Revenue Per User (ARPU)

```sql
SELECT 
    SUM(total) / COUNT(DISTINCT userid) as arpu
FROM tblinvoices
WHERE status = 'Paid'
AND date >= DATE_SUB(NOW(), INTERVAL 12 MONTH);
```

## Step 7: Calculate Revenue Growth

```sql
SELECT 
    YEAR(date) as year,
    MONTH(date) as month,
    SUM(total) as monthly_revenue
FROM tblinvoices
WHERE status = 'Paid'
GROUP BY YEAR(date), MONTH(date)
ORDER BY year DESC, month DESC;
```

## Step 8: Generate Revenue Report

Create Excel/CSV report:
```
Total Revenue: $XXX
MRR: $XXX
ARR: $XXX
ARPU: $XXX
Revenue Growth: XX%
```

## Revenue Calculation Checklist

- [ ] Total revenue calculated
- [ ] MRR calculated
- [ ] ARR calculated
- [ ] Revenue by product analyzed
- [ ] Revenue by client analyzed
- [ ] ARPU calculated
- [ ] Growth calculated
- [ ] Report generated
