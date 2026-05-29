---
name: whmcs-domain-epp
description: Get EPP code for domain transfer in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, epp, transfer]
---

# WHMCS Domain EPP Code Request Workflow

## Purpose
Step-by-step guide for obtaining EPP auth codes for domain transfers.

## Prerequisites
- WHMCS admin access
- Domain registered in WHMCS
- Transfer request or verification need

## Step 1: Verify Transfer Readiness
- Confirm domain is eligible
- Check domain unlocked
- Verify ownership confirmed
- Note transfer deadline

## Step 2: Access EPP Request
- Navigate to Domains > Management
- Select domain
- Find "Release Domain" or "Get Auth Code"

## Step 3: Request EPP Code
- Click "Get Auth Code" or "Request EPP"
- System retrieves from registrar
- Code displayed in WHMCS
- Code also sent to registrant email

## Step 4: Receive Code
- EPP code displayed on screen
- Code sent to domain email
- Note code securely
- Provide to recipient registrar

## Step 5: Provide to Recipient
- Give EPP code to receiving registrar
- Initiate transfer at new registrar
- Confirm transfer request
- Approve transfer notification

## EPP Code Security
- Send via secure channel
- Don't share publicly
- Document code provision
- Monitor for transfer

## EPP Code Format
- Alphanumeric string
- 6-32 characters
- Case-sensitive
- Format varies by registry

## Common EPP Issues
- Email not received
- Code expired
- Invalid code format
- Registrar doesn't support

## Related Workflows
- whmcs-domain-transfer
- whmcs-domain-unlock
- whmcs-domain-lock