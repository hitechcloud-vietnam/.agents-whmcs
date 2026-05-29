---
name: whmcs-payment-accept
description: Accept payment in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, accept, billing]
---

# WHMCS Accept Payment Workflow

## Purpose
Step-by-step guide for accepting payments in WHMCS.

## Prerequisites
- WHMCS admin access
- Payment gateway configured
- Valid invoice
- Client payment details

## Step 1: Prepare for Payment
- Navigate to client invoice
- Verify invoice status (Unpaid)
- Check payment amount
- Confirm payment gateway availability

## Step 2: Initiate Payment
- Click "Pay Invoice" button
- Select payment method:
  - Credit Card
  - PayPal
  - Bank Transfer
  - Other gateway
- Verify gateway is active

## Step 3: Gateway Payment Process
For Credit Card:
- Enter card details
- Verify billing address
- Authorize payment

For PayPal:
- Redirect to PayPal
- Customer logs in
- Authorize payment

For Bank Transfer:
- Display bank details
- Generate reference
- Set payment pending

## Step 4: Payment Confirmation
- Gateway returns result
- System validates payment
- Invoice marked "Paid"
- Transaction recorded

## Step 5: Post-Payment Actions
- Service provisioning triggered
- Order status updated
- Client notified
- Receipt generated

## Payment Gateway Integration
- Stripe
- PayPal Pro/Express
- Authorize.net
- 2Checkout
- Custom gateway

## Related Workflows
- whmcs-payment-methods
- whmcs-payment-gateway
- whmcs-invoice-payment