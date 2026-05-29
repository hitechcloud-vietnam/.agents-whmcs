---
name: whmcs-product-edit
description: Edit product in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, product, edit, catalog]
---

# WHMCS Product Editing Workflow

## Purpose
Step-by-step guide for editing existing products.

## Prerequisites
- WHMCS admin access
- Product ID
- Existing product configuration

## Step 1: Access Product Editor
- Navigate to Catalog > Products/Services
- Find product to edit
- Click "Edit" button

## Step 2: Edit Basic Information
- Modify product name
- Update description
- Change product group
- Update type if needed

## Step 3: Update Pricing
- Modify pricing for cycles
- Add/remove billing cycles
- Update setup fees
- Adjust tax settings
- Note: Existing orders keep original pricing

## Step 4: Modify Module Settings
- Change server assignment
- Update module configuration
- Adjust package settings
- Configure new options

## Step 5: Update Custom Fields
- Add new custom fields
- Modify existing fields
- Remove unused fields
- Update field options

## Step 6: Adjust Order Settings
- Change auto-setup settings
- Update termination options
- Modify upgrade settings
- Configure proration

## Step 7: Review Changes
- Preview all modifications
- Check impact on existing orders
- Verify pricing updates
- Confirm module changes

## Step 8: Save Changes
- Click "Save Changes"
- System updates product
- Review confirmation
- Notify affected clients if needed

## Editing Considerations
- Existing orders may not be affected
- Pricing changes apply to new orders
- Module changes affect provisioning
- Custom fields changes apply immediately

## Related Workflows
- whmcs-product-create
- whmcs-product-clone
- whmcs-product-pricing