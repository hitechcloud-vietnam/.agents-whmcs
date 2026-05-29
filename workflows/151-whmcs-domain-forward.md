---
name: whmcs-domain-forward
description: Set domain forwarding in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, forwarding, redirect]
---

# WHMCS Domain Forwarding Setup Workflow

## Purpose
Step-by-step guide for setting up domain forwarding/redirects.

## Prerequisites
- WHMCS admin access
- Domain registered
- Forwarding service enabled

## Step 1: Access Forwarding Settings
- Navigate to Domains > Management
- Select domain
- Find "Forwarding" or "URL Redirect"
- Click "Configure Forwarding"

## Step 2: Configure Forwarding
Select forwarding type:
- 301 Permanent Redirect
- 302 Temporary Redirect
- Frame redirect (masking)

## Step 3: Set Target URL
- Enter destination URL
- Example: https://example.com
- Verify URL is valid
- Select protocol (http/https)

## Step 4: Configure Options
- Forward with path: Yes/No
- Forward with query string: Yes/No
- SEO-friendly options
- URL masking settings

## Step 5: Apply Forwarding
- Click "Enable Forwarding"
- System configures DNS
- Forwarding service activated
- Domain redirects to target

## Step 6: Test Forwarding
- Open domain in browser
- Verify redirect works
- Check redirect type
- Confirm URL masking if set

## Step 7: Monitor Forwarding
- Check redirect logs
- Monitor traffic
- Verify SEO impact
- Adjust settings if needed

## Forwarding Types
1. 301 Permanent: SEO best for moved content
2. 302 Temporary: When content returns
3. Meta Refresh: Browser-based redirect
4. Frame: Shows target in frame

## Use Cases
- Parked domains
- Brand protection
- Marketing campaigns
- Legacy URLs

## Related Workflows
- whmcs-domain-nameservers
- whmcs-domain-forwarding-setup
- whmcs-subdomain-automation