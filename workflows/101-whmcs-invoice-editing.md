---
name: whmcs-invoice-editing
description: Edit existing invoice in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, billing, edit]
---

# WHMCS Invoice Editing Workflow

## Purpose
Step-by-step guide for editing existing invoices in WHMCS.

## Prerequisites
- WHMCS admin access
- Existing invoice ID
- Invoice in editable status (Draft, Unpaid)

## Step 1: Locate Invoice
- Navigate to WHMCS Admin > Billing > Invoices
- Search by invoice number or client name
- Click on invoice to open details

## Step 2: Verify Edit Permissions
- Check invoice status allows editing
- Note: Paid/Void/Cancelled invoices are locked
- Create credit note if invoice is paid

## Step 3: Edit Invoice Header
- Modify invoice date if needed
- Update payment terms
- Change due date if applicable
- Update billing address

## Step 4: Modify Line Items
- Add new items via "Add Item" button
- Edit existing item description/price
- Remove items by clearing line
- Update quantities as needed

## Step 5: Adjust Taxes
- Recalculate tax on modified items
- Apply tax exemption if applicable
- Verify tax jurisdiction is correct

## Step 6: Save Changes
- Review all modifications
- Click "Save Changes"
- System logs edit in audit trail

## Step 7: Notify Client (if sent)
- Resend updated invoice
- Include change summary in email
- Document communication in client log

## Important Notes
- Original invoice number preserved
- Edit timestamp recorded
- Payment history preserved
- Tax recalculations processed

## Related Workflows
- whmcs-invoice-creation
- whmcs-invoice-credit-note
- whmcs-invoice-void