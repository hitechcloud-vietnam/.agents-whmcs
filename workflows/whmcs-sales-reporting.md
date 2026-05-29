# WHMCS Sales Reporting Workflow

## Purpose
Generate comprehensive sales reports

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Generate Order Report

Navigate to: Reports > Orders > Orders Report

Configure:
- Date range
- Status filter
- Product filter

Export to CSV.

## Step 2: Generate Revenue Report

Navigate to: Reports > Revenue > Revenue Summary

Metrics:
- Total revenue
- Recurring revenue
- One-time revenue
- Refunds

## Step 3: Analyze Orders by Product

```sql
SELECT 
    p.name as product,
    COUNT(o.id) as total_orders,
    SUM(o.amount) as total_revenue
FROM tblorders o
JOIN tblproducts p ON o.packageid = p.id
WHERE o.date >= '2024-01-01'
GROUP BY p.name
ORDER BY total_revenue DESC;
```

## Step 4: Analyze Orders by Status

```sql
SELECT 
    status,
    COUNT(*) as count,
    SUM(amount) as revenue
FROM tblorders
GROUP BY status;
```

## Step 5: Calculate Conversion Rate

```sql
SELECT 
    COUNT(*) as total_orders,
    SUM(CASE WHEN status = 'Active' THEN 1 ELSE 0 END) as accepted,
    SUM(CASE WHEN status = 'Cancelled' THEN 1 ELSE 0 END) as cancelled,
    SUM(CASE WHEN status = 'Active' THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as acceptance_rate
FROM tblorders
WHERE date >= '2024-01-01';
```

## Step 6: Analyze Average Order Value

```sql
SELECT 
    AVG(amount) as aov,
    MIN(amount) as min_order,
    MAX(amount) as max_order
FROM tblorders
WHERE status = 'Active'
AND date >= '2024-01-01';
```

## Step 7: Analyze Trends

Create trend analysis:
- Daily orders (last 30 days)
- Weekly orders (last 12 weeks)
- Monthly orders (last 12 months)

## Step 8: Generate Sales Report

Create comprehensive report:
```
SALES REPORT
Period: [dates]

OVERVIEW
Total Orders: XXX
Total Revenue: $XXX
Average Order Value: $XX
Conversion Rate: XX%

BY PRODUCT
1. Product A: $XXX (XX%)
2. Product B: $XXX (XX%)
3. Product C: $XXX (XX%)

BY STATUS
Active: XXX
Cancelled: XXX
Pending: XXX

TRENDS
- Best Day: [date] - $XXX
- Best Week: [week] - $XXX
- Best Month: [month] - $XXX
```

## Sales Reporting Checklist

- [ ] Order report generated
- [ ] Revenue report generated
- [ ] Product analysis done
- [ ] Status analysis done
- [ ] Conversion rate calculated
- [ ] AOV calculated
- [ ] Trends analyzed
- [ ] Comprehensive report created
