---
name: whmcs-domain-nameservers
description: Update domain nameservers in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, nameservers, dns]
---

# WHMCS Domain Nameserver Update Workflow

## Purpose
Step-by-step guide for updating domain nameservers in WHMCS.

## Prerequisites
- WHMCS admin access
- Domain registered in WHMCS
- New nameserver addresses

## Step 1: Access Domain Management
- Navigate to WHMCS Admin > Domains
- Find target domain
- Click domain name to open

## Step 2: Locate Nameserver Section
- In domain details, find "Nameservers" section
- Current nameservers displayed
- Click "Change Nameservers"

## Step 3: Enter New Nameservers
Enter nameserver addresses:
- Primary: ns1.example.com
- Secondary: ns2.example.com
- Additional: ns3.example.com (optional)
- IPv6 if required

## Step 4: Validate Nameservers
- System checks format
- Verify NS records exist
- Confirm nameservers are valid
- Check propagation status

## Step 5: Apply Changes
- Click "Update Nameservers"
- System sends update to registry
- Registry confirms change
- Nameservers updated

## Step 6: Propagation
DNS propagation:
- TTL: 0-48 hours
- Global propagation: 24-72 hours
- Monitor propagation status
- Verify changes reflect globally

## Step 7: Verify Update
- Use DNS lookup tools
- Check WHOIS nameservers
- Test domain resolution
- Verify web/email services work

## Common Nameserver Changes
- Point to hosting provider
- Point to DNS provider
- Use default registrar DNS
- Point to cloud provider

## Preserving DNS Records
When changing NS:
- Backup existing DNS records
- Migrate records to new provider
- Test before final change
- Update after propagation

## Related Workflows
- whmcs-domain-sync
- whmcs-dns-configuration
- whmcs-domain-contact