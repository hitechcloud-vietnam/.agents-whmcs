---
name: whmcs-domain-search
description: Domain search and availability in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, search, availability]
---

# WHMCS Domain Search Workflow

## Purpose
Step-by-step guide for searching and checking domain availability.

## Prerequisites
- WHMCS installation
- Domain search module configured
- Registrar API connected

## Step 1: Access Domain Search
- Open WHMCS client area
- Navigate to Domain Search
- Enter domain name
- Click "Search"

## Step 2: Configure Search Options
TLD Selection:
- Select TLDs to check
- Popular TLDs
- All available TLDs
- Specific categories

Search Type:
- Single domain
- Multiple domains
- Bulk search
- Domain suggestions

## Step 3: Perform Search
- System queries registrar APIs
- Check availability across TLDs
- Display results:
  - Available domains
  - Taken domains
  - Premium domains

## Step 4: Review Results
- View pricing for available
- Check availability status
- See alternative suggestions
- Compare registration costs

## Step 5: Select Domain
- Choose desired domain
- Add to cart
- Review total cost
- Proceed to checkout

## Step 6: Domain Alternatives
If taken:
- View suggested alternatives
- Check similar domains
- Consider different TLDs
- Offer backorder option

## Step 7: Complete Registration
- Select registration period
- Configure DNS options
- Add privacy protection
- Process payment

## Domain Search Features
- Real-time availability
- Bulk search
- Suggestions
- Backorder
- Transfer check

## Related Workflows
- whmcs-domain-register
- whmcs-domain-suggestion
- whmcs-domain-backorder