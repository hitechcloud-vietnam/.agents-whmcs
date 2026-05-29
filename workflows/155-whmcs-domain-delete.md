---
name: whmcs-domain-delete
description: Delete domain from WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, delete, remove]
---

# WHMCS Domain Delete Workflow

## Purpose
Step-by-step guide for deleting/removing domains from WHMCS.

## Prerequisites
- WHMCS admin access
- Domain in WHMCS
- No active services
- Authorization to delete

## Step 1: Verify Delete Eligibility
- Check domain status
- Verify no pending transfers
- Confirm no active services
- Check no pending payments

## Step 2: Cancel Related Services
- Terminate hosting accounts
- Cancel SSL certificates
- Stop email services
- Remove DNS records

## Step 3: Check Domain Status
- Domain must be:
  - Expired
  - Or in client's possession
- Cannot delete active domain
- Domain remains with registrar

## Step 4: Remove from WHMCS
- Navigate to Domains > Management
- Select domain
- Click "Delete Domain"
- Confirm deletion

## Step 5: Handle Data
- Client data preserved
- Historical records kept
- Invoices remain
- Transaction history maintained

## Step 6: Notify Client
- Inform domain removed
- Confirm domain ownership transfer
- Provide domain details
- Document in notes

## Step 7: Post-Delete
- Remove from lists
- Update reports
- Archive for records
- Document in audit log

## Delete vs Cancel
| Aspect | Delete | Cancel |
|--------|--------|--------|
| Scope | WHMCS record | Service |
| Data | Historical preserved | Active removed |
| Domain | May remain registered | Depends |

## Related Workflows
- whmcs-domain-restore
- whmcs-domain-transfer
- whmcs-service-cancellation