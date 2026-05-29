---
name: whmcs-invoice-credit-note
description: Create credit note in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, credit-note, billing]
---

# WHMCS Credit Note Workflow

## Purpose
Step-by-step guide for creating credit notes in WHMCS.

## Prerequisites
- WHMCS admin access
- Paid invoice (or partial payment)
- Credit note authorization
- Valid reason for credit

## Step 1: Access Credit Note Function
- Navigate to WHMCS Admin > Billing > Invoices
- Locate paid or partially paid invoice
- Click "Create Credit Note" button
- Or via Orders > Credit Invoices

## Step 2: Verify Credit Note Eligibility
- Invoice has been paid (full or partial)
- No existing credit note for same invoice
- Valid reason documented
- Authorization obtained

## Step 3: Select Credit Type
- Full Credit: Entire invoice amount
- Partial Credit: Specific amount
- Service Credit: For specific service
- Promotional Credit: Discount or credit

## Step 4: Configure Credit Note
- Link to original invoice
- Enter credit amount
- Select reason:
  - Service Issue
  - Overcharge
  - Customer Satisfaction
  - Promotional
  - Other
- Add internal notes

## Step 5: Specify Credit Application
- To Client Credit Balance:
  - Add to client's credit account
  - Use for future invoices
- To Original Payment Method:
  - Refund to credit card
  - Refund to PayPal
  - Bank transfer refund
- To New Invoice:
  - Apply to specific invoice

## Step 6: Generate Credit Note
- Review credit note details
- Verify amount and linked invoice
- Check credit destination
- Click "Create Credit Note"

## Step 7: Post-Creation Actions
- Send credit note to client
- Process refund if applicable
- Update client credit balance
- Document in audit trail
- Update financial records

## Credit Note Uses
1. Refund paid invoice
2. Correct overcharge
3. Service outage compensation
4. Customer retention credit
5. Promotional adjustment

## Credit Note Templates
- Standard credit note
- Refund credit note
- Service credit memo
- Promotional credit

## Related Workflows
- whmcs-invoice-refund
- whmcs-payment-credit
- whmcs-invoice-void