---
name: whmcs-product-upgrade
description: Configure product upgrade options in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, product, upgrade, configuration]
---

# WHMCS Product Upgrade Configuration Workflow

## Purpose
Step-by-step guide for configuring product upgrade options.

## Prerequisites
- WHMCS admin access
- Multiple product tiers
- Upgrade paths defined

## Step 1: Enable Upgrade/Downgrade
- Navigate to Product Settings
- Enable "Allow Upgrade/Downgrade"
- Configure upgrade paths
- Set upgrade pricing

## Step 2: Define Upgrade Paths
- Create product hierarchy
- Set allowed upgrades:
  - From Basic to Standard
  - From Standard to Premium
- Configure downgrade options

## Step 3: Configure Upgrade Pricing
- Set upgrade fees
- Configure prorata calculation
- Set immediate billing
- Configure recurring change

## Step 4: Configure Downgrade
- Enable downgrade option
- Set downgrade fees
- Configure credit on account
- Set effective date

## Step 5: Set Module Changes
- Configure package changes
- Set server package mapping
- Configure resource changes
- Enable automatic provisioning

## Step 6: Client Upgrade Flow
- Show upgrade options
- Display price difference
- Confirm upgrade selection
- Process payment
- Provision changes

## Step 7: Test Upgrade Process
- Test upgrade order
- Verify billing
- Check provisioning
- Confirm module changes
- Test downgrade flow

## Upgrade Best Practices
- Clear tier differentiation
- Reasonable upgrade pricing
- Automatic provisioning
- Clear client communication

## Related Workflows
- whmcs-product-create
- whmcs-product-pricing
- whmcs-prorating-workflow