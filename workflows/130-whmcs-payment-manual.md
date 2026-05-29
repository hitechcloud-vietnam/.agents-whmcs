---
name: whmcs-payment-manual
description: Manual payment entry in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, manual, billing]
---

# WHMCS Manual Payment Workflow

## Purpose
Step-by-step guide for entering manual payments in WHMCS.

## Prerequisites
- WHMCS admin access
- Payment received (cash, check, wire)
- Client invoice ready
- Payment documentation

## Step 1: Prepare Payment Info
Gather payment details:
- Amount received
- Payment method:
  - Cash
  - Check
  - Bank transfer
  - Money order
- Transaction reference
- Payment date
- Client name

## Step 2: Locate Invoice
- Navigate to WHMCS Admin > Billing > Invoices
- Search for client invoice
- Verify invoice is unpaid
- Note invoice total

## Step 3: Access Manual Payment
- Open invoice details
- Click "Add Payment" button
- Or access via:
  - Client > Add Transaction

## Step 4: Enter Payment Details
- Select payment type: Manual
- Enter amount received
- Select payment method:
  - Cash
  - Check
  - Bank Transfer
  - Money Order
  - Other
- Enter date received
- Enter reference/transaction ID
- Add notes (optional)

## Step 5: Apply to Invoice
- Confirm invoice selection
- Verify amount matches
- Check allocation
- Review transaction

## Step 6: Record Payment
- Click "Submit Payment"
- System records transaction
- Invoice status updated
- Payment logged

## Step 7: Post-Entry Tasks
- Store check/receipt copy
- Update physical records
- Generate receipt
- Send confirmation to client

## Manual Payment Types
- Cash payments at office
- Check payments by mail
- Bank wire transfers
- Money orders
- Western Union
- Cashier's checks

## Common Uses
- Overphone payments
- In-person payments
- Checks received by mail
- Bank transfers outside system
- Cash payments

## Related Workflows
- whmcs-payment-accept
- whmcs-payment-allocation
- whmcs-invoice-payment