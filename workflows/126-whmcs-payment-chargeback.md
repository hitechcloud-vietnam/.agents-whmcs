---
name: whmcs-payment-chargeback
description: Respond to chargeback in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, chargeback, billing]
---

# WHMCS Chargeback Response Workflow

## Purpose
Step-by-step guide for responding to payment chargebacks in WHMCS.

## Prerequisites
- WHMCS admin access
- Chargeback notification
- Transaction evidence
- Response deadline

## Step 1: Receive Chargeback Notice
- Payment processor notification:
  - Stripe: Dashboard alert
  - PayPal: Resolution center
  - Authorize.net: Merchant interface
- Note chargeback amount
- Record dispute deadline
- Check reason code

## Step 2: Analyze Chargeback Reason
Common reasons:
- E001: Card unauthorized
- E02: Product not received
- E03: Product not as described
- E10: Credit not processed
- E12: Duplicate transaction

## Step 3: Assess Response Strategy
- Review transaction history
- Check customer communication
- Evaluate evidence availability
- Determine likelihood of success

## Step 4: Collect Evidence
Required documents:
- Signed agreement/contract
- Product/service description
- Delivery/completion proof
- Customer acknowledgment
- Email correspondence
- Transaction records

## Step 5: Prepare Chargeback Response
- Write response letter
- Attach evidence documents
- Complete response form
- Calculate response deadline

## Step 6: Submit Response
- Access processor portal
- Navigate to chargeback
- Upload evidence
- Submit response
- Confirm receipt

## Step 7: Monitor and Update
- Track response status
- Monitor deadline alerts
- Update WHMCS records
- Set reminders for resolution

## Step 8: Resolution Actions
Won chargeback:
- Update transaction status
- Document win
- Update fraud filters

Lost chargeback:
- Debit applied
- Update client account
- Consider service review
- Document loss reason

## Chargeback Prevention
- Clear refund policy
- Responsive support
- Verify card ownership
- Clear billing descriptors
- Delivery confirmation
- Customer communication

## Fee Impact
- Chargeback fees: $15-$100 per case
- High chargeback rate: Higher fees
- Threshold: Monitor rate (1% typical)

## Related Workflows
- whmcs-payment-dispute
- whmcs-fraud-detection
- whmcs-payment-refund