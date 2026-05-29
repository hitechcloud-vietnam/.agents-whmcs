---
name: whmcs-product-bundle
description: Configure product bundles in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, product, bundle, package]
---

# WHMCS Product Bundle Configuration Workflow

## Purpose
Step-by-step guide for creating product bundles/packages.

## Prerequisites
- WHMCS admin access
- Products available
- Bundle offering defined

## Step 1: Plan Bundle Structure
- Identify bundle components
- Calculate combined cost
- Set bundle price
- Determine savings display

## Step 2: Create Bundle Product
- Create new product
- Name it bundle name
- Set as "bundle" type
- Configure description

## Step 3: Set Bundle Pricing
- Enter combined price
- Set billing cycles
- Configure setup fee
- Set bundle-specific pricing

## Step 4: Configure Bundle Components
- Add included products
- Set quantities
- Configure included addons
- Define included services

## Step 5: Set Upgrades Within Bundle
- Define component upgrades
- Configure pricing differences
- Set upgrade paths
- Manage component changes

## Step 6: Configure Order Form
- Show bundle savings
- Display included items
- Configure component display
- Set bundle presentation

## Step 7: Test Bundle Order
- Place test order
- Verify all components
- Check combined pricing
- Confirm provisioning

## Bundle Examples
- Starter Pack (host + domain + email)
- Business Suite (host + SSL + backup)
- E-commerce Bundle (host + SSL + cart)

## Related Workflows
- whmcs-product-create
- whmcs-product-addon
- whmcs-product-pricing