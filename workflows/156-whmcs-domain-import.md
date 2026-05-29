---
name: whmcs-domain-import
description: Bulk import domains into WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, import, bulk]
---

# WHMCS Domain Bulk Import Workflow

## Purpose
Step-by-step guide for importing domains in bulk.

## Prerequisites
- WHMCS admin access
- Import file (CSV/Excel)
- Domain registrar configured
- Template mapping

## Step 1: Prepare Import File
CSV format with columns:
- Domain name
- Registration date
- Expiration date
- Registrant info
- Nameservers
- Status

## Step 2: Access Import Function
- Navigate to Domains > Management
- Click "Import Domains"
- Or use Tools > Import

## Step 3: Upload File
- Select import file
- Upload to WHMCS
- System validates format
- Preview data

## Step 4: Map Fields
Map columns to WHMCS:
- Domain name
- Client ID
- Registrant
- Nameservers
- Registration date
- Expiration date

## Step 5: Configure Import
- Select default client (if no client match)
- Set default registrar
- Configure default settings
- Handle duplicates

## Step 6: Run Import
- Click "Import Domains"
- Monitor progress
- View results:
  - Imported count
  - Failed count
  - Warnings

## Step 7: Review and Fix
- Review failed imports
- Fix data errors
- Re-import failed
- Verify imported domains

## Step 8: Post-Import
- Verify domains in WHMCS
- Set up renewal reminders
- Configure sync
- Update reports

## Import Best Practices
- Backup before import
- Test with small sample
- Validate file format
- Check for duplicates

## Related Workflows
- whmcs-domain-export
- whmcs-domain-pricing
- whmcs-data-export-import