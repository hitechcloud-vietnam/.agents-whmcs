---
name: whmcs-product-clone
description: Clone product in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, product, clone, duplicate]
---

# WHMCS Product Clone Workflow

## Purpose
Step-by-step guide for cloning/duplicating products.

## Prerequisites
- WHMCS admin access
- Product to clone
- Target group for clone

## Step 1: Access Clone Function
- Navigate to Catalog > Products/Services
- Find product to clone
- Click "Clone" button
- Or via Actions menu

## Step 2: Configure New Product
- Enter new product name
- Select target product group
- Set SKU if used
- Verify product type

## Step 3: Review Cloned Settings
The following are copied:
- Pricing structure
- Module settings
- Custom fields
- Configurable options
- Order form settings

## Step 4: Modify as Needed
- Update product name
- Adjust pricing
- Change server assignment
- Modify module settings
- Update description

## Step 5: Adjust Pricing
- Review cloned pricing
- Set new pricing
- Adjust markup
- Configure billing cycles

## Step 6: Configure Module
- Select appropriate server
- Set package name
- Configure module options
- Test module connection

## Step 7: Finalize Clone
- Review all settings
- Click "Save Changes"
- New product created
- Add to order form

## Clone Use Cases
- Similar product tier
- Different server/location
- Regional product variation
- Test environment

## Related Workflows
- whmcs-product-create
- whmcs-product-edit
- whmcs-product-pricing