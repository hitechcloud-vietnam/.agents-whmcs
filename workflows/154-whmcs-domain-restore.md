---
name: whmcs-domain-restore
description: Restore expired domain in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, restore, expired]
---

# WHMCS Domain Restore Workflow

## Purpose
Step-by-step guide for restoring expired domains.

## Prerequisites
- WHMCS admin access
- Domain in expired state
- Within restore period
- Sufficient balance

## Step 1: Identify Expired Domain
- Navigate to Domains
- Filter by expired status
- View expiration date
- Note restore deadline

## Step 2: Verify Restore Period
Check restoration eligibility:
- Grace period status (varies by TLD)
- Redemption period status
- Cost of restoration
- Success probability

## Step 3: Initiate Restore
- Open domain details
- Click "Restore" button
- Review restoration fee
- Confirm action

## Step 4: Process Payment
- Restoration fee applied
- Process payment
- System submits restore
- Registry processes request

## Step 5: Registry Processing
- Registry restores domain
- Restore period may be 30+ days
- Domain remains expired until complete
- Monitor restore status

## Step 6: Verify Restoration
- Domain status changes to Active
- Expiration date updated
- WHOIS restored
- DNS resumes

## Step 7: Post-Restoration
- Update expiration reminder
- Verify services work
- Check DNS resolution
- Set up auto-renewal

## Restore Timeframes
- Grace period: 0-30 days
- Redemption period: 30-75 days
- Pending delete: 75+ days
- After deletion: Not restorable

## Restore Costs
- Standard restoration fee
- Plus renewal fee
- May include backorder fee
- Registry-specific pricing

## Related Workflows
- whmcs-domain-renew
- whmcs-domain-delete
- whmcs-domain-sync