---
name: whmcs-product-custom
description: Configure product custom fields in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, product, custom-fields, configuration]
---

# WHMCS Product Custom Fields Workflow

## Purpose
Step-by-step guide for configuring product custom fields.

## Prerequisites
- WHMCS admin access
- Product configured
- Field requirements identified

## Step 1: Access Custom Fields
- Navigate to Product > Custom Fields
- Select product
- View existing fields

## Step 2: Create Custom Field
- Click "Add New Custom Field"
- Enter field name
- Select field type:
  - Text
  - Textarea
  - Dropdown
  - Radio
  - Checkbox
  - Password
  - Date

## Step 3: Configure Field Options
For dropdown/radio:
- Add options
- Set default selection
- Order options

For text/textarea:
- Set placeholder
- Set validation rules
- Set max length

## Step 4: Set Field Properties
- Required: Yes/No
- Show on order: Yes/No
- Show on invoice: Yes/No
- Admin only: Yes/No
- Read only: Yes/No

## Step 5: Configure Client Area
- Show in client area
- Editable by client
- Show in product details
- Hide from order form

## Step 6: Set Validation Rules
- Required field
- Minimum length
- Maximum length
- Regex pattern
- Custom validation

## Step 7: Save Field
- Test field in order
- Verify client area display
- Check invoice output
- Adjust as needed

## Common Custom Fields
- Server hostname
- Control panel URL
- Admin email
- Operating system
- Control panel preference

## Related Workflows
- whmcs-product-create
- whmcs-product-config
- whmcs-configurable-options