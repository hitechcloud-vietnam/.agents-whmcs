---
name: whmcs-payment-method-add
description: Add payment method in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, method, add]
---

# WHMCS Add Payment Method Workflow

## Purpose
Step-by-step guide for adding a new payment method in WHMCS.

## Prerequisites
- WHMCS admin access
- Payment module file
- Gateway credentials

## Step 1: Access Payment Methods
- Navigate to WHMCS Admin > Setup > Payments
- Click "Payment Gateways"
- View installed methods

## Step 2: Install New Method
- Click "Manage Existing Gateways"
- View available modules
- Find desired payment method
- Click "Install"

## Step 3: Configure Module
- Enter module settings
- Configure credentials:
  - API Key
  - Merchant ID
  - Secret key
- Set parameters:
  - Currency
  - Transaction mode
  - Payment types

## Step 4: Set Display Options
- Configure display name
- Set visible in order form
- Choose icon/logo
- Configure description

## Step 5: Test Configuration
- Enable test mode
- Process test transaction
- Verify integration
- Check webhook callback

## Step 6: Activate Method
- Set to active
- Order payment methods
- Set as default if needed
- Publish to order form

## Step 7: Verify Operation
- Test live transaction
- Check transaction record
- Verify email notification
- Monitor initial transactions

## Payment Methods Available
- Credit Card (Stripe, PayPal, etc.)
- Digital Wallets (PayPal, etc.)
- Bank Transfer
- Cryptocurrency
- Regional methods

## Related Workflows
- whmcs-payment-methods
- whmcs-payment-gateway
- whmcs-payment-method-remove