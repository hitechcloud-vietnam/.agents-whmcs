---
name: whmcs-product-cycle
description: Configure product billing cycles in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, product, billing, cycle]
---

# WHMCS Product Billing Cycle Workflow

## Purpose
Step-by-step guide for configuring billing cycles.

## Prerequisites
- WHMCS admin access
- Product configured

## Step 1: Access Billing Settings
- Navigate to Product > Pricing
- Select product
- Find "Billing Cycle" section

## Step 2: Select Available Cycles
Choose cycles to offer:
- Monthly
- Quarterly
- Semi-Annual
- Annual
- Biennial
- Triennial

## Step 3: Set Cycle Pricing
For each cycle enabled:
- Enter setup fee
- Enter recurring price
- Set first payment amount
- Configure renewal amount

## Step 4: Configure Cycle Options
- Default cycle selection
- Cycle display order
- Hide/show specific cycles
- Set recommended cycle

## Step 5: Set Proration
- Enable proration
- Configure proration rules
- Set prorata billing
- Configure mid-cycle changes

## Step 6: Configure Renewals
- Set renewal automation
- Configure renewal reminders
- Set renewal pricing
- Configure termination on non-renewal

## Step 7: Advanced Cycle Options
- Trial periods
- First payment different
- Sign-up fee
- Cancellation terms per cycle

## Step 8: Save Configuration
- Review all cycles
- Verify pricing consistency
- Click "Save"
- Test ordering

## Billing Cycle Best Practices
- Offer multiple options
- Annual discount incentive
- Clear pricing display
- Easy upgrade path

## Related Workflows
- whmcs-product-pricing
- whmcs-product-upgrade
- whmcs-prorating-workflow