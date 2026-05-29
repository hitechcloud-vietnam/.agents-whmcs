---
name: whmcs-payment-credit
description: Apply credit to payment in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, credit, billing]
---

# WHMCS Apply Credit Payment Workflow

## Purpose
Step-by-step guide for applying client credit to payments in WHMCS.

## Prerequisites
- WHMCS admin access
- Client credit balance
- Outstanding invoice
- Credit allocation authorization

## Step 1: Check Client Credit
- Navigate to client profile
- View "Credit Balance" field
- Note available credit amount
- Check credit history

## Step 2: Identify Application Need
- Locate invoice to pay
- Check invoice balance
- Compare to available credit

## Step 3: Access Credit Application
- Open client invoice
- Click "Apply Credit"
- Or navigate to:
  - Client > Billing > Apply Credit

## Step 4: Configure Credit Application
- Enter credit amount to apply:
  - Full credit amount
  - Partial amount
  - Up to invoice total
- Verify amount doesn't exceed:
  - Available credit
  - Invoice balance

## Step 5: Preview Application
- View original balance
- Show credit applied
- Display new balance
- Confirm calculation

## Step 6: Execute Application
- Click "Apply Credit"
- System deducts from credit balance
- Invoice balance reduced
- Transaction recorded

## Step 7: Handle Remaining Balance
- If invoice has remainder:
  - Client pays difference
  - Or add to credit balance
- If credit exceeds:
  - Remainder stays in credit
  - Future invoice use

## Step 8: Verification
- Credit balance updated
- Invoice status checked
- Transaction logged
- Client notified

## Credit Sources
- Overpayment refund credit
- Credit note from refund
- Promotional credit
- Service adjustment credit
- Manual credit adjustment

## Credit Management
- View credit history
- Add manual credit
- Expire old credits
- Set credit expiration policy

## Related Workflows
- whmcs-payment-allocation
- whmcs-invoice-credit-note
- whmcs-payment-manual