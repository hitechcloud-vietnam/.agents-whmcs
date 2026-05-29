# WHMCS Service Audit Workflow

## Purpose
Conduct comprehensive service/product audit

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Get Service Overview

```sql
SELECT 
    COUNT(*) as total_services,
    SUM(CASE WHEN domainstatus = 'Active' THEN 1 ELSE 0 END) as active,
    SUM(CASE WHEN domainstatus = 'Suspended' THEN 1 ELSE 0 END) as suspended,
    SUM(CASE WHEN domainstatus = 'Terminated' THEN 1 ELSE 0 END) as terminated,
    SUM(CASE WHEN domainstatus = 'Cancelled' THEN 1 ELSE 0 END) as cancelled
FROM tblhosting;
```

## Step 2: Analyze Services by Product

```sql
SELECT 
    p.name as product,
    COUNT(h.id) as total,
    SUM(CASE WHEN h.domainstatus = 'Active' THEN 1 ELSE 0 END) as active,
    SUM(CASE WHEN h.domainstatus = 'Active' THEN h.amount ELSE 0 END) as monthly_revenue
FROM tblhosting h
JOIN tblproducts p ON h.packageid = p.id
GROUP BY p.name
ORDER BY monthly_revenue DESC;
```

## Step 3: Identify Expiring Services

```sql
SELECT 
    h.id,
    c.email,
    p.name as product,
    h.domain,
    h.nextduedate,
    DATEDIFF(h.nextduedate, CURDATE()) as days_until_due
FROM tblhosting h
JOIN tblclients c ON h.userid = c.id
JOIN tblproducts p ON h.packageid = p.id
WHERE h.domainstatus = 'Active'
AND h.nextduedate BETWEEN CURDATE() AND DATE_ADD(CURDATE(), INTERVAL 30 DAY)
ORDER BY h.nextduedate;
```

## Step 4: Analyze Service Termination

```sql
SELECT 
    DATE_FORMAT(regdate, '%Y-%m') as month,
    COUNT(*) as new_services
FROM tblhosting
GROUP BY DATE_FORMAT(regdate, '%Y-%m')
ORDER BY month DESC
LIMIT 12;
```

## Step 5: Calculate Service Metrics

```sql
-- Average services per client
SELECT 
    AVG(service_count) as avg_services
FROM (
    SELECT 
        userid,
        COUNT(*) as service_count
    FROM tblhosting
    WHERE domainstatus = 'Active'
    GROUP BY userid
) as client_services;

-- Services by billing cycle
SELECT 
    billingcycle,
    COUNT(*) as count,
    SUM(amount) as revenue
FROM tblhosting
WHERE domainstatus = 'Active'
GROUP BY billingcycle;
```

## Step 6: Identify Unused Services

```sql
SELECT 
    h.id,
    c.email,
    p.name,
    h.regdate,
    h.nextduedate
FROM tblhosting h
JOIN tblclients c ON h.userid = c.id
JOIN tblproducts p ON h.packageid = p.id
WHERE h.domainstatus = 'Active'
AND DATEDIFF(h.nextduedate, CURDATE()) > 365;
```

## Step 7: Review Service Pricing

Navigate to: Setup > Products/Services > Products/Services

Verify:
- Pricing current
- Promotions expired
- Cost alignment

## Step 8: Generate Service Audit Report

```
SERVICE AUDIT REPORT

TOTAL SERVICES: XXX
- Active: XXX
- Suspended: XXX
- Terminated: XXX

MONTHLY RECURRING REVENUE: $XXX

TOP PRODUCTS BY REVENUE:
1. Product A: $XXX (XX services)
2. Product B: $XXX (XX services)

EXPIRING SERVICES (30 DAYS):
- This Week: XX
- Next Week: XX

UNDERUTILIZED SERVICES: XX

DATA QUALITY ISSUES:
- Missing next due date: XX
- Invalid renewal amounts: XX
```

## Service Audit Checklist

- [ ] Service overview obtained
- [ ] Products analyzed
- [ ] Expiring services identified
- [ ] Termination analyzed
- [ ] Metrics calculated
- [ ] Unused services found
- [ ] Pricing reviewed
- [ ] Audit report generated
