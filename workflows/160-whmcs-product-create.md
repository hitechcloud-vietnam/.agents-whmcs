---
name: whmcs-product-create
description: Create product in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, product, create, catalog]
---

# WHMCS Product Creation Workflow

## Purpose
Step-by-step guide for creating products in WHMCS.

## Prerequisites
- WHMCS admin access
- Product group defined
- Server configured (if applicable)
- Pricing determined

## Step 1: Access Product Management
- Navigate to WHMCS Admin > Catalog > Products/Services
- Click "Create New Product"

## Step 2: Configure Basic Info
- Enter product name
- Select product type:
  - Hosting Account
  - Reseller Account
  - Virtual Server
  - Other Product/Service
- Select product group
- Enter product description

## Step 3: Set Pricing
- Select billing cycle:
  - Monthly
  - Quarterly
  - Semi-Annual
  - Annual
  - Biennial
  - Triennial
- Enter price for each cycle
- Set setup fee (optional)
- Configure tax settings

## Step 4: Configure Module Settings
For hosting/products:
- Select server
- Choose module
- Configure module settings:
  - Package name
  - Options
  - Configurable options
- Set provisioning settings

## Step 5: Add Custom Fields
- Click "Custom Fields"
- Add field name
- Set field type:
  - Text
  - Dropdown
  - Checkbox
  - Textarea
- Set required/optional
- Configure validation

## Step 6: Set Product Options
- Enable/disable auto-setup
- Set allow recurring
- Configure prorata billing
- Set upgrade options
- Configure termination settings

## Step 7: Configure Order Form
- Set product appearance
- Configure order stock
- Set hidden/visible
- Add to featured products
- Configure product bundles

## Step 8: Save Product
- Review all settings
- Click "Save Changes"
- Product created
- Configure other products

## Product Types
- Shared Hosting
- VPS/Dedicated
- SSL Certificates
- Domains
- Custom Services
- Software Licenses

## Related Workflows
- whmcs-product-edit
- whmcs-product-pricing
- whmcs-product-config