---
name: whmcs-domain-lock
description: Lock domain in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, lock, security]
---

# WHMCS Domain Lock Workflow

## Purpose
Step-by-step guide for locking domains to prevent unauthorized transfers.

## Prerequisites
- WHMCS admin access
- Domain registered in WHMCS
- Registrar supports locking

## Step 1: Verify Lock Support
Check if registrar supports:
- Registrar lock
- Transfer lock
- Status lock
- Registry lock (premium)

## Step 2: Access Domain Management
- Navigate to Domains > Management
- Select domain
- View current lock status

## Step 3: Enable Domain Lock
- Click "Lock Domain"
- Confirm action
- System sends lock request
- Registrar applies lock

## Step 4: Verify Lock Applied
- Check domain status
- WHOIS shows lock
- Test transfer blocked
- Confirm lock active

## Step 5: Update Security
- Document lock status
- Notify client
- Set lock as permanent
- Add to security checklist

## Domain Lock Benefits
- Prevents unauthorized transfer
- Stops accidental transfers
- Protects against social engineering
- Adds security layer

## Lock Status Types
- ClientTransferProhibited
- TransferLock
- RegistrarLock
- ClientUpdateProhibited

## When to Lock
- High-value domains
- Primary business domains
- Brand domains
- Premium domains

## Related Workflows
- whmcs-domain-unlock
- whmcs-domain-transfer
- whmcs-domain-epp