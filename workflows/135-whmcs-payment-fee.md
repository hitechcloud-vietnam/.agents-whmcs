---
name: whmcs-payment-fee
description: Payment fee calculation in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, fee, billing]
---

# WHMCS Payment Fee Calculation Workflow

## Purpose
Step-by-step guide for calculating and managing payment fees in WHMCS.

## Prerequisites
- WHMCS admin access
- Payment gateway configured
- Fee structure defined

## Step 1: Review Fee Structure
Gateway fees:
- Stripe: 2.9% + $0.30
- PayPal: 2.9% + $0.30
- Authorize.net: 2.9% + $0.10
- Square: 2.6% + $0.10

## Step 2: Configure Fee Settings
- Navigate to Setup > Payments
- Select gateway
- Configure fee rules:
  - Percentage fee
  - Fixed fee
  - Combined calculation
  - Minimum fee

## Step 3: Set Fee Application
Options:
- Pass fees to client
- Absorb fees
- Mark up fees
- Fee-free threshold

## Step 4: Calculate Fees
For $100 transaction:
- Stripe: $100 x 2.9% + $0.30 = $3.20
- PayPal: $100 x 2.9% + $0.30 = $3.20
- Authorize: $100 x 2.9% + $0.10 = $2.20

## Step 5: Fee Tracking
- View fees in transactions
- Generate fee report
- Track by gateway
- Monthly fee summary

## Step 6: Fee Reporting
- Navigate to Reports
- Select Fee Report
- Configure date range
- Generate report

## Step 7: Fee Optimization
Review fees:
- Compare gateway rates
- Negotiate with providers
- Consider volume discounts
- Switch if better rates available

## Fee Minimization
- Use lowest cost gateway
- Combine payments when possible
- Negotiate better rates
- Use ACH when available

## Related Workflows
- whmcs-payment-report
- whmcs-payment-gateway
- whmcs-cost-optimization