---
name: whmcs-payment-allocation
description: Allocate payment in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, allocation, billing]
---

# WHMCS Payment Allocation Workflow

## Purpose
Step-by-step guide for allocating payments to invoices in WHMCS.

## Prerequisites
- WHMCS admin access
- Client payment received
- Multiple invoices for allocation

## Step 1: Access Payment Allocation
- Navigate to WHMCS Admin > Billing > Payments
- Click "Allocate Payment"
- Or from client account:
  - Open client profile
  - Click "Payments"
  - Select "Add Payment"

## Step 2: Select Client
- Search client by name/email
- Select client from results
- View client invoice list

## Step 3: Enter Payment Details
- Payment amount received
- Payment method:
  - Credit card
  - Bank transfer
  - Cash
  - Check
- Transaction ID (if applicable)
- Payment date

## Step 4: View Outstanding Invoices
- List all unpaid invoices
- Show invoice amounts
- Display due dates
- Sort by priority

## Step 5: Allocate to Invoices
Auto Allocation:
- Click "Auto Allocate"
- System applies to oldest first

Manual Allocation:
- Select invoice 1
- Enter allocation amount
- Select invoice 2
- Enter allocation amount
- Continue until allocated

## Step 6: Handle Overpayment
- If payment exceeds invoice total:
  - Apply excess to credit balance
  - Or allocate to next invoice
- Credit for future use

## Step 7: Confirm Allocation
- Review allocation summary
- Verify amounts correct
- Check total matches payment
- Click "Apply Payment"

## Step 8: Post-Allocation Actions
- Invoices marked paid
- Update client balance
- Generate receipts
- Send confirmation email

## Allocation Options
- Full allocation to single invoice
- Split across multiple invoices
- Partial payment with credit
- Prepayment allocation

## Related Workflows
- whmcs-payment-credit
- whmcs-invoice-payment
- whmcs-payment-batch