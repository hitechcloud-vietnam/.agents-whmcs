---
name: whmcs-domain-backorder
description: Backorder domain in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, backorder, auction]
---

# WHMCS Domain Backorder Workflow

## Purpose
Step-by-step guide for setting up domain backorders.

## Prerequisites
- WHMCS admin access
- Backorder service configured
- Payment method set

## Step 1: Check Domain Availability
- Search for expiring domain
- Check backorder availability
- Verify registrar supports backorder

## Step 2: Place Backorder
- Navigate to Domain Search
- Enter desired domain
- Click "Backorder" or "Notify"
- Confirm backorder request

## Step 3: Configure Backorder
- Set maximum bid (optional)
- Add payment method
- Confirm terms of service
- Accept backorder agreement

## Step 4: Payment Setup
- Add credit/payment method
- Set auto-fund for winning
- Configure notification preferences
- Set alert options

## Step 5: Monitor Backorder
- Track domain expiration
- Monitor auction status
- Check if domain becomes available
- Receive notifications

## Step 6: Auction Participation
When domain becomes available:
- Automatic bidding if configured
- Manual bid if threshold exceeded
- Monitor auction progress
- Set maximum bid

## Step 7: Win/ Lose Domain
If won:
- Payment processed
- Domain registered
- Client notified
- DNS configured

If lost:
- Notification sent
- Partial refund if applicable
- Remove from tracking

## Backorder Services
- Drop catching
- Pre-release registration
- Expiring domain auction
- Deleted domain capture

## Related Workflows
- whmcs-domain-backorder-flow
- whmcs-domain-auction
- whmcs-domain-drop-catch