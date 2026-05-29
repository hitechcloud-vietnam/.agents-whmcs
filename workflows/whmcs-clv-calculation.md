# WHMCS CLV (Customer Lifetime Value) Calculation Workflow

## Purpose
Calculate customer lifetime value

## Prerequisites
- WHMCS installed
- Admin access
- SQL/database access

## Step 1: Calculate Total Customer Value

```sql
SELECT 
    c.id,
    c.email,
    c.createdat,
    COALESCE(SUM(i.total), 0) as lifetime_value,
    COUNT(i.id) as total_invoices,
    COALESCE(MAX(i.datepaid), c.createdat) as last_purchase,
    DATEDIFF(NOW(), c.createdat) as customer_age_days
FROM tblclients c
LEFT JOIN tblinvoices i ON c.id = i.userid AND i.status = 'Paid'
GROUP BY c.id, c.email, c.createdat
ORDER BY lifetime_value DESC;
```

## Step 2: Calculate Average LTV

```sql
SELECT 
    COUNT(*) as total_customers,
    AVG(lifetime_value) as avg_ltv,
    SUM(lifetime_value) as total_revenue,
    AVG(CASE WHEN lifetime_value > 0 THEN lifetime_value ELSE NULL END) as avg_ltv_excluding_zero
FROM (
    SELECT 
        c.id,
        COALESCE(SUM(i.total), 0) as lifetime_value
    FROM tblclients c
    LEFT JOIN tblinvoices i ON c.id = i.userid AND i.status = 'Paid'
    GROUP BY c.id
) as client_values;
```

## Step 3: Calculate LTV by Cohort

```sql
SELECT 
    DATE_FORMAT(createdat, '%Y-%m') as cohort,
    COUNT(*) as cohort_size,
    AVG(lifetime_value) as avg_ltv,
    SUM(lifetime_value) as cohort_revenue
FROM (
    SELECT 
        c.id,
        c.createdat,
        COALESCE(SUM(i.total), 0) as lifetime_value
    FROM tblclients c
    LEFT JOIN tblinvoices i ON c.id = i.userid AND i.status = 'Paid'
    GROUP BY c.id, c.createdat
) as cohorts
GROUP BY cohort
ORDER BY cohort DESC;
```

## Step 4: Calculate LTV by Segment

```sql
SELECT 
    CASE
        WHEN lifetime_value >= 5000 THEN 'High Value'
        WHEN lifetime_value >= 1000 THEN 'Medium Value'
        WHEN lifetime_value >= 100 THEN 'Low Value'
        ELSE 'At Risk'
    END as segment,
    COUNT(*) as customers,
    AVG(lifetime_value) as avg_ltv
FROM (
    SELECT 
        c.id,
        COALESCE(SUM(i.total), 0) as lifetime_value
    FROM tblclients c
    LEFT JOIN tblinvoices i ON c.id = i.userid AND i.status = 'Paid'
    GROUP BY c.id
) as segments
GROUP BY segment;
```

## Step 5: Calculate Predicted LTV

```sql
-- Based on average monthly value and expected lifetime
SELECT 
    c.id,
    c.email,
    COALESCE(SUM(i.total), 0) as historical_ltv,
    COALESCE(SUM(i.total) / NULLIF(DATEDIFF(NOW(), MIN(i.date)), 0), 0) * 30 as avg_monthly_value,
    COALESCE(SUM(i.total) / NULLIF(DATEDIFF(NOW(), MIN(i.date)), 0), 0) * 30 * 24 as predicted_ltv_2yr,
    COALESCE(SUM(i.total) / NULLIF(DATEDIFF(NOW(), MIN(i.date)), 0), 0) * 30 * 36 as predicted_ltv_3yr
FROM tblclients c
LEFT JOIN tblinvoices i ON c.id = i.userid AND i.status = 'Paid'
GROUP BY c.id, c.email;
```

## Step 6: Calculate LTV by Acquisition Source

```sql
SELECT 
    COALESCE(c.affiliateid, 0) as affiliate,
    COUNT(DISTINCT c.id) as clients,
    AVG(COALESCE(SUM(i.total), 0)) as avg_ltv
FROM tblclients c
LEFT JOIN tblinvoices i ON c.id = i.userid AND i.status = 'Paid'
GROUP BY COALESCE(c.affiliateid, 0);
```

## Step 7: Generate LTV Report

```
CUSTOMER LIFETIME VALUE REPORT
Date: [current date]

OVERALL METRICS
Total Customers: XXX
Total Revenue: $XXX
Average LTV: $XXX
Median LTV: $XXX

LTV DISTRIBUTION:
- High Value ($5,000+): XX clients
- Medium Value ($1,000-$4,999): XX clients
- Low Value ($100-$999): XX clients
- At Risk ($0-$99): XX clients

TOP 10 CUSTOMERS BY LTV:
1. Client A: $XXX
2. Client B: $XXX
...

COHORT LTV:
- 2024 Cohorts: $XXX avg
- 2023 Cohorts: $XXX avg
- 2022 Cohorts: $XXX avg
```

## LTV Calculation Checklist

- [ ] Total customer value calculated
- [ ] Average LTV calculated
- [ ] LTV by cohort analyzed
- [ ] LTV by segment analyzed
- [ ] Predicted LTV calculated
- [ ] LTV by acquisition source analyzed
- [ ] LTV report generated
