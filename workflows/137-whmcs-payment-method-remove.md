---
name: whmcs-payment-method-remove
description: Remove payment method in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, method, remove]
---

# WHMCS Remove Payment Method Workflow

## Purpose
Step-by-step guide for removing a payment method in WHMCS.

## Prerequisites
- WHMCS admin access
- Method to remove identified
- No active transactions

## Step 1: Verify Method Status
- Check active transactions
- Review pending payments
- Verify no scheduled payments
- Check client default methods

## Step 2: Review Impact
- How many clients use this?
- What percentage of payments?
- Alternative methods available?
- Migration plan needed?

## Step 3: Notify Clients
- If replacing:
  - Notify clients of change
  - Provide new method instructions
  - Set transition period
- Update default methods

## Step 4: Access Payment Methods
- Navigate to Setup > Payments
- Click "Payment Gateways"
- Find method to remove

## Step 5: Deactivate Method
- Click module settings
- Set to "Disabled"
- Remove from order form
- Remove as default

## Step 6: Remove Module
- Click "Uninstall"
- Confirm removal
- System removes module

## Step 7: Post-Removal
- Verify no broken references
- Test alternative methods
- Monitor for issues
- Update documentation

## Removal Considerations
- Some methods required by law
- Keep alternatives available
- Document reason for removal
- Track client impact

## Related Workflows
- whmcs-payment-method-add
- whmcs-payment-methods
- whmcs-payment-default