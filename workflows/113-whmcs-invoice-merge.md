---
name: whmcs-invoice-merge
description: Merge multiple invoices in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, merge, billing]
---

# WHMCS Invoice Merge Workflow

## Purpose
Step-by-step guide for merging multiple invoices into one in WHMCS.

## Prerequisites
- WHMCS admin access
- Multiple invoices (same client)
- Invoices in compatible status
- No payments applied

## Step 1: Verify Merge Eligibility
- All invoices must be for same client
- Invoices should be unpaid
- No partial payments applied
- No credits allocated
- Compatible currencies

## Step 2: Access Merge Function
- Navigate to WHMCS Admin > Billing > Invoices
- Select multiple invoices via checkbox
- Click "Merge Invoices" action
- Or access via Tools menu

## Step 3: Select Invoices to Merge
- Checkbox select method:
  - Select first invoice (target)
  - Ctrl+click to select additional
  - Confirm all selected invoices
- Verify all same client

## Step 4: Configure Merge Options
- Target invoice: Choose primary invoice
- Date: Use first invoice date or new date
- Due date: Set new or use earliest
- Notes: Preserve or clear

## Step 5: Review Combined Items
- View all line items from selected invoices
- Arrange items by date or category
- Remove duplicates if any
- Verify totals calculation

## Step 6: Execute Merge
- Confirm merge action
- System creates merged invoice
- Original invoices marked as "Merged"
- Links preserved in notes

## Step 7: Post-Merge Actions
- Review merged invoice details
- Send to client if appropriate
- Update tracking
- Document merge in audit log

## Merge Scenarios
1. Multiple small invoices for same client
2. Recurring invoice merge for period
3. Related service invoices

## Merge Limitations
- Different clients: Cannot merge
- Paid invoices: Cannot merge
- Different currencies: Cannot merge
- Mixed currencies: Convert first

## Reversal Process
To undo merge:
- Open merged invoice
- View original invoice links
- Cannot automatically split
- Create separate invoices if needed

## Related Workflows
- whmcs-invoice-split
- whmcs-invoice-creation
- whmcs-invoice-editing