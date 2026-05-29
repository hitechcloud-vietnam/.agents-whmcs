---
name: whmcs-product-tax
description: Configure product tax settings in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, product, tax, billing]
---

# WHMCS Product Tax Configuration Workflow

## Purpose
Step-by-step guide for configuring tax on products.

## Prerequisites
- WHMCS admin access
- Tax rules defined
- Products configured

## Step 1: Access Tax Settings
- Navigate to Configuration > Billing > Tax
- Or Product > Tax Settings
- View tax configuration

## Step 2: Configure Tax Rules
- Set tax level: Inclusive or Exclusive
- Configure tax states:
  - Product-based tax
  - Account-based tax
  - Location-based tax
- Enable multiple tax levels

## Step 3: Configure Product Tax
Per product settings:
- Taxable: Yes/No
- Tax class
- Tax exemption
- Tax rate override

## Step 4: Set Tax per Product
- Edit product
- Set tax settings
- Choose tax rule
- Configure tax override

## Step 5: Handle Tax Exemptions
- Enable exemption feature
- Configure exemption process
- Set exemption verification
- Handle tax-exempt clients

## Step 6: Regional Tax Rules
- Set country-specific rules
- Configure EU VAT
- Configure US sales tax
- Handle digital goods tax

## Step 7: Verify Tax Calculation
- Test with different clients
- Verify tax jurisdictions
- Check multiple tax levels
- Confirm tax display

## Tax Configuration Options
- Tax inclusive pricing
- Tax exclusive pricing
- Multiple tax levels
- Location-based tax
- Product-based tax

## Related Workflows
- whmcs-product-pricing
- whmcs-invoice-tax-adjust
- whmcs-tax-rule-setup