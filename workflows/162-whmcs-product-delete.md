---
name: whmcs-product-delete
description: Delete product in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, product, delete, catalog]
---

# WHMCS Product Deletion Workflow

## Purpose
Step-by-step guide for deleting products from WHMCS.

## Prerequisites
- WHMCS admin access
- Product ID
- No active orders

## Step 1: Verify Deletion Eligibility
- Check for active orders
- Verify no pending orders
- Check no subscriptions
- Review order history

## Step 2: Migrate Existing Services
If product has active orders:
- Decide on migration path
- Migrate to similar product
- Communicate with clients
- Complete migrations first

## Step 3: Access Delete Function
- Navigate to Catalog > Products/Services
- Find product
- Click "Delete" button
- Confirm deletion

## Step 4: Handle Replacement
Select action:
- No replacement (hide)
- Replace with similar product
- Archive for reference

## Step 5: Confirm Deletion
- Review product details
- Confirm no active services
- Enter product name to confirm
- Click "Delete Product"

## Step 6: Post-Deletion
- Verify product removed from lists
- Update order form
- Remove from reports
- Archive documentation

## Deletion Safety
- Cannot delete product with orders
- Historical data preserved
- Invoices remain intact
- Order history maintained

## Alternative to Deletion
- Hide product from order form
- Mark as discontinued
- Keep for reference
- Archive instead

## Related Workflows
- whmcs-product-create
- whmcs-product-hidden
- whmcs-service-cancellation