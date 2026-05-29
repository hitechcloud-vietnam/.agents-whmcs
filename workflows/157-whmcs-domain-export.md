---
name: whmcs-domain-export
description: Export domains from WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, domain, export, report]
---

# WHMCS Domain Export Workflow

## Purpose
Step-by-step guide for exporting domain data.

## Prerequisites
- WHMCS admin access
- Export permissions
- Output format selected

## Step 1: Access Export Function
- Navigate to Domains > Management
- Click "Export"
- Or Reports > Domain Reports

## Step 2: Configure Export Filters
Date Range:
- Custom dates
- Registration date
- Expiration date
- All time

Status Filter:
- All domains
- Active
- Expired
- Pending
- Transferred

Other Filters:
- Registrar
- Client
- TLD
- Registration period

## Step 3: Select Export Format
- CSV
- Excel (.xlsx)
- PDF

## Step 4: Choose Export Fields
Select columns:
- Domain name
- Client name
- Registration date
- Expiration date
- Registrar
- Nameservers
- Status
- Auto-renew
- Privacy status
- Cost

## Step 5: Generate Export
- Click "Generate Export"
- System processes data
- Download file
- Save to location

## Step 6: Verify Data
- Open export file
- Check record count
- Verify data accuracy
- Validate formatting

## Export Use Cases
- Domain portfolio management
- Expiration tracking
- Registrar comparison
- Renewal planning
- Compliance audit

## Scheduled Exports
- Weekly expiration report
- Monthly domain summary
- Quarterly audit
- Annual renewal planning

## Related Workflows
- whmcs-domain-import
- whmcs-domain-pricing
- whmcs-report-generation