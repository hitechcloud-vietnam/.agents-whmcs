---
name: whmcs-invoice-void
description: Void invoice in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, void, billing]
---

# WHMCS Invoice Void Workflow

## Purpose
Step-by-step guide for voiding invoices in WHMCS.

## Prerequisites
- WHMCS admin access
- Invoice ID
- Invoice in valid status (Unpaid)
- Void reason

## Step 1: Verify Void Eligibility
- Check invoice status:
  - Unpaid: Can void
  - Draft: Can delete instead
  - Paid: Use refund/credit process
  - Partially Paid: Reverse payment first

## Step 2: Review Invoice Details
- Verify all line items
- Check for applied credits
- Review transaction history
- Confirm no pending payments

## Step 3: Prepare Void Documentation
- Document void reason
- Note original invoice amount
- Identify affected orders/services
- Plan client notification

## Step 4: Execute Void
- Open invoice in WHMCS Admin
- Click "Void Invoice" button
- Confirm void action
- Enter void reason (internal note)
- System marks invoice as "Void"

## Step 5: Post-Void Actions
- Reverse any pending actions
- Update client records
- Cancel related automation (reminders)
- Unlink from orders if applicable

## Step 6: Client Communication
- Send void notification (optional)
- Explain reason for void
- Provide alternative actions
- Update support ticket if exists

## Step 7: Accounting Adjustments
- Ensure no payments recorded
- Verify no credits allocated
- Update accounts receivable
- Document in audit trail

## Void vs Delete
| Aspect | Void | Delete |
|--------|------|--------|
| Status | Marked void, preserved | Removed |
| Audit | Full audit trail | Partial trail |
| Use Case | Unpaid, needs record | Drafts, errors |
| Reversibility | Cannot undo | Cannot undo |

## Related Workflows
- whmcs-invoice-delete
- whmcs-invoice-credit-note
- whmcs-invoice-payment-reversal