---
name: whmcs-payment-reconciliation
description: Payment reconciliation in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, reconciliation, billing]
---

# WHMCS Payment Reconciliation Workflow

## Purpose
Step-by-step guide for reconciling payments in WHMCS.

## Prerequisites
- WHMCS admin access
- Bank/merchant statements
- Transaction records

## Step 1: Prepare Statements
- Download bank statement
- Download gateway statement
- Prepare WHMCS transaction export
- Gather all records for period

## Step 2: Access Reconciliation
- Navigate to WHMCS Admin > Billing > Transactions
- Click "Reconciliation"
- Or Reports > Reconciliation

## Step 3: Set Reconciliation Period
- Select date range
- Define period boundaries
- Note opening balance
- Note closing balance

## Step 4: Compare Records
Compare WHMCS to Bank:
- List all WHMCS transactions
- List all bank deposits
- Match by date/amount
- Identify discrepancies

## Step 5: Identify Issues
Common issues:
- Missing WHMCS transactions
- Missing bank deposits
- Amount mismatches
- Duplicate entries
- Timing differences

## Step 6: Resolve Discrepancies
For missing WHMCS:
- Create missing transaction
- Investigate source

For missing bank:
- Check pending
- Verify processing
- Check for errors

For amount mismatch:
- Investigate fees
- Check currency
- Verify calculation

## Step 7: Document Reconciliation
- Record all adjustments
- Document findings
- Note causes of issues
- Record resolution
- Update records

## Step 8: Finalize
- Confirm all reconciled
- Sign off on reconciliation
- File documentation
- Update accounting records

## Reconciliation Schedule
- Daily: Quick check
- Weekly: Full reconciliation
- Monthly: Complete review
- Quarterly: Detailed audit

## Related Workflows
- whmcs-payment-report
- whmcs-payment-export
- whmcs-billing-audit