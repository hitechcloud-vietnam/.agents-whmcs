---
name: whmcs-domain-sync
description: Sync domain with registrar in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, sync, registrar]
---

# WHMCS Domain Sync Workflow

## Purpose
Step-by-step guide for synchronizing domain data with registrar.

## Prerequisites
- WHMCS admin access
- Registrar module configured
- Active registrar account

## Step 1: Configure Sync Settings
- Navigate to Setup > Domains
- Configure registrar sync:
  - Sync frequency
  - Auto-sync enabled
  - Sync details to include

## Step 2: Manual Sync
To sync specific domain:
- Open Domain Management
- Select domain
- Click "Sync with Registrar"
- System retrieves current data

## Step 3: Bulk Sync
To sync multiple domains:
- Select domains via checkboxes
- Click "Bulk Actions"
- Select "Sync with Registrar"
- System syncs all selected

## Step 4: Verify Sync Data
Check synced information:
- Expiration date
- Status (active, expired, pending)
- Nameservers
- Contact information
- Registration period

## Step 5: Handle Discrepancies
If WHMCS differs from registrar:
- Review differences
- Update WHMCS to match registrar
- Or update registrar if needed
- Document changes

## Step 6: Schedule Automatic Sync
- Setup > Automation > Cron
- Configure domain sync cron
- Set frequency (daily recommended)
- Enable automatic sync

## Step 7: Post-Sync Actions
- Review sync log
- Check for errors
- Update domain status
- Process renewals if needed

## Sync Updates
Domain sync updates:
- Expiration dates
- Status changes
- Transfer status
- Contact updates
- DNS changes

## Common Sync Issues
- Registrar API timeout
- Authentication failure
- Invalid response
- Rate limiting

## Related Workflows
- whmcs-domain-renew
- whmcs-domain-sync-automation
- whmcs-domain-nameservers