---
name: whmcs-product-bandwidth
description: Configure product bandwidth limits in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, product, bandwidth, limit]
---

# WHMCS Product Bandwidth Configuration Workflow

## Purpose
Step-by-step guide for setting bandwidth limits on products.

## Prerequisites
- WHMCS admin access
- Server module configured
- Bandwidth monitoring enabled

## Step 1: Access Bandwidth Settings
- Navigate to Product > Module Settings
- Select server module
- Find bandwidth configuration

## Step 2: Configure Base Limits
- Set default bandwidth (GB)
- Configure overage pricing
- Set warning threshold
- Set hard limit behavior

## Step 3: Create Bandwidth Tiers
Create options in configurable options:
- Basic: 100GB/month
- Standard: 500GB/month
- Premium: 1TB/month
- Unlimited: No limit

## Step 4: Set Overages
- Define overage rate per GB
- Set overage notification
- Configure suspension on excess
- Set resume policy

## Step 5: Monitor Usage
- Enable bandwidth tracking
- Set monitoring interval
- Configure warning emails
- Set reporting

## Step 6: Bandwidth Addons
- Create bandwidth addons
- Set addon pricing
- Allow client upgrade
- Configure automatic upgrade

## Step 7: Client Communication
- Show bandwidth usage
- Display limits in client area
- Send usage warnings
- Notify of approaching limit

## Bandwidth Management
- Track usage per client
- Generate usage reports
- Bill overages
- Manage limits

## Related Workflows
- whmcs-product-config
- whmcs-product-upgrade
- whmcs-product-addon