---
name: whmcs-domain-contact
description: Update domain contact information in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, contact, registrant]
---

# WHMCS Domain Contact Update Workflow

## Purpose
Step-by-step guide for updating domain contact information.

## Prerequisites
- WHMCS admin access
- Domain in WHMCS
- Updated contact details
- Proper authorization

## Step 1: Access Domain Contacts
- Navigate to Domains > Management
- Select domain
- Click "Contact Information"

## Step 2: Review Current Contacts
View current information:
- Registrant contact
- Admin contact
- Technical contact
- Billing contact

## Step 3: Update Registrant
- Modify registrant details:
  - First/Last name
  - Organization
  - Email address
  - Phone number
  - Address information
- Changes may require verification

## Step 4: Update Other Contacts
Update contacts:
- Admin contact
- Technical contact
- Billing contact
- Use same or different contacts

## Step 5: GDPR Considerations
- Privacy protection
- Data minimization
- Consent requirements
- Local requirements

## Step 6: Save Changes
- Click "Save Changes"
- System submits to registry
- Verification email sent
- WHOIS updated

## Step 7: Verification Process
- Email sent to domain email
- Owner must verify
- Complete within time limit
- Contact updated after verification

## Step 8: Post-Update
- Verify WHOIS updated
- Confirm all contacts set
- Update client records
- Document change

## Contact Change Restrictions
Some TLDs:
- Require transfer for registrant change
- Have grace period
- May trigger renewal
- Require documentation

## Related Workflows
- whmcs-domain-sync
- whmcs-domain-register
- whmcs-registrant-verification