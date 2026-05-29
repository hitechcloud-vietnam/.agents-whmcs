---
name: whmcs-domain-unlock
description: Unlock domain in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, unlock, transfer]
---

# WHMCS Domain Unlock Workflow

## Purpose
Step-by-step guide for unlocking domains for transfers.

## Prerequisites
- WHMCS admin access
- Domain locked
- Transfer authorization
- Valid reason for unlock

## Step 1: Verify Unlock Authorization
- Confirm transfer request
- Get client approval
- Verify recipient registrar
- Document reason for unlock

## Step 2: Access Domain Management
- Navigate to Domains > Management
- Select locked domain
- View lock status

## Step 3: Disable Domain Lock
- Click "Unlock Domain"
- Confirm action
- System sends unlock request
- Registrar removes lock

## Step 4: Verification
- Check domain status
- WHOIS shows unlocked
- Confirm transfers enabled
- Document unlock

## Step 5: Initiate Transfer
After unlock:
- Obtain EPP code
- Begin transfer process
- Monitor transfer status
- Complete transfer

## Unlock Safety
- Verify recipient
- Set short unlock window
- Monitor for transfers
- Lock after transfer if needed

## Unlock Considerations
- Only unlock when transferring
- Unlock only as long as needed
- Re-lock after transfer
- Document all unlocks

## Related Workflows
- whmcs-domain-lock
- whmcs-domain-transfer
- whmcs-domain-epp