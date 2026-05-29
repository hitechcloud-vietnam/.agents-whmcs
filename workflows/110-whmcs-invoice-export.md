---
name: whmcs-invoice-export
description: Export invoices from WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, export, billing]
---

# WHMCS Invoice Export Workflow

## Purpose
Step-by-step guide for exporting invoices from WHMCS for reporting or migration.

## Prerequisites
- WHMCS admin access
- Export permissions
- Output format selected

## Step 1: Access Export Function
- Navigate to WHMCS Admin > Billing > Invoices
- Click "Export" button
- Or access via Reports > Invoice Reports

## Step 2: Configure Export Filters
- Date Range:
  - Custom date range
  - Month/Quarter/Year
  - All dates
- Status Filter:
  - All statuses
  - Paid only
  - Unpaid only
  - Void only
- Client Filter:
  - All clients
  - Specific client
  - Client group

## Step 3: Select Export Format
- CSV (Comma Separated Values)
- Excel (.xlsx)
- PDF (merged invoice reports)
- XML (for integrations)

## Step 4: Choose Export Fields
Select columns to include:
- Invoice Number
- Invoice Date
- Due Date
- Client Name
- Client Email
- Subtotal
- Tax Amount
- Total
- Balance
- Status
- Payment Method
- Created By

## Step 5: Configure Advanced Options
- Include line items: Yes/No
- Include payments: Yes/No
- Include notes: Yes/No
- Encoding: UTF-8, ISO-8859-1
- Date format: YYYY-MM-DD, MM/DD/YYYY

## Step 6: Generate Export
- Click "Generate Export"
- System processes data
- Download file when ready
- Save to local system

## Step 7: Verify Export Data
- Open exported file
- Verify record count
- Check data accuracy
- Validate formatting
- Archive for records

## Export Use Cases
- Accounting software integration
- Financial auditing
- Tax reporting
- Client billing history
- Data backup

## Scheduled Exports
Configure automated exports:
- Daily invoice summary
- Weekly report
- Monthly financial close
- Quarterly tax report

## Related Workflows
- whmcs-invoice-import
- whmcs-payment-export
- whmcs-report-generation