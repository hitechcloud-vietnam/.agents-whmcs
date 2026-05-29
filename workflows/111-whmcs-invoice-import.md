---
name: whmcs-invoice-import
description: Import invoices into WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, import, billing]
---

# WHMCS Invoice Import Workflow

## Purpose
Step-by-step guide for importing invoices into WHMCS from external sources.

## Prerequisites
- WHMCS admin access
- Import file (CSV/Excel)
- Valid client mappings
- Field mapping configuration

## Step 1: Prepare Import File
- Format: CSV or Excel (.xlsx)
- Required fields:
  - Client identifier (email or client ID)
  - Invoice date
  - Line items (description, amount)
  - Due date
- Optional fields:
  - Invoice number (auto-generate if missing)
  - Tax rate
  - Payment terms

## Step 2: Access Import Function
- Navigate to WHMCS Admin > Billing > Invoices
- Click "Import Invoices"
- Or access via Tools > Import/Export

## Step 3: Upload Import File
- Select file from local system
- Upload to WHMCS
- System validates file format
- Preview first few records

## Step 4: Configure Field Mapping
Map CSV columns to WHMCS fields:
- Client Email → Client Lookup
- Invoice Date → Invoice Date
- Amount → Line Item Amount
- Description → Line Item Description
- Due Date → Due Date

## Step 5: Handle Data Mapping
- Client resolution: Match existing or create new
- Product resolution: Map to existing products
- Tax rules: Apply default or specific
- Currency conversion if needed

## Step 6: Preview and Validate
- Review mapped data
- Check for errors/warnings
- Verify record count
- Identify failed mappings
- Adjust mappings as needed

## Step 7: Execute Import
- Click "Import Invoices"
- Monitor progress
- View import summary:
  - Success count
  - Failed count
  - Errors list
- Resolve any failures

## Step 8: Post-Import Actions
- Verify imported invoices in WHMCS
- Review invoice details
- Check client records updated
- Process any follow-up needed

## Import Best Practices
- Backup before bulk imports
- Test with small sample first
- Validate file format
- Check for duplicates
- Document mapping configuration

## Common Import Errors
- Invalid client email
- Missing required fields
- Duplicate invoice numbers
- Invalid date format
- Amount format errors

## Related Workflows
- whmcs-invoice-export
- whmcs-data-export-import
- whmcs-client-migration