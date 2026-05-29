---
name: whmcs-payment-export
description: Export payment data from WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, export, billing]
---

# WHMCS Payment Export Workflow

## Purpose
Step-by-step guide for exporting payment data from WHMCS.

## Prerequisites
- WHMCS admin access
- Export permissions
- Output format selected

## Step 1: Access Export Function
- Navigate to WHMCS Admin > Billing > Transactions
- Click "Export" button
- Or Reports > Payment Reports

## Step 2: Configure Filters
Date Range:
- Custom dates
- Current month
- Previous month
- Quarter
- Year
- All time

Status Filter:
- All transactions
- Completed
- Pending
- Failed
- Refunded

Client Filter:
- All clients
- Specific client
- Client group

## Step 3: Select Export Format
- CSV (default)
- Excel (.xlsx)
- PDF
- OFX (for accounting software)

## Step 4: Choose Fields
Select columns:
- Transaction ID
- Date
- Client Name
- Amount
- Fee
- Net Amount
- Payment Method
- Gateway
- Invoice Number
- Status
- Reference

## Step 5: Configure Options
- Include header row
- Date format selection
- Number format
- Encoding (UTF-8)
- Compress output

## Step 6: Generate Export
- Click "Generate Export"
- System processes data
- Download file
- Save to location

## Step 7: Verify Data
- Open file
- Check record count
- Verify amounts
- Validate formatting
- Archive copy

## Export Use Cases
- Accounting software sync
- Financial auditing
- Tax reporting
- Payment reconciliation
- Bank statement matching

## Scheduled Exports
- Daily payment summary
- Weekly payment report
- Monthly reconciliation
- Quarterly tax report

## Related Workflows
- whmcs-payment-report
- whmcs-payment-reconciliation
- whmcs-invoice-export