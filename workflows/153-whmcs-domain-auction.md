---
name: whmcs-domain-auction
description: Participate in domain auction in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, auction, bid]
---

# WHMCS Domain Auction Workflow

## Purpose
Step-by-step guide for participating in domain auctions.

## Prerequisites
- WHMCS admin access
- Auction access
- Payment method configured

## Step 1: Find Auction Domains
- Access auction listings
- Browse available domains
- Filter by:
  - Price range
  - TLD
  - Length
  - Category

## Step 2: Research Domain
- Check domain value
- Review history
- Check traffic potential
- Verify legal status

## Step 3: Place Bid
- Enter maximum bid
- Set bid increment
- Confirm bid amount
- Submit bid

## Step 4: Monitor Auction
- Track bid status
- Watch competing bids
- Monitor time remaining
- Set auto-bid if outbid

## Step 5: Auction End
If winning:
- Confirm payment
- Process payment
- Domain transferred
- Client notified

If outbid:
- Decide on counter-bid
- Set new maximum
- Or let auction end

## Step 6: Payment
- Process auction payment
- Add to client account
- Generate invoice
- Update domain status

## Step 7: Post-Auction
- Register domain
- Configure DNS
- Set up management
- Update client records

## Auction Types
- Dutch auction
- English auction
- Buy it now
- Sealed bid

## Related Workflows
- whmcs-domain-backorder
- whmcs-domain-register
- whmcs-domain-auction-flow