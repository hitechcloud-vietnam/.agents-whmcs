---
name: whmcs-payment-methods
description: Manage payment methods in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, methods, billing]
---

# WHMCS Payment Methods Management Workflow

## Purpose
Step-by-step guide for managing payment methods in WHMCS.

## Prerequisites
- WHMCS admin access
- System configuration rights

## Step 1: Access Payment Methods
- Navigate to WHMCS Admin > Billing > Payment Methods
- View available payment methods
- Check active/inactive status

## Step 2: Configure Payment Gateway
- Select gateway module
- Configure credentials:
  - API keys
  - Merchant ID
  - Secret keys
- Set gateway settings:
  - Transaction mode (Live/Test)
  - Currency settings
  - Fee structure

## Step 3: Set Payment Method Order
- Drag/drop to reorder
- Client sees methods in order
- Set default method
- Configure visibility per product

## Step 4: Enable/Disable Methods
- Toggle active status
- Set availability by product
- Configure client groups
- Set country restrictions

## Step 5: Configure Payment Terms
- Set default payment terms
- Configure late fees
- Set auto-reminder schedule
- Configure grace period

## Step 6: Test Payment Method
- Use test mode
- Process test transaction
- Verify webhook callback
- Check email notifications

## Payment Methods Available
- Credit/Debit Card
- PayPal
- Bank Transfer
- Cheque
- Cash
- Cryptocurrency
- Custom methods

## Related Workflows
- whmcs-payment-gateway
- whmcs-payment-method-add
- whmcs-payment-method-remove