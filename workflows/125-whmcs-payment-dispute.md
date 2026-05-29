---
name: whmcs-payment-dispute
description: Handle payment dispute in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, dispute, billing]
---

# WHMCS Payment Dispute Workflow

## Purpose
Step-by-step guide for handling payment disputes in WHMCS.

## Prerequisites
- WHMCS admin access
- Disputed transaction
- Evidence documentation
- Response deadline tracking

## Step 1: Identify Dispute
- Receive dispute notification:
  - Stripe: Stripe dashboard
  - PayPal: Resolution center
  - Authorize.net: Merchant portal
- Note dispute reason:
  - Product/service not received
  - Product/service not as described
  - Unauthorized transaction
  - Credit not processed
- Review deadline

## Step 2: Gather Evidence
- Pull original transaction details
- Collect supporting documents:
  - Order confirmation
  - Delivery confirmation
  - Communication history
  - Service usage logs
  - Terms acceptance
- Document timeline

## Step 3: Review Customer Claim
- Analyze dispute reason
- Identify response points
- Prepare counter-evidence
- Assess dispute validity

## Step 4: Prepare Response
- Gather required evidence:
  - Proof of delivery
  - Service description
  - Cancellation confirmation
  - Customer communication
  - Terms and conditions
- Create dispute response

## Step 5: Submit Response
- Login to payment gateway
- Navigate to dispute
- Upload evidence
- Submit within deadline
- Confirm submission

## Step 6: Track Dispute Status
- Monitor dispute dashboard
- Check for updates
- Respond to queries
- Prepare for arbitration if needed

## Step 7: Resolution Handling
If won:
- Dispute resolved in your favor
- Funds released to you
- Update records

If lost:
- Funds debited
- Process refund internally
- Update client records
- Consider service cancellation

## Dispute Prevention
- Clear product descriptions
- Accurate billing descriptors
- Responsive customer service
- Clear cancellation policies
- Document all interactions

## Related Workflows
- whmcs-payment-chargeback
- whmcs-payment-refund
- whmcs-fraud-detection