---
name: whmcs-payment-batch
description: Batch payment processing in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, batch, billing]
---

# WHMCS Batch Payment Workflow

## Purpose
Step-by-step guide for processing batch payments in WHMCS.

## Prerequisites
- WHMCS admin access
- Multiple payments to process
- Payment file (CSV/Excel)
- Batch processing configured

## Step 1: Prepare Payment File
Export/csv format:
- Client identifier
- Amount
- Payment method
- Reference
- Date

## Step 2: Access Batch Processing
- Navigate to WHMCS Admin > Billing
- Click "Batch Processing"
- Or Setup > Payments > Batch

## Step 3: Upload Payment File
- Select file from system
- Upload to WHMCS
- System validates format
- Preview payment data

## Step 4: Map Fields
Map CSV columns:
- Client ID/Email
- Amount
- Payment Date
- Reference
- Gateway

## Step 5: Configure Processing
- Select invoice matching:
  - Auto-match by client
  - Manual matching
  - Invoice ID reference
- Set default payment method
- Configure validation rules

## Step 6: Preview Batch
- View all payments
- Check for errors
- Verify amounts
- Count total payments

## Step 7: Process Batch
- Click "Process Batch"
- Monitor progress
- View results:
  - Successful payments
  - Failed payments
  - Errors list

## Step 8: Review Results
- Check success count
- Review failed items
- Resolve errors
- Generate report

## Batch Processing Options
- Credit card batch
- Bank transfer batch
- Mixed payment batch
- Recurring payment batch

## Best Practices
- Backup before batch
- Test with small batch first
- Validate file format
- Keep transaction records

## Related Workflows
- whmcs-payment-allocation
- whmcs-payment-manual
- whmcs-payment-export