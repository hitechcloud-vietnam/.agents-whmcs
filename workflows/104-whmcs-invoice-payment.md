---
name: whmcs-invoice-payment
description: Process invoice payment in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, payment, billing]
---

# WHMCS Invoice Payment Processing Workflow

## Purpose
Step-by-step guide for processing invoice payments in WHMCS.

## Prerequisites
- WHMCS admin access
- Invoice ID
- Payment method configured
- Client payment details

## Step 1: Locate Invoice
- Navigate to WHMCS Admin > Billing > Invoices
- Search and open target invoice
- Verify invoice is in "Unpaid" status

## Step 2: Verify Payment Amount
- Check invoice total
- Review any existing credits
- Calculate amount due
- Confirm currency matches payment

## Step 3: Select Payment Method
- Choose appropriate payment gateway
- Or select "Manual Payment" for offline
- Verify method is active and configured

## Step 4: Process Payment
- Click "Pay Invoice" button
- Enter payment amount
- Select payment method
- Add transaction notes (optional)
- Confirm payment details

## Step 5: Execute Payment
- For online: Gateway redirects/processes
- For manual: Record payment manually
- Enter payment date (defaults to today)
- Enter reference/transaction ID
- Click "Submit Payment"

## Step 6: Payment Confirmation
- System validates payment
- Invoice status changes to "Paid"
- Transaction recorded in WHMCS
- Client notified (if configured)
- Related services activated/updated

## Step 7: Post-Payment Tasks
- Generate receipt (optional)
- Update accounting records
- Mark related orders complete
- Trigger service provisioning

## Payment Methods
- Credit Card (via gateway)
- PayPal
- Bank Transfer
- Cheque
- Cash
- Custom payment methods

## Related Workflows
- whmcs-payment-accept
- whmcs-payment-manual
- whmcs-invoice-refund
- whmcs-payment-receipt