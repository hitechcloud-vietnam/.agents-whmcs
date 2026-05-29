---
name: whmcs-product-config
description: Configure product configurable options in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, product, configurable-options, configuration]
---

# WHMCS Product Configurable Options Workflow

## Purpose
Step-by-step guide for configuring product configurable options.

## Prerequisites
- WHMCS admin access
- Product configured
- Options to offer defined

## Step 1: Access Configurable Options
- Navigate to Product > Configurable Options
- Select product
- View available options groups

## Step 2: Create Option Group
- Click "Create New Group"
- Enter group name
- Add options to group
- Set options as required/optional

## Step 3: Configure Option Types
Select option type:
- Dropdown
- Checkbox
- Radio
- Text
- Quantity

## Step 4: Add Options
- Enter option name
- Set option values
- Configure pricing per option:
  - Setup fee
  - Recurring fee
  - One-time fee
- Set default selection

## Step 5: Set Option Dependencies
- Configure dependencies
- Set conditional options
- Hide/show based on selection
- Set mutually exclusive options

## Step 6: Assign to Product
- Select option groups
- Assign to product
- Configure order of display
- Set in which cycle options apply

## Step 7: Test Configuration
- Place test order
- Verify options work
- Check pricing calculation
- Confirm module integration

## Common Configurable Options
- Operating System
- Control Panel
- Addon Domains
- Disk Space
- Bandwidth
- Backups

## Related Workflows
- whmcs-product-create
- whmcs-product-custom
- whmcs-configurable-options