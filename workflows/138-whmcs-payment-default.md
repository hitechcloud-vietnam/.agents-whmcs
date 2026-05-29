---
name: whmcs-payment-default
description: Set default payment method in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, default, billing]
---

# WHMCS Set Default Payment Method Workflow

## Purpose
Step-by-step guide for setting default payment methods in WHMCS.

## Prerequisites
- WHMCS admin access
- Payment methods configured
- Active payment methods available

## Step 1: Access Payment Settings
- Navigate to Setup > Payments
- Click "Payment Gateways"
- View all payment methods

## Step 2: Access Default Settings
- Navigate to Configuration > Billing > Payment Methods
- Or client profile > Payment Methods

## Step 3: Set System Default
- Select payment method
- Click "Set as Default"
- Confirm action
- System updates default

## Step 4: Configure Default per Client
For specific clients:
- Open client profile
- Click "Payment Methods"
- Set preferred method
- Configure auto-payment

## Step 5: Set Default per Product
- Navigate to product
- Click "Pricing" tab
- Set available payment methods
- Define default for product

## Step 6: Configure Recurring Defaults
- For automatic billing:
  - Set default card on file
  - Configure auto-charge
  - Set retry rules

## Step 7: Verify Default
- Test checkout process
- Verify default selected
- Check client override
- Confirm system works

## Default Configuration Options
- Global system default
- Per-client default
- Per-product default
- Per-order default

## Related Workflows
- whmcs-payment-methods
- whmcs-payment-method-add
- whmcs-payment-method-remove