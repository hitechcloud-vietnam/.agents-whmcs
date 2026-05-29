---
name: whmcs-invoice-debit-note
description: Create debit note in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, debit-note, billing]
---

# WHMCS Debit Note Workflow

## Purpose
Step-by-step guide for creating debit notes in WHMCS for additional charges.

## Prerequisites
- WHMCS admin access
- Existing invoice (paid or unpaid)
- Valid reason for additional charge
- Proper authorization

## Step 1: Access Debit Note Function
- Navigate to WHMCS Admin > Billing > Invoices
- Find the related invoice
- Click "Add Debit Note" or "Add Adjustment"
- Or create standalone debit note

## Step 2: Verify Debit Note Eligibility
- Original invoice exists
- Valid additional charge reason
- Documentation prepared
- Client notification planned

## Step 3: Select Debit Note Type
- Additional Charge: Extra services
- Price Adjustment: Rate change
- Penalty Fee: Late payment, contract violation
- Correction: Undercharged originally
- Additional Usage: Overage charges

## Step 4: Configure Debit Note
- Link to original invoice (optional)
- Enter debit amount
- Select reason:
  - Additional Services
  - Overage Charges
  - Price Adjustment
  - Penalty Fee
  - Correction
- Add detailed description

## Step 5: Apply to Invoice
- Add to existing invoice:
  - Creates additional balance
  - Original invoice remains
- Create separate debit note:
  - Standalone document
  - Separate payment tracking
- Add to next invoice

## Step 6: Set Debit Note Status
- Draft: Pending approval
- Sent: Client notified
- Paid: Payment received
- Cancelled: Not processed

## Step 7: Generate Debit Note
- Review debit note details
- Verify amount and description
- Check client notification
- Click "Create Debit Note"

## Step 8: Post-Creation Actions
- Send debit note to client
- Update invoice balance
- Document reason in audit trail
- Update financial records
- Monitor payment

## Debit Note Use Cases
1. Additional domain renewal fees
2. Overage bandwidth charges
3. Additional storage usage
4. Late payment penalties
5. Price correction (undercharged)

## Debit Note vs Invoice
| Aspect | Debit Note | Invoice |
|--------|------------|---------|
| Purpose | Additional charge | New charge |
| Link | Often linked to invoice | Standalone |
| Amount | Can be positive or negative | Always positive |
| Tax | Same treatment as invoice | Standard |

## Related Workflows
- whmcs-invoice-creation
- whmcs-invoice-editing
- whmcs-invoice-payment