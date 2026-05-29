---
name: whmcs-payment-refund
description: Process payment refund in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, refund, billing]
---

# WHMCS Payment Refund Workflow

## Purpose
Step-by-step guide for processing payment refunds in WHMCS.

## Prerequisites
- WHMCS admin access
- Original payment record
- Refund authorization
- Gateway refund capability

## Step 1: Locate Original Payment
- Navigate to WHMCS Admin > Billing > Transactions
- Search by invoice or client
- Open transaction details
- Note transaction ID and gateway

## Step 2: Verify Refund Eligibility
- Check payment status
- Determine refund type:
  - Full refund
  - Partial refund
- Check time limits (gateway dependent)
- Confirm refund reason

## Step 3: Initiate Refund
- Click "Refund" on transaction
- Select refund amount:
  - Full refund
  - Partial amount
- Choose refund method:
  - Original payment method
  - Credit balance
  - Manual method

## Step 4: Process via Gateway
For credit card (Stripe):
- Open Stripe dashboard
- Navigate to payment
- Create refund
- Record transaction ID

For PayPal:
- Open PayPal dashboard
- Locate transaction
- Issue refund
- Record reference ID

## Step 5: Record in WHMCS
- Enter refund amount
- Add transaction notes
- Link to original payment
- Update client credit if applicable

## Step 6: Send Notification
- Email client refund confirmation
- Include refund amount
- Include transaction reference
- Add expected processing time

## Step 7: Verify Refund Completion
- Confirm gateway processed
- Check client account updated
- Verify accounting records
- Document in audit log

## Refund Policies
- Time limits: 30-90 days typical
- Partial refund window: Varies by gateway
- Processing time: 3-10 business days
- Refund fees: May apply

## Related Workflows
- whmcs-invoice-refund
- whmcs-payment-reversal
- whmcs-payment-receipt