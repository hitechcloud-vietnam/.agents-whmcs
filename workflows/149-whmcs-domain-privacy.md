---
name: whmcs-domain-privacy
description: Toggle domain privacy in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, privacy, whois]
---

# WHMCS Domain Privacy Toggle Workflow

## Purpose
Step-by-step guide for enabling/disabling domain WHOIS privacy.

## Prerequisites
- WHMCS admin access
- Domain with privacy option
- Registrar supports privacy

## Step 1: Access Privacy Settings
- Navigate to Domains > Management
- Select domain
- Find "Privacy Protection" section
- View current status

## Step 2: Check Privacy Availability
- Verify TLD supports privacy
- Check registrar offers privacy
- Compare pricing options
- Review privacy features

## Step 3: Enable Privacy
To enable WHOIS privacy:
- Click "Enable Privacy"
- Review privacy cost
- Confirm purchase
- System enables privacy

## Step 4: Disable Privacy
To disable privacy:
- Click "Disable Privacy"
- Confirm action
- Review implications
- WHOIS shows actual data

## Step 5: Verify Privacy Status
- Check WHOIS lookup
- Verify proxy details
- Confirm email protection
- Test contact forwarding

## Step 6: Privacy Management
- Configure email forwarding
- Set privacy level
- Manage contact visibility
- Handle legal requests

## Privacy Features
- Hidden registrant details
- Proxy email address
- Contact forwarding service
- Mail forwarding
- Phone masking

## Privacy vs Public
| Aspect | Privacy | Public |
|--------|---------|--------|
| Contact Info | Hidden | Visible |
| Email | Proxy | Direct |
| Address | Proxy | Direct |
| Cost | Extra fee | Included |

## Related Workflows
- whmcs-domain-contact
- whmcs-domain-sync
- whmcs-whois-privacy-setup