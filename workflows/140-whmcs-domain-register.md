---
name: whmcs-domain-register
description: Register domain in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, register, registrar]
---

# WHMCS Domain Registration Workflow

## Purpose
Step-by-step guide for registering domains in WHMCS.

## Prerequisites
- WHMCS admin access
- Domain registrar module configured
- Sufficient account balance
- Client account ready

## Step 1: Domain Search
- Navigate to WHMCS order page
- Enter desired domain name
- Check availability
- View available TLDs

## Step 2: Select Domain
- Choose available domain
- Select registration period (1-10 years)
- Add to cart
- Review cart

## Step 3: Configure Registration
- Select registrant:
  - New client
  - Existing client
  - Current user
- Enter registrant details:
  - Name
  - Address
  - Email
  - Phone

## Step 4: DNS Configuration
- Set nameservers:
  - Default nameservers
  - Custom nameservers
  - WHMCS default DNS
- Configure options:
  - DNS management
  - Email forwarding

## Step 5: Privacy Protection
- Enable WHOIS privacy
- Choose privacy level
- Configure contact visibility
- Set proxy information

## Step 6: Complete Order
- Review order details
- Apply promo code
- Select payment method
- Process payment

## Step 7: Registration Confirmation
- Order completed
- Domain registered with registrar
- Confirmation email sent
- Client account updated
- Domain shows in client area

## Step 8: Post-Registration
- Verify domain active
- Check DNS propagation
- Update domain management
- Set renewal reminders

## Domain Registration Options
- Register new domain
- Transfer domain
- Renew domain
- Pre-registration

## Registrar Configuration
Configure registrar module:
- Modules > Domain Registrars
- Install registrar module
- Enter credentials
- Configure settings

## Related Workflows
- whmcs-domain-transfer
- whmcs-domain-renew
- whmcs-domain-nameservers