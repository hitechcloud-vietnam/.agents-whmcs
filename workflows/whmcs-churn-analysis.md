# WHMCS Churn Analysis Workflow

## Purpose
Analyze customer and revenue churn

## Prerequisites
- WHMCS installed
- Admin access
- Historical data

## Step 1: Calculate Customer Churn Rate

```sql
-- Monthly customer churn
SELECT 
    DATE_FORMAT(createdat, '%Y-%m') as month,
    COUNT(*) as new_clients,
    SUM(CASE WHEN status = 'Inactive' THEN 1 ELSE 0 END) as churned
FROM tblclients
GROUP BY DATE_FORMAT(createdat, '%Y-%m')
ORDER BY month DESC
LIMIT 12;
```

## Step 2: Calculate Revenue Churn

```sql
-- Monthly revenue churn
SELECT 
    DATE_FORMAT(i.date, '%Y-%m') as month,
    SUM(i.total) as revenue,
    SUM(CASE WHEN i.status = 'Cancelled' THEN i.total ELSE 0 END) as churned_revenue
FROM tblinvoices i
GROUP BY DATE_FORMAT(i.date, '%Y-%m')
ORDER BY month DESC
LIMIT 12;
```

## Step 3: Analyze Service Churn

```sql
SELECT 
    DATE_FORMAT(regdate, '%Y-%m') as month,
    COUNT(*) as new_services,
    SUM(CASE WHEN domainstatus = 'Terminated' THEN 1 ELSE 0 END) as churned,
    SUM(CASE WHEN domainstatus = 'Cancelled' THEN 1 ELSE 0 END) as cancelled
FROM tblhosting
GROUP BY DATE_FORMAT(regdate, '%Y-%m')
ORDER BY month DESC
LIMIT 12;
```

## Step 4: Identify Churned Clients

```sql
SELECT 
    c.id,
    c.email,
    c.createdat,
    MAX(i.date) as last_activity,
    SUM(i.total) as lifetime_value
FROM tblclients c
LEFT JOIN tblinvoices i ON c.id = i.userid
WHERE c.status = 'Inactive'
GROUP BY c.id, c.email, c.createdat
ORDER BY lifetime_value DESC;
```

## Step 5: Analyze Churn Reasons

Navigate to: Clients > [Client] > Notes

Categorize churn:
- Price
- Service quality
- Competitor
- No longer needed
- Support issues

## Step 6: Calculate Churn Metrics

```sql
-- Gross Churn Rate
SELECT 
    (COUNT(DISTINCT CASE WHEN status = 'Inactive' THEN id END) * 100.0 / 
     COUNT(DISTINCT id)) as gross_churn_rate
FROM tblclients
WHERE createdat >= DATE_SUB(NOW(), INTERVAL 12 MONTH);

-- Net Revenue Churn
SELECT 
    SUM(CASE WHEN status = 'Cancelled' THEN total ELSE 0 END) as churned_revenue,
    SUM(CASE WHEN status = 'Paid' THEN total ELSE 0 END) as total_revenue,
    (SUM(CASE WHEN status = 'Cancelled' THEN total ELSE 0 END) * 100.0 / 
     SUM(CASE WHEN status = 'Paid' THEN total ELSE 0 END)) as net_churn_rate
FROM tblinvoices
WHERE date >= DATE_SUB(NOW(), INTERVAL 12 MONTH);
```

## Step 7: Cohort Analysis

```sql
SELECT 
    DATE_FORMAT(createdat, '%Y-%m') as cohort,
    COUNT(*) as cohort_size,
    SUM(CASE WHEN DATEDIFF(NOW(), createdat) >= 30 THEN 1 ELSE 0 END) as retained_30d,
    SUM(CASE WHEN DATEDIFF(NOW(), createdat) >= 90 THEN 1 ELSE 0 END) as retained_90d,
    SUM(CASE WHEN DATEDIFF(NOW(), createdat) >= 365 THEN 1 ELSE 0 END) as retained_1y
FROM tblclients
GROUP BY DATE_FORMAT(createdat, '%Y-%m')
ORDER BY cohort DESC
LIMIT 12;
```

## Step 8: Generate Churn Report

```
CHURN ANALYSIS REPORT
Period: [12 months]

OVERALL METRICS
Gross Churn Rate: XX%
Net Churn Rate: XX%
Monthly Churned Clients: XX
Monthly Churned Revenue: $XXX

CHURN BY PRODUCT:
- Product A: XX%
- Product B: XX%

COHORT RETENTION:
- 30-day: XX%
- 90-day: XX%
- 1-year: XX%

TOP CHURN REASONS:
1. Price: XX%
2. Service Quality: XX%
3. Competitor: XX%

AT-RISK CLIENTS: XX
```

## Churn Analysis Checklist

- [ ] Customer churn calculated
- [ ] Revenue churn calculated
- [ ] Service churn analyzed
- [ ] Churned clients identified
- [ ] Churn reasons analyzed
- [ ] Churn metrics calculated
- [ ] Cohort analysis done
- [ ] Churn report generated
