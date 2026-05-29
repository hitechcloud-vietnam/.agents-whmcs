---
name: whmcs-domain-transfer
description: Transfer domain in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, transfer, registrar]
---

# WHMCS Domain Transfer Workflow

## Purpose
Step-by-step guide for transferring domains in WHMCS.

## Prerequisites
- WHMCS admin access
- Domain registrar configured
- EPP code (auth code) from current registrar
- Domain unlocked at current registrar
- Sufficient balance

## Step 1: Verify Transfer Eligibility
- Check domain is eligible:
  - 60+ days since registration
  - Not recently transferred
  - Not in legal dispute
  - Email verified
- Confirm domain is unlocked

## Step 2: Obtain EPP Code
- Contact current registrar
- Request auth code
- Verify domain email
- Note EPP code

## Step 3: Initiate Transfer in WHMCS
- Navigate to order page
- Enter domain to transfer
- Select transfer option
- Add to cart

## Step 4: Enter Transfer Details
- Enter domain name
- Enter EPP auth code
- Confirm registrant details
- Select registration period

## Step 5: Complete Order
- Review transfer cost
- Process payment
- Confirm transfer request
- System submits to registrar

## Step 6: Transfer Confirmation
- Domain owner receives email
- Owner must approve transfer
- Approve within specified time
- Confirmation sent to new registrar

## Step 7: Transfer Completion
- Registry processes transfer
- WHOIS updated
- Nameservers preserved
- Transfer complete
- Client notified

## Step 8: Post-Transfer
- Verify domain in WHMCS
- Update nameservers if needed
- Set renewal reminders
- Update DNS records

## Transfer Timeframes
- gTLD domains: 5-7 days
- ccTLD domains: Varies (1-14 days)
- Registry approval may be required

## Transfer Issues
- Incorrect EPP code
- Domain locked
- Pending transfer already
- Registry lock
- Legal transfer lock

## Related Workflows
- whmcs-domain-register
- whmcs-domain-renew
- whmcs-domain-epp