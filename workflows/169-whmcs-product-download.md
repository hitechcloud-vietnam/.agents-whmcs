---
name: whmcs-product-download
description: Configure downloadable product in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, product, download, license]
---

# WHMCS Downloadable Product Workflow

## Purpose
Step-by-step guide for setting up downloadable products.

## Prerequisites
- WHMCS admin access
- Download files ready
- Product type identified

## Step 1: Configure Product Type
- Create new product
- Select type: "Other Products & Services"
- Configure as downloadable

## Step 2: Upload Download Files
- Navigate to Setup > Downloads
- Create download category
- Upload files:
  - Software installer
  - Documentation
  - Media files
- Set file access restrictions

## Step 3: Configure Download Settings
- Set download limit
- Configure expiration
- Set access control
- Enable/disable download

## Step 4: Link to Product
- Configure product module
- Link to download
- Set automatic delivery
- Configure license generation

## Step 5: Configure License
- Enable license delivery
- Set license type:
  - Standard
  - Serial Number
  - Custom
- Configure license parameters

## Step 6: Order Fulfillment
- Automatic download on payment
- Email with download link
- License email delivery
- Access control by client

## Step 7: Test Download Flow
- Place test order
- Verify download access
- Check license delivery
- Confirm email notification

## Download Management
- Track downloads
- Monitor usage
- Update files
- Control access

## Related Workflows
- whmcs-product-create
- whmcs-product-license
- whmcs-service-provisioning