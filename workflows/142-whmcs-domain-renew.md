---
name: whmcs-domain-renew
description: Renew domain in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, renew, registrar]
---

# WHMCS Domain Renewal Workflow

## Purpose
Step-by-step guide for renewing domains in WHMCS.

## Prerequisites
- WHMCS admin access
- Domain registrar module
- Domain in WHMCS
- Sufficient balance

## Step 1: Identify Renewal Needed
- Navigate to Domains > Management
- View domain list
- Identify expiring domains
- Check renewal dates

## Step 2: Configure Auto-Renewal
- Setup > Domains > Domain Pricing
- Enable auto-renewal
- Set renewal threshold (e.g., 14 days)
- Configure billing settings

## Step 3: Manual Renewal
For specific domain:
- Open domain management
- Click "Renew" button
- Select renewal period:
  - 1 year
  - 2 years
  - 5 years
  - 10 years
- Review cost

## Step 4: Process Renewal
- Confirm renewal period
- Select payment method
- Process payment
- System sends renewal request

## Step 5: Renewal Confirmation
- Registry confirms renewal
- Expiration date extended
- Client notified
- Invoice generated

## Step 6: Verify Renewal
- Check new expiration date
- Confirm WHOIS updated
- Verify domain status
- Update renewal reminder

## Step 7: Post-Renewal
- Set next reminder
- Update documentation
- Verify DNS still working
- Check services attached

## Renewal Notification
Configure reminders:
- 90 days before
- 60 days before
- 30 days before
- 14 days before
- Expiration day

## Renewal Failure
If renewal fails:
- Check account balance
- Verify registrar connection
- Check domain status
- Retry manually
- Contact registrar support

## Related Workflows
- whmcs-domain-register
- whmcs-domain-sync
- whmcs-domain-pricing