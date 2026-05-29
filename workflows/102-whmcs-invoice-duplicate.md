---
name: whmcs-invoice-duplicate
description: Duplicate existing invoice in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, billing, duplicate]
---

# WHMCS Invoice Duplication Workflow

## Purpose
Step-by-step guide for duplicating invoices in WHMCS for recurring billing or similar clients.

## Prerequisites
- WHMCS admin access
- Existing invoice to duplicate
- Appropriate permissions

## Step 1: Source Invoice Selection
- Navigate to WHMCS Admin > Billing > Invoices
- Locate source invoice by number or client
- Open invoice details

## Step 2: Initiate Duplication
- Click "Duplicate Invoice" action
- System presents duplication options:
  - Include all line items
  - Include taxes
  - Reset dates to current
  - Apply to same or different client

## Step 3: Configure New Invoice
- Select target client
- Set new invoice date
- Choose payment terms
- Update description if needed

## Step 4: Modify Line Items
- Review pre-filled items from source
- Update product versions if applicable
- Adjust pricing for new period
- Add or remove items as needed

## Step 5: Finalize Duplicate
- Review all configurations
- Verify tax calculations
- Set appropriate status (Draft/Sent)
- Click "Create Invoice"

## Step 6: Post-Creation
- Assign new invoice number (auto-generated)
- Send to client if appropriate
- Link to original invoice in notes
- Schedule follow-up if needed

## Use Cases
- Monthly recurring invoices
- Similar services for different clients
- Annual renewal preparation

## Related Workflows
- whmcs-invoice-creation
- whmcs-invoice-batch
- whmcs-invoice-merge