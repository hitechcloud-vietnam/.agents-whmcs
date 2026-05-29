# WHMCS CAC (Customer Acquisition Cost) Calculation Workflow

## Purpose
Calculate customer acquisition cost

## Prerequisites
- WHMCS installed
- Admin access
- Marketing expense data

## Step 1: Gather Marketing Expenses

Document marketing costs for period:
- Google Ads: $XXX
- Facebook Ads: $XXX
- SEO/SEM: $XXX
- Content Marketing: $XXX
- Email Marketing: $XXX
- Affiliate Commissions: $XXX
- Other: $XXX

## Step 2: Count New Customers

```sql
SELECT 
    COUNT(*) as new_customers
FROM tblclients
WHERE createdat >= DATE_SUB(NOW(), INTERVAL 30 DAY);
```

## Step 3: Calculate Simple CAC

```
CAC = Total Marketing Expenses / Number of New Customers
```

Example:
```
CAC = $5,000 / 50 customers = $100 per customer
```

## Step 4: Calculate CAC by Channel

### Affiliate CAC
```sql
SELECT 
    COUNT(DISTINCT c.id) as affiliate_customers
FROM tblclients c
JOIN tblorders o ON c.id = o.userid
WHERE c.createdat >= DATE_SUB(NOW(), INTERVAL 30 DAY)
AND c.affiliateid > 0;
```

### Organic CAC
```sql
SELECT 
    COUNT(*) as organic_customers
FROM tblclients
WHERE createdat >= DATE_SUB(NOW(), INTERVAL 30 DAY)
AND affiliateid = 0
AND leadsource = 'Organic';
```

## Step 5: Calculate Blended CAC

```sql
SELECT 
    COUNT(*) as total_new,
    SUM(CASE WHEN affiliateid > 0 THEN 1 ELSE 0 END) as affiliate_new,
    SUM(CASE WHEN affiliateid = 0 THEN 1 ELSE 0 END) as organic_new
FROM tblclients
WHERE createdat >= DATE_SUB(NOW(), INTERVAL 30 DAY);
```

## Step 6: Calculate CAC by Cohort

```sql
SELECT 
    DATE_FORMAT(createdat, '%Y-%m') as cohort,
    COUNT(*) as new_customers,
    SUM(
        CASE WHEN referralid > 0 THEN 1 ELSE 0 END
    ) as referred_customers
FROM tblclients
GROUP BY DATE_FORMAT(createdat, '%Y-%m')
ORDER BY cohort DESC
LIMIT 12;
```

## Step 7: Calculate CAC Payback Period

```sql
-- Calculate average revenue per customer
SELECT 
    AVG(monthly_revenue) as avg_monthly_revenue
FROM (
    SELECT 
        c.id,
        SUM(
            CASE h.billingcycle
                WHEN 'Monthly' THEN h.amount
                ELSE h.amount / 12
            END
        ) as monthly_revenue
    FROM tblclients c
    JOIN tblhosting h ON c.id = h.userid
    WHERE h.domainstatus = 'Active'
    GROUP BY c.id
) as client_revenue;

-- Payback = CAC / Monthly Revenue
-- Example: $100 CAC / $25 MRR = 4 months payback
```

## Step 8: Calculate LTV:CAC Ratio

```
LTV:CAC Ratio = Customer Lifetime Value / Customer Acquisition Cost
```

Example:
```
LTV:CAC = $1,200 / $100 = 12:1
```

A healthy ratio is typically 3:1 or higher.

## Step 9: Calculate CAC Efficiency

```sql
SELECT 
    DATE_FORMAT(c.createdat, '%Y-%m') as month,
    COUNT(c.id) as new_customers,
    SUM(i.total) as revenue,
    SUM(i.total) / COUNT(c.id) as revenue_per_customer
FROM tblclients c
LEFT JOIN tblinvoices i ON c.id = i.userid AND i.status = 'Paid'
WHERE c.createdat >= DATE_SUB(NOW(), INTERVAL 12 MONTH)
GROUP BY DATE_FORMAT(c.createdat, '%Y-%m')
ORDER BY month;
```

## Step 10: Generate CAC Report

```
CUSTOMER ACQUISITION COST REPORT
Period: [dates]

MARKETING EXPENSES:
- Google Ads: $XXX
- Facebook Ads: $XXX
- Other: $XXX
- TOTAL: $XXX

CUSTOMER ACQUISITION:
Total New Customers: XXX
- Organic: XXX
- Affiliate: XXX
- Other: XXX

CAC METRICS:
Blended CAC: $XXX
Organic CAC: $XXX
Affiliate CAC: $XXX

EFFICIENCY:
Average Revenue per Customer: $XXX
CAC Payback Period: X months
LTV:CAC Ratio: X:1

CAC BY MONTH:
Month 1: $XXX
Month 2: $XXX
...
```

## CAC Calculation Checklist

- [ ] Marketing expenses gathered
- [ ] New customers counted
- [ ] Simple CAC calculated
- [ ] CAC by channel calculated
- [ ] Blended CAC calculated
- [ ] CAC by cohort calculated
- [ ] Payback period calculated
- [ ] LTV:CAC ratio calculated
- [ ] CAC efficiency calculated
- [ ] CAC report generated
