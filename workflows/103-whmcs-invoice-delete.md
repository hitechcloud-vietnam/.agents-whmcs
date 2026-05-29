---
name: whmcs-invoice-delete
description: Delete invoice in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, billing, delete]
---

# WHMCS Invoice Deletion Workflow

## Purpose
Step-by-step guide for deleting invoices in WHMCS with proper procedures.

## Prerequisites
- WHMCS admin access
- Invoice ID to delete
- Appropriate permissions (admin/owner)

## Step 1: Verify Deletion Eligibility
- Check invoice status:
  - Draft: Can delete
  - Unpaid: Void instead (recommended)
  - Paid: Create credit note instead
  - Voided: Can delete
  - Cancelled: Can delete

## Step 2: Pre-Deletion Checks
- Verify no payments applied
- Check no credit allocated
- Ensure no related transactions
- Review audit trail implications

## Step 3: Alternative Recommendations
For different statuses:
- Unpaid: Use Void workflow
- Paid: Use Credit Note workflow
- Partial Payment: Reverse payment first

## Step 4: Execute Deletion
- Navigate to invoice details
- Click "Delete Invoice" button
- Confirm deletion in dialog
- System removes invoice record

## Step 5: Post-Deletion Actions
- Document deletion reason
- Update client records if needed
- Check accounting records
- Remove from reports

## Step 6: Verification
- Confirm invoice removed from list
- Verify client balance adjusted
- Check audit log entry created
- Ensure no orphaned transactions

## Safety Considerations
- Deletion is permanent
- Create backup before bulk operations
- Consider archiving instead
- Maintain audit trail

## Related Workflows
- whmcs-invoice-void
- whmcs-invoice-credit-note
- whmcs-invoice-payment