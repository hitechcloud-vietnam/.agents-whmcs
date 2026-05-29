# WHMCS ARR (Annual Recurring Revenue) Calculation Workflow

## Purpose
Calculate and track annual recurring revenue

## Prerequisites
- WHMCS installed
- Admin access
- SQL/database access

## Step 1: Calculate Current ARR

```sql
SELECT 
    SUM(annual_amount) as total_arr
FROM (
    SELECT 
        h.id,
        CASE h.billingcycle
            WHEN 'Monthly' THEN h.amount * 12
            WHEN 'Quarterly' THEN h.amount * 4
            WHEN 'Semi-Annual' THEN h.amount * 2
            WHEN 'Annual' THEN h.amount
            WHEN 'Biennial' THEN h.amount / 2
            WHEN 'Triennial' THEN h.amount / 3
            ELSE h.amount
        END as annual_amount
    FROM tblhosting h
    WHERE h.domainstatus = 'Active'
) as services;
```

## Step 2: Calculate ARR by Product

```sql
SELECT 
    p.name as product,
    COUNT(h.id) as active_services,
    SUM(
        CASE h.billingcycle
            WHEN 'Monthly' THEN h.amount * 12
            WHEN 'Annual' THEN h.amount
            ELSE h.amount
        END
    ) as arr
FROM tblhosting h
JOIN tblproducts p ON h.packageid = p.id
WHERE h.domainstatus = 'Active'
GROUP BY p.name
ORDER BY arr DESC;
```

## Step 3: Calculate ARR Growth Rate

```sql
-- ARR this year vs last year
SELECT 
    (SELECT SUM(amount * 12) FROM tblhosting WHERE domainstatus = 'Active' AND regdate <= NOW()) as current_arr,
    (SELECT SUM(amount * 12) FROM tblhosting WHERE domainstatus = 'Active' AND regdate <= DATE_SUB(NOW(), INTERVAL 1 YEAR)) as previous_arr,
    ((SELECT SUM(amount * 12) FROM tblhosting WHERE domainstatus = 'Active' AND regdate <= NOW()) -
     (SELECT SUM(amount * 12) FROM tblhosting WHERE domainstatus = 'Active' AND regdate <= DATE_SUB(NOW(), INTERVAL 1 YEAR))) as arr_growth,
    (((SELECT SUM(amount * 12) FROM tblhosting WHERE domainstatus = 'Active' AND regdate <= NOW()) -
     (SELECT SUM(amount * 12) FROM tblhosting WHERE domainstatus = 'Active' AND regdate <= DATE_SUB(NOW(), INTERVAL 1 YEAR))) * 100.0 /
     (SELECT SUM(amount * 12) FROM tblhosting WHERE domainstatus = 'Active' AND regdate <= DATE_SUB(NOW(), INTERVAL 1 YEAR))) as growth_percentage;
```

## Step 4: Calculate ARR by Client Tier

```sql
SELECT 
    CASE
        WHEN client_arr >= 10000 THEN 'Enterprise'
        WHEN client_arr >= 1000 THEN 'Business'
        WHEN client_arr >= 100 THEN 'Professional'
        ELSE 'Starter'
    END as tier,
    COUNT(*) as clients,
    SUM(client_arr) as tier_arr
FROM (
    SELECT 
        c.id,
        SUM(
            CASE h.billingcycle
                WHEN 'Monthly' THEN h.amount * 12
                WHEN 'Annual' THEN h.amount
                ELSE h.amount
            END
        ) as client_arr
    FROM tblhosting h
    JOIN tblclients c ON h.userid = c.id
    WHERE h.domainstatus = 'Active'
    GROUP BY c.id
) as client_tiers
GROUP BY tier;
```

## Step 5: Calculate ARR Metrics

```sql
-- ARR per customer
SELECT 
    COUNT(DISTINCT h.userid) as total_customers,
    SUM(
        CASE h.billingcycle
            WHEN 'Monthly' THEN h.amount * 12
            ELSE h.amount
        END
    ) / COUNT(DISTINCT h.userid) as arr_per_customer
FROM tblhosting h
WHERE h.domainstatus = 'Active';

-- ARR by billing cycle
SELECT 
    billingcycle,
    COUNT(*) as services,
    SUM(amount) as period_revenue,
    SUM(
        CASE billingcycle
            WHEN 'Monthly' THEN amount * 12
            WHEN 'Quarterly' THEN amount * 4
            WHEN 'Annual' THEN amount
            ELSE amount
        END
    ) as arr
FROM tblhosting
WHERE domainstatus = 'Active'
GROUP BY billingcycle;
```

## Step 6: Generate ARR Report

```
ARR REPORT
Date: [current date]

CURRENT ARR: $XXX

ARR GROWTH:
- vs Last Month: +$XXX (XX%)
- vs Last Year: +$XXX (XX%)

ARR BY TIER:
- Enterprise: $XXX (XX clients)
- Business: $XXX (XX clients)
- Professional: $XXX (XX clients)
- Starter: $XXX (XX clients)

ARR BY BILLING:
- Monthly: $XXX
- Quarterly: $XXX
- Annual: $XXX

ARR PER CUSTOMER: $XXX
```

## ARR Calculation Checklist

- [ ] Current ARR calculated
- [ ] ARR by product calculated
- [ ] ARR growth rate calculated
- [ ] ARR by tier calculated
- [ ] ARR metrics calculated
- [ ] ARR report generated
