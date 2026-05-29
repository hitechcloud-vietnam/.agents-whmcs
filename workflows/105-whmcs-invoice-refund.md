---
name: whmcs-invoice-refund
description: Refund invoice payment in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, refund, billing]
---

# WHMCS Invoice Refund Workflow

## Purpose
Step-by-step guide for processing invoice refunds in WHMCS.

## Prerequisites
- WHMCS admin access
- Paid invoice
- Original payment record
- Refund authorization

## Step 1: Locate Paid Invoice
- Navigate to WHMCS Admin > Billing > Invoices
- Search for paid invoice
- Verify payment status: "Paid"

## Step 2: Verify Payment Record
- Open transaction history
- Check original payment method
- Note transaction ID
- Determine refund eligibility

## Step 3: Determine Refund Type
- Full Refund: Complete payment reversal
- Partial Refund: Specific amount
- Credit Only: Refund to client credit
- Original Payment Method: Gateway refund

## Step 4: Create Refund Record
- Click "Refund Invoice" button
- Select refund type
- Enter refund amount
- Choose refund method:
  - To Credit Balance
  - To Original Payment
  - To New Payment Method

## Step 5: Process Refund
- Add refund notes/reason
- Verify amount is valid
- Click "Process Refund"
- System processes refund based on method

## Step 6: For Gateway Refunds
- If refunding to credit card:
  - Access payment gateway
  - Submit refund request
  - Record refund transaction
- If refunding to PayPal:
  - Process via PayPal dashboard
  - Record transaction ID

## Step 7: Post-Refund Actions
- Update invoice status if partial
- Create credit note if needed
- Notify client of refund
- Update accounting records
- Document in audit log

## Refund Scenarios
- Full refund with service cancellation
- Partial refund for downgraded service
- Credit balance for future use
- Third-party refund (PayPal, Stripe)

## Compliance Notes
- Follow refund policy
- Maintain refund documentation
- Process within timeframes
- Track refund statistics

## Related Workflows
- whmcs-payment-refund
- whmcs-invoice-credit-note
- whmcs-invoice-void