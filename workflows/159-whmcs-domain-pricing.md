---
name: whmcs-domain-pricing
description: Update domain pricing in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, pricing, billing]
---

# WHMCS Domain Pricing Update Workflow

## Purpose
Step-by-step guide for configuring domain pricing.

## Prerequisites
- WHMCS admin access
- Domain pricing configured
- Registrar module active

## Step 1: Access Pricing Configuration
- Navigate to Setup > Domains
- Click "Domain Pricing"
- View current pricing table

## Step 2: Select TLD Category
- Select TLD group (e.g., .com, .net)
- Or select specific TLD
- View current pricing

## Step 3: Set Registration Price
- Enter registration cost
- Set registration price for clients
- Configure markup
- Set minimum price

## Step 4: Configure Renewal Price
- Enter renewal cost
- Set client renewal price
- Configure auto-renewal pricing
- Set renewal period pricing

## Step 5: Set Transfer Price
- Enter transfer cost
- Set client transfer price
- Configure transfer markup
- Set transfer eligibility

## Step 6: Bulk Update Pricing
To update multiple TLDs:
- Select TLDs to update
- Choose update type
- Set percentage increase
- Apply changes

## Step 7: Set Currency
- Configure currency for pricing
- Set multi-currency pricing
- Configure exchange rates
- Set regional pricing

## Step 8: Review and Save
- Preview pricing changes
- Verify all settings
- Click "Save Changes"
- Update pricing table

## Pricing Options
- Registration pricing
- Renewal pricing
- Transfer pricing
- Bulk registration
- Multi-year registration

## Price Management
- Cost-based markup
- Percentage increase
- Fixed margin
- Competitive pricing

## Related Workflows
- whmcs-domain-register
- whmcs-domain-renew
- whmcs-domain-transfer