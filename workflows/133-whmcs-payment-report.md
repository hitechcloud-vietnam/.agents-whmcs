---
name: whmcs-payment-report
description: Generate payment reports in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, report, billing]
---

# WHMCS Payment Report Workflow

## Purpose
Step-by-step guide for generating payment reports in WHMCS.

## Prerequisites
- WHMCS admin access
- Report permissions
- Reporting modules configured

## Step 1: Access Reports
- Navigate to WHMCS Admin > Reports
- Click "Billing" tab
- Select "Payment Reports"

## Step 2: Select Report Type
Available reports:
- Transaction Summary
- Payment by Method
- Payment by Client
- Refund Report
- Fee Report
- Failed Payments
- Chargeback Report

## Step 3: Configure Report Parameters
Date Selection:
- Date range
- Compare periods
- Year over year

Filters:
- Payment method
- Gateway
- Client group
- Amount range
- Status

Grouping:
- By day/week/month
- By payment method
- By client
- By gateway

## Step 4: Generate Report
- Click "Generate Report"
- System processes data
- Display results on screen
- View summary metrics

## Step 5: Review Report Data
Key metrics displayed:
- Total payments
- Total amount
- Total fees
- Net revenue
- Transaction count
- Average transaction

## Step 6: Export Report
- Click "Export" button
- Select format (PDF/CSV/Excel)
- Download report
- Save to location

## Step 7: Schedule Reports
- Create scheduled report
- Set frequency (daily/weekly/monthly)
- Configure recipients
- Set delivery method (email)

## Report Use Cases
- Financial review
- Fee analysis
- Payment trend analysis
- Client payment history
- Gateway performance
- Refund tracking

## Key Reports to Run
1. Daily payment summary
2. Weekly fee analysis
3. Monthly revenue report
4. Quarterly chargeback report
5. Annual payment history

## Related Workflows
- whmcs-payment-export
- whmcs-payment-reconciliation
- whmcs-report-generation