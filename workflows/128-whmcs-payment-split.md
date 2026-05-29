---
name: whmcs-payment-split
description: Split payment across invoices in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, split, billing]
---

# WHMCS Payment Split Workflow

## Purpose
Step-by-step guide for splitting a single payment across multiple invoices.

## Prerequisites
- WHMCS admin access
- Payment received from client
- Multiple invoices to pay

## Step 1: Receive Payment
- Payment deposited to account
- Note payment amount
- Identify source client
- Check for transaction reference

## Step 2: Locate Client Account
- Search client in WHMCS
- Open client profile
- View all outstanding invoices
- Note invoice amounts

## Step 3: Determine Split Strategy
- Total invoice amount
- Payment amount
- Calculate split:
  - Pay invoices in full
  - Split partial across invoices
  - Apply to highest priority first

## Step 4: Initiate Split Payment
- Navigate to client payments
- Click "Allocate Payment"
- Enter total payment amount

## Step 5: Configure Split
- Select first invoice
- Enter amount to apply
- Select second invoice
- Enter amount to apply
- Continue for all invoices

## Step 6: Handle Remainder
- If overpaid:
  - Apply to credit balance
  - Create credit note
- If underpaid:
  - Record partial payment
  - Mark invoice partial
  - Set balance remaining

## Step 7: Confirm Split
- Review allocation summary
- Verify each invoice amount
- Check total equals payment
- Click "Apply Payment"

## Step 8: Verification
- All selected invoices updated
- Payment transaction created
- Receipts generated
- Client notified

## Split Scenarios
1. Exact match: Payment covers all invoices
2. Overpayment: Excess to credit
3. Partial: Some invoices remain unpaid
4. Prepayment: More than total invoices

## Reporting
- Track split payments
- Generate split payment report
- Monitor payment allocation

## Related Workflows
- whmcs-payment-allocation
- whmcs-payment-credit
- whmcs-invoice-split