---
name: whmcs-payment-reversal
description: Reverse payment in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, reversal, billing]
---

# WHMCS Payment Reversal Workflow

## Purpose
Step-by-step guide for reversing payments in WHMCS.

## Prerequisites
- WHMCS admin access
- Payment record
- Reversal reason
- Gateway support

## Step 1: Identify Reversal Need
- Duplicate payment detected
- Incorrect amount paid
- Fraudulent transaction
- Client dispute
- Authorization void

## Step 2: Verify Reversal Eligibility
- Check payment status
- Check gateway support:
  - Stripe: Supports full/partial refund
  - PayPal: Supports refund
  - Authorize.net: Supports refund
- Check time limits
- Determine reversal type

## Step 3: Initiate Reversal
- Navigate to transaction
- Click "Reverse" or "Refund"
- Select reversal amount:
  - Full reversal
  - Partial reversal
- Confirm reversal reason

## Step 4: Process Reversal
- Execute via payment gateway
- For ACH/Bank transfer:
  - Contact bank
  - Submit reversal request
  - Wait for processing
- For card payments:
  - Submit refund via gateway
  - Wait for settlement

## Step 5: Update WHMCS Records
- Mark transaction reversed
- Update invoice status
- Adjust client balance
- Log reversal details

## Step 6: Handle Invoice Impact
- If invoice was paid:
  - Set back to unpaid
  - Or create credit note
- Update accounts receivable
- Adjust related orders

## Step 7: Client Communication
- Notify client of reversal
- Explain reason
- Provide reference
- Set expectations for completion

## Reversal vs Refund
| Aspect | Reversal | Refund |
|--------|----------|--------|
| Timing | Before settlement | After settlement |
| Process | Cancel before complete | Return funds |
| Fees | May avoid fees | Gateway fees apply |
| Use | Authorization expired | Post-settlement |

## Related Workflows
- whmcs-payment-refund
- whmcs-invoice-void
- whmcs-payment-dispute