---
name: whmcs-product-addon
description: Configure product addons in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, product, addon, upsell]
---

# WHMCS Product Addon Configuration Workflow

## Purpose
Step-by-step guide for creating and managing product addons.

## Prerequisites
- WHMCS admin access
- Base products configured
- Addon offerings defined

## Step 1: Access Addon Configuration
- Navigate to Catalog > Addon Products
- Click "Create New Addon"

## Step 2: Configure Addon Details
- Enter addon name
- Select addon type
- Enter description
- Set billing type (monthly, one-time)

## Step 3: Set Addon Pricing
- Enter setup fee
- Enter recurring price
- Configure billing cycles
- Set tax settings

## Step 4: Configure Module Settings
- Select module if applicable
- Configure provisioning
- Set module commands
- Configure options

## Step 5: Link to Products
- Select products that can have this addon
- Set as required/optional
- Configure auto-add options
- Set addon order

## Step 6: Configure Order Form
- Show on order form
- Set default selection
- Configure appearance
- Set in cart display

## Step 7: Client Management
- Allow client to add
- Allow client to remove
- Configure upgrade path
- Set termination handling

## Step 8: Test Addon Flow
- Order with addon
- Verify provisioning
- Check billing
- Test removal
- Verify upgrades

## Common Addons
- Additional storage
- Extra bandwidth
- SSL certificates
- Domain privacy
- Backup services
- Priority support

## Related Workflows
- whmcs-product-create
- whmcs-product-bundle
- whmcs-product-config