---
name: whmcs-product-hidden
description: Hide product from order form in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, product, hidden, order-form]
---

# WHMCS Hide Product from Order Form Workflow

## Purpose
Step-by-step guide for hiding products from display.

## Prerequisites
- WHMCS admin access
- Product configured

## Step 1: Access Product Settings
- Navigate to Catalog > Products/Services
- Select product
- Open product settings

## Step 2: Configure Visibility
- Find "Order Form" settings
- Set "Show on Order Form" to No
- Or set hidden status
- Configure hidden product options

## Step 3: Set Hidden Options
- Hidden from new orders
- Visible to existing clients
- Hidden from search
- Only accessible via direct link

## Step 4: Test Hidden Status
- Check order form display
- Verify hidden from search
- Test direct access
- Confirm client visibility

## Step 5: Bulk Hide Products
- Select multiple products
- Use bulk actions
- Set hide status
- Apply to all selected

## Hidden Product Use Cases
- Discontinued products
- Legacy products
- Special offer products
- Internal products
- Client-specific products

## Related Workflows
- whmcs-product-create
- whmcs-product-order
- whmcs-product-featured